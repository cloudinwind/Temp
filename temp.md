StreamAlsa.cpp 内容如下：
```bash
alsa git:(1a56e38edc) ✗ cat StreamAlsa.cpp  
/*
 * Copyright (C) 2023 The Android Open Source Project
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

#include <cmath>
#include <limits>

#define LOG_TAG "AHAL_StreamAlsa"
#include <android-base/logging.h>

#include <Utils.h>
#include <audio_utils/clock.h>
#include <error/expected_utils.h>
#include <media/AidlConversionCppNdk.h>

#include "core-impl/StreamAlsa.h"

using aidl::android::hardware::audio::common::getChannelCount;

namespace aidl::android::hardware::audio::core {

StreamAlsa::StreamAlsa(StreamContext* context, const Metadata& metadata, int readWriteRetries)
    : StreamCommonImpl(context, metadata),
      mBufferSizeFrames(getContext().getBufferSizeInFrames()),
      mFrameSizeBytes(getContext().getFrameSize()),
      mSampleRate(getContext().getSampleRate()),
      mIsInput(isInput(metadata)),
      mConfig(alsa::getPcmConfig(getContext(), mIsInput)),
      mReadWriteRetries(readWriteRetries) {}

StreamAlsa::~StreamAlsa() {
    cleanupWorker();
}

::android::NBAIO_Format StreamAlsa::getPipeFormat() const {
    const audio_format_t audioFormat = VALUE_OR_FATAL(
            aidl2legacy_AudioFormatDescription_audio_format_t(getContext().getFormat()));
    const int channelCount = getChannelCount(getContext().getChannelLayout());
    return ::android::Format_from_SR_C(getContext().getSampleRate(), channelCount, audioFormat);
}

::android::sp<::android::MonoPipe> StreamAlsa::makeSink(bool writeCanBlock) {
    const ::android::NBAIO_Format format = getPipeFormat();
    auto sink = ::android::sp<::android::MonoPipe>::make(mBufferSizeFrames, format, writeCanBlock);
    const ::android::NBAIO_Format offers[1] = {format};
    size_t numCounterOffers = 0;
    ssize_t index = sink->negotiate(offers, 1, nullptr, numCounterOffers);
    LOG_IF(FATAL, index != 0) << __func__ << ": Negotiation for the sink failed, index = " << index;
    return sink;
}

::android::sp<::android::MonoPipeReader> StreamAlsa::makeSource(::android::MonoPipe* pipe) {
    const ::android::NBAIO_Format format = getPipeFormat();
    const ::android::NBAIO_Format offers[1] = {format};
    auto source = ::android::sp<::android::MonoPipeReader>::make(pipe);
    size_t numCounterOffers = 0;
    ssize_t index = source->negotiate(offers, 1, nullptr, numCounterOffers);
    LOG_IF(FATAL, index != 0) << __func__
                              << ": Negotiation for the source failed, index = " << index;
    return source;
}

::android::status_t StreamAlsa::init(DriverCallbackInterface* /*callback*/) {
    return mConfig.has_value() ? ::android::OK : ::android::NO_INIT;
}

::android::status_t StreamAlsa::drain(StreamDescriptor::DrainMode) {
    if (!mIsInput) {
        static constexpr float kMicrosPerSecond = MICROS_PER_SECOND;
        const size_t delayUs = static_cast<size_t>(
                std::roundf(mBufferSizeFrames * kMicrosPerSecond / mSampleRate));
        usleep(delayUs);
    }
    return ::android::OK;
}

::android::status_t StreamAlsa::flush() {
    return ::android::OK;
}

::android::status_t StreamAlsa::pause() {
    return ::android::OK;
}

::android::status_t StreamAlsa::standby() {
    teardownIo();
    return ::android::OK;
}

::android::status_t StreamAlsa::start() {
    if (!mAlsaDeviceProxies.empty()) {
        // This is a resume after a pause.
        return ::android::OK;
    }
    decltype(mAlsaDeviceProxies) alsaDeviceProxies;
    decltype(mSources) sources;
    decltype(mSinks) sinks;
    for (const auto& device : getDeviceProfiles()) {
        if ((device.direction == PCM_OUT && mIsInput) ||
            (device.direction == PCM_IN && !mIsInput)) {
            continue;
        }
        alsa::DeviceProxy proxy;
        if (device.isExternal) {
            // Always ask alsa configure as required since the configuration should be supported
            // by the connected device. That is guaranteed by `setAudioPortConfig` and
            // `setAudioPatch`.
            proxy = alsa::openProxyForExternalDevice(
                    device, const_cast<struct pcm_config*>(&mConfig.value()),
                    true /*require_exact_match*/);
        } else {
            proxy = alsa::openProxyForAttachedDevice(
                    device, const_cast<struct pcm_config*>(&mConfig.value()), mBufferSizeFrames);
        }
        if (proxy.get() == nullptr) {
            return ::android::NO_INIT;
        }
        alsaDeviceProxies.push_back(std::move(proxy));
        auto sink = makeSink(mIsInput);  // Do not block the writer when it is on our thread.
        if (sink != nullptr) {
            sinks.push_back(sink);
        } else {
            return ::android::NO_INIT;
        }
        if (auto source = makeSource(sink.get()); source != nullptr) {
            sources.push_back(source);
        } else {
            return ::android::NO_INIT;
        }
    }
    if (alsaDeviceProxies.empty()) {
        return ::android::NO_INIT;
    }
    mAlsaDeviceProxies = std::move(alsaDeviceProxies);
    mSources = std::move(sources);
    mSinks = std::move(sinks);
    mIoThreadIsRunning = true;
    for (size_t i = 0; i < mAlsaDeviceProxies.size(); ++i) {
        mIoThreads.emplace_back(mIsInput ? &StreamAlsa::inputIoThread : &StreamAlsa::outputIoThread,
                                this, i);
    }
    return ::android::OK;
}

::android::status_t StreamAlsa::transfer(void* buffer, size_t frameCount, size_t* actualFrameCount,
                                         int32_t* latencyMs) {
    if (mAlsaDeviceProxies.empty()) {
        LOG(FATAL) << __func__ << ": no opened devices";
        return ::android::NO_INIT;
    }
    const size_t bytesToTransfer = frameCount * mFrameSizeBytes;
    unsigned maxLatency = 0;
    if (mIsInput) {
        const size_t i = 0;  // For the input case, only support a single device.
        LOG(VERBOSE) << __func__ << ": reading from sink " << i;
        ssize_t framesRead = mSources[i]->read(buffer, frameCount);
        LOG_IF(FATAL, framesRead < 0) << "Error reading from the pipe: " << framesRead;
        if (ssize_t framesMissing = static_cast<ssize_t>(frameCount) - framesRead;
            framesMissing > 0) {
            LOG(WARNING) << __func__ << ": incomplete data received, inserting " << framesMissing
                         << " frames of silence";
            memset(static_cast<char*>(buffer) + framesRead * mFrameSizeBytes, 0,
                   framesMissing * mFrameSizeBytes);
        }
        maxLatency = proxy_get_latency(mAlsaDeviceProxies[i].get());
    } else {
        alsa::applyGain(buffer, mGain, bytesToTransfer, mConfig.value().format, mConfig->channels);
        for (size_t i = 0; i < mAlsaDeviceProxies.size(); ++i) {
            LOG(VERBOSE) << __func__ << ": writing into sink " << i;
            ssize_t framesWritten = mSinks[i]->write(buffer, frameCount);
            LOG_IF(FATAL, framesWritten < 0) << "Error writing into the pipe: " << framesWritten;
            if (ssize_t framesLost = static_cast<ssize_t>(frameCount) - framesWritten;
                framesLost > 0) {
                LOG(WARNING) << __func__ << ": sink " << i << " incomplete data sent, dropping "
                             << framesLost << " frames";
            }
            maxLatency = std::max(maxLatency, proxy_get_latency(mAlsaDeviceProxies[i].get()));
        }
    }
    *actualFrameCount = frameCount;
    maxLatency = std::min(maxLatency, static_cast<unsigned>(std::numeric_limits<int32_t>::max()));
    *latencyMs = maxLatency;
    return ::android::OK;
}

::android::status_t StreamAlsa::refinePosition(StreamDescriptor::Position* position) {
    if (mAlsaDeviceProxies.empty()) {
        LOG(WARNING) << __func__ << ": no opened devices";
        return ::android::NO_INIT;
    }
    // Since the proxy can only count transferred frames since its creation,
    // we override its counter value with ours and let it to correct for buffered frames.
    alsa::resetTransferredFrames(mAlsaDeviceProxies[0], position->frames);
    if (mIsInput) {
        if (int ret = proxy_get_capture_position(mAlsaDeviceProxies[0].get(), &position->frames,
                                                 &position->timeNs);
            ret != 0) {
            LOG(WARNING) << __func__ << ": failed to retrieve capture position: " << ret;
            return ::android::INVALID_OPERATION;
        }
    } else {
        uint64_t hwFrames;
        struct timespec timestamp;
        if (int ret = proxy_get_presentation_position(mAlsaDeviceProxies[0].get(), &hwFrames,
                                                      &timestamp);
            ret == 0) {
            if (hwFrames > std::numeric_limits<int64_t>::max()) {
                hwFrames -= std::numeric_limits<int64_t>::max();
            }
            position->frames = static_cast<int64_t>(hwFrames);
            position->timeNs = audio_utils_ns_from_timespec(&timestamp);
        } else {
            LOG(WARNING) << __func__ << ": failed to retrieve presentation position: " << ret;
            return ::android::INVALID_OPERATION;
        }
    }
    return ::android::OK;
}

void StreamAlsa::shutdown() {
    teardownIo();
}

ndk::ScopedAStatus StreamAlsa::setGain(float gain) {
    mGain = gain;
    return ndk::ScopedAStatus::ok();
}

void StreamAlsa::inputIoThread(size_t idx) {
#if defined(__ANDROID__)
    setWorkerThreadPriority(pthread_gettid_np(pthread_self()));
    const std::string threadName = (std::string("in_") + std::to_string(idx)).substr(0, 15);
    pthread_setname_np(pthread_self(), threadName.c_str());
#endif
    const size_t bufferSize = mBufferSizeFrames * mFrameSizeBytes;
    std::vector<char> buffer(bufferSize);
    while (mIoThreadIsRunning) {
        if (int ret = proxy_read_with_retries(mAlsaDeviceProxies[idx].get(), &buffer[0], bufferSize,
                                              mReadWriteRetries);
            ret == 0) {
            size_t bufferFramesWritten = 0;
            while (bufferFramesWritten < mBufferSizeFrames) {
                if (!mIoThreadIsRunning) return;
                ssize_t framesWrittenOrError =
                        mSinks[idx]->write(&buffer[0], mBufferSizeFrames - bufferFramesWritten);
                if (framesWrittenOrError >= 0) {
                    bufferFramesWritten += framesWrittenOrError;
                } else {
                    LOG(WARNING) << __func__ << "[" << idx
                                 << "]: Error while writing into the pipe: "
                                 << framesWrittenOrError;
                }
            }
        } else {
            // Errors when the stream is being stopped are expected.
            LOG_IF(WARNING, mIoThreadIsRunning)
                    << __func__ << "[" << idx << "]: Error reading from ALSA: " << ret;
        }
    }
}

void StreamAlsa::outputIoThread(size_t idx) {
#if defined(__ANDROID__)
    setWorkerThreadPriority(pthread_gettid_np(pthread_self()));
    const std::string threadName = (std::string("out_") + std::to_string(idx)).substr(0, 15);
    pthread_setname_np(pthread_self(), threadName.c_str());
#endif
    const size_t bufferSize = mBufferSizeFrames * mFrameSizeBytes;
    std::vector<char> buffer(bufferSize);
    while (mIoThreadIsRunning) {
        ssize_t framesReadOrError = mSources[idx]->read(&buffer[0], mBufferSizeFrames);
        if (framesReadOrError > 0) {
            int ret = proxy_write_with_retries(mAlsaDeviceProxies[idx].get(), &buffer[0],
                                               framesReadOrError * mFrameSizeBytes,
                                               mReadWriteRetries);
            // Errors when the stream is being stopped are expected.
            LOG_IF(WARNING, ret != 0 && mIoThreadIsRunning)
                    << __func__ << "[" << idx << "]: Error writing into ALSA: " << ret;
        } else if (framesReadOrError == 0) {
            // MonoPipeReader does not have a blocking read, while use of std::condition_variable
            // requires use of a mutex. For now, just do a 1ms sleep. Consider using a different
            // pipe / ring buffer mechanism.
            if (mIoThreadIsRunning) usleep(1000);
        } else {
            LOG(WARNING) << __func__ << "[" << idx
                         << "]: Error while reading from the pipe: " << framesReadOrError;
        }
    }
}

void StreamAlsa::teardownIo() {
    mIoThreadIsRunning = false;
    if (mIsInput) {
        LOG(DEBUG) << __func__ << ": shutting down pipes";
        for (auto& sink : mSinks) {
            sink->shutdown(true);
        }
    }
    LOG(DEBUG) << __func__ << ": stopping PCM streams";
    for (const auto& proxy : mAlsaDeviceProxies) {
        proxy_stop(proxy.get());
    }
    LOG(DEBUG) << __func__ << ": joining threads";
    for (auto& thread : mIoThreads) {
        if (thread.joinable()) thread.join();
    }
    mIoThreads.clear();
    LOG(DEBUG) << __func__ << ": closing PCM devices";
    mAlsaDeviceProxies.clear();
    mSources.clear();
    mSinks.clear();
}

}  // namespace aidl::android::hardware::audio::core
➜  alsa git:(1a56e38edc) ✗
```

Mixer.cpp 内容如下：

```bash
➜  alsa git:(1a56e38edc) ✗ cat Mixer.cpp 
/*
 * Copyright (C) 2023 The Android Open Source Project
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

#include <algorithm>
#include <cmath>

#define LOG_TAG "AHAL_AlsaMixer"
#include <android-base/logging.h>
#include <android/binder_status.h>
#include <error/expected_utils.h>

#include "Mixer.h"

namespace ndk {

// This enables use of 'error/expected_utils' for ScopedAStatus.

inline bool errorIsOk(const ScopedAStatus& s) {
    return s.isOk();
}

inline std::string errorToString(const ScopedAStatus& s) {
    return s.getDescription();
}

}  // namespace ndk

namespace aidl::android::hardware::audio::core::alsa {

// static
const std::map<Mixer::Control, std::vector<Mixer::ControlNamesAndExpectedCtlType>>
        Mixer::kPossibleControls = {
                {Mixer::MASTER_SWITCH, {{"Master Playback Switch", MIXER_CTL_TYPE_BOOL}}},
                {Mixer::MASTER_VOLUME, {{"Master Playback Volume", MIXER_CTL_TYPE_INT}}},
                {Mixer::HW_VOLUME,
                 {{"Headphone Playback Volume", MIXER_CTL_TYPE_INT},
                  {"Headset Playback Volume", MIXER_CTL_TYPE_INT},
                  {"PCM Playback Volume", MIXER_CTL_TYPE_INT}}},
                {Mixer::MIC_SWITCH, {{"Capture Switch", MIXER_CTL_TYPE_BOOL}}},
                {Mixer::MIC_GAIN, {{"Capture Volume", MIXER_CTL_TYPE_INT}}}};

// static
Mixer::Controls Mixer::initializeMixerControls(struct mixer* mixer) {
    if (mixer == nullptr) return {};
    Controls mixerControls;
    std::string mixerCtlNames;
    for (const auto& [control, possibleCtls] : kPossibleControls) {
        for (const auto& [ctlName, expectedCtlType] : possibleCtls) {
            struct mixer_ctl* ctl = mixer_get_ctl_by_name(mixer, ctlName.c_str());
            if (ctl != nullptr && mixer_ctl_get_type(ctl) == expectedCtlType) {
                mixerControls.emplace(control, ctl);
                if (!mixerCtlNames.empty()) {
                    mixerCtlNames += ",";
                }
                mixerCtlNames += ctlName;
                break;
            }
        }
    }
    LOG(DEBUG) << __func__ << ": available mixer control names=[" << mixerCtlNames << "]";
    return mixerControls;
}

std::ostream& operator<<(std::ostream& s, Mixer::Control c) {
    switch (c) {
        case Mixer::Control::MASTER_SWITCH:
            s << "master mute";
            break;
        case Mixer::Control::MASTER_VOLUME:
            s << "master volume";
            break;
        case Mixer::Control::HW_VOLUME:
            s << "volume";
            break;
        case Mixer::Control::MIC_SWITCH:
            s << "mic mute";
            break;
        case Mixer::Control::MIC_GAIN:
            s << "mic gain";
            break;
    }
    return s;
}

Mixer::Mixer(int card) : mMixer(mixer_open(card)), mMixerControls(initializeMixerControls(mMixer)) {
    if (!isValid()) {
        PLOG(ERROR) << __func__ << ": failed to open mixer for card=" << card;
    }
}

Mixer::~Mixer() {
    if (isValid()) {
        std::lock_guard l(mMixerAccess);
        mixer_close(mMixer);
    }
}

ndk::ScopedAStatus Mixer::getMasterMute(bool* muted) {
    return getMixerControlMute(MASTER_SWITCH, muted);
}

ndk::ScopedAStatus Mixer::getMasterVolume(float* volume) {
    return getMixerControlVolume(MASTER_VOLUME, volume);
}

ndk::ScopedAStatus Mixer::getMicGain(float* gain) {
    return getMixerControlVolume(MIC_GAIN, gain);
}

ndk::ScopedAStatus Mixer::getMicMute(bool* muted) {
    return getMixerControlMute(MIC_SWITCH, muted);
}

ndk::ScopedAStatus Mixer::getVolumes(std::vector<float>* volumes) {
    struct mixer_ctl* mctl;
    RETURN_STATUS_IF_ERROR(findControl(Mixer::HW_VOLUME, &mctl));
    std::vector<int> percents;
    std::lock_guard l(mMixerAccess);
    if (int err = getMixerControlPercent(mctl, &percents); err != 0) {
        LOG(ERROR) << __func__ << ": failed to get volume, err=" << err;
        return ndk::ScopedAStatus::fromExceptionCode(EX_ILLEGAL_STATE);
    }
    std::transform(percents.begin(), percents.end(), std::back_inserter(*volumes),
                   [](int percent) -> float { return std::clamp(percent / 100.0f, 0.0f, 1.0f); });
    return ndk::ScopedAStatus::ok();
}

ndk::ScopedAStatus Mixer::setMasterMute(bool muted) {
    return setMixerControlMute(MASTER_SWITCH, muted);
}

ndk::ScopedAStatus Mixer::setMasterVolume(float volume) {
    return setMixerControlVolume(MASTER_VOLUME, volume);
}

ndk::ScopedAStatus Mixer::setMicGain(float gain) {
    return setMixerControlVolume(MIC_GAIN, gain);
}

ndk::ScopedAStatus Mixer::setMicMute(bool muted) {
    return setMixerControlMute(MIC_SWITCH, muted);
}

ndk::ScopedAStatus Mixer::setVolumes(const std::vector<float>& volumes) {
    struct mixer_ctl* mctl;
    RETURN_STATUS_IF_ERROR(findControl(Mixer::HW_VOLUME, &mctl));
    std::vector<int> percents;
    std::transform(
            volumes.begin(), volumes.end(), std::back_inserter(percents),
            [](float volume) -> int { return std::floor(std::clamp(volume, 0.0f, 1.0f) * 100); });
    std::lock_guard l(mMixerAccess);
    if (int err = setMixerControlPercent(mctl, percents); err != 0) {
        LOG(ERROR) << __func__ << ": failed to set volume, err=" << err;
        return ndk::ScopedAStatus::fromExceptionCode(EX_ILLEGAL_STATE);
    }
    return ndk::ScopedAStatus::ok();
}

ndk::ScopedAStatus Mixer::findControl(Control ctl, struct mixer_ctl** result) {
    if (!isValid()) {
        return ndk::ScopedAStatus::fromExceptionCode(EX_ILLEGAL_STATE);
    }
    if (auto it = mMixerControls.find(ctl); it != mMixerControls.end()) {
        *result = it->second;
        return ndk::ScopedAStatus::ok();
    }
    return ndk::ScopedAStatus::fromExceptionCode(EX_UNSUPPORTED_OPERATION);
}

ndk::ScopedAStatus Mixer::getMixerControlMute(Control ctl, bool* muted) {
    struct mixer_ctl* mctl;
    RETURN_STATUS_IF_ERROR(findControl(ctl, &mctl));
    std::lock_guard l(mMixerAccess);
    std::vector<int> mutedValues;
    if (int err = getMixerControlValues(mctl, &mutedValues); err != 0) {
        LOG(ERROR) << __func__ << ": failed to get " << ctl << ", err=" << err;
        return ndk::ScopedAStatus::fromExceptionCode(EX_ILLEGAL_STATE);
    }
    if (mutedValues.empty()) {
        LOG(ERROR) << __func__ << ": got no values for " << ctl;
        return ndk::ScopedAStatus::fromExceptionCode(EX_ILLEGAL_STATE);
    }
    *muted = mutedValues[0] != 0;
    return ndk::ScopedAStatus::ok();
}

ndk::ScopedAStatus Mixer::getMixerControlVolume(Control ctl, float* volume) {
    struct mixer_ctl* mctl;
    RETURN_STATUS_IF_ERROR(findControl(ctl, &mctl));
    std::lock_guard l(mMixerAccess);
    std::vector<int> percents;
    if (int err = getMixerControlPercent(mctl, &percents); err != 0) {
        LOG(ERROR) << __func__ << ": failed to get " << ctl << ", err=" << err;
        return ndk::ScopedAStatus::fromExceptionCode(EX_ILLEGAL_STATE);
    }
    if (percents.empty()) {
        LOG(ERROR) << __func__ << ": got no values for " << ctl;
        return ndk::ScopedAStatus::fromExceptionCode(EX_ILLEGAL_STATE);
    }
    *volume = std::clamp(percents[0] / 100.0f, 0.0f, 1.0f);
    return ndk::ScopedAStatus::ok();
}

ndk::ScopedAStatus Mixer::setMixerControlMute(Control ctl, bool muted) {
    struct mixer_ctl* mctl;
    RETURN_STATUS_IF_ERROR(findControl(ctl, &mctl));
    std::lock_guard l(mMixerAccess);
    if (int err = setMixerControlValue(mctl, muted ? 0 : 1); err != 0) {
        LOG(ERROR) << __func__ << ": failed to set " << ctl << " to " << muted << ", err=" << err;
        return ndk::ScopedAStatus::fromExceptionCode(EX_ILLEGAL_STATE);
    }
    return ndk::ScopedAStatus::ok();
}

ndk::ScopedAStatus Mixer::setMixerControlVolume(Control ctl, float volume) {
    struct mixer_ctl* mctl;
    RETURN_STATUS_IF_ERROR(findControl(ctl, &mctl));
    volume = std::clamp(volume, 0.0f, 1.0f);
    std::lock_guard l(mMixerAccess);
    if (int err = setMixerControlPercent(mctl, std::floor(volume * 100)); err != 0) {
        LOG(ERROR) << __func__ << ": failed to set " << ctl << " to " << volume << ", err=" << err;
        return ndk::ScopedAStatus::fromExceptionCode(EX_ILLEGAL_STATE);
    }
    return ndk::ScopedAStatus::ok();
}

int Mixer::getMixerControlPercent(struct mixer_ctl* ctl, std::vector<int>* percents) {
    const unsigned int n = mixer_ctl_get_num_values(ctl);
    percents->resize(n);
    for (unsigned int id = 0; id < n; id++) {
        if (int valueOrError = mixer_ctl_get_percent(ctl, id); valueOrError >= 0) {
            (*percents)[id] = valueOrError;
        } else {
            return valueOrError;
        }
    }
    return 0;
}

int Mixer::getMixerControlValues(struct mixer_ctl* ctl, std::vector<int>* values) {
    const unsigned int n = mixer_ctl_get_num_values(ctl);
    values->resize(n);
    for (unsigned int id = 0; id < n; id++) {
        if (int valueOrError = mixer_ctl_get_value(ctl, id); valueOrError >= 0) {
            (*values)[id] = valueOrError;
        } else {
            return valueOrError;
        }
    }
    return 0;
}

int Mixer::setMixerControlPercent(struct mixer_ctl* ctl, int percent) {
    const unsigned int n = mixer_ctl_get_num_values(ctl);
    for (unsigned int id = 0; id < n; id++) {
        if (int error = mixer_ctl_set_percent(ctl, id, percent); error != 0) {
            return error;
        }
    }
    return 0;
}

int Mixer::setMixerControlPercent(struct mixer_ctl* ctl, const std::vector<int>& percents) {
    const unsigned int n = mixer_ctl_get_num_values(ctl);
    for (unsigned int id = 0; id < n; id++) {
        if (int error = mixer_ctl_set_percent(ctl, id, id < percents.size() ? percents[id] : 0);
            error != 0) {
            return error;
        }
    }
    return 0;
}

int Mixer::setMixerControlValue(struct mixer_ctl* ctl, int value) {
    const unsigned int n = mixer_ctl_get_num_values(ctl);
    for (unsigned int id = 0; id < n; id++) {
        if (int error = mixer_ctl_set_value(ctl, id, value); error != 0) {
            return error;
        }
    }
    return 0;
}

}  // namespace aidl::android::hardware::audio::core::alsa
➜  alsa git:(1a56e38edc) ✗

```

ModuleAlsa.cpp

```bash

➜  alsa git:(1a56e38edc) ✗ cat ModuleAlsa.cpp 
/*
 * Copyright (C) 2023 The Android Open Source Project
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

#define LOG_TAG "AHAL_ModuleAlsa"

#include <vector>

#include <android-base/logging.h>

#include "Utils.h"
#include "core-impl/ModuleAlsa.h"

extern "C" {
#include "alsa_device_profile.h"
}

using aidl::android::media::audio::common::AudioChannelLayout;
using aidl::android::media::audio::common::AudioFormatType;
using aidl::android::media::audio::common::AudioPort;
using aidl::android::media::audio::common::AudioProfile;

namespace aidl::android::hardware::audio::core {

ndk::ScopedAStatus ModuleAlsa::populateConnectedDevicePort(AudioPort* audioPort, int32_t) {
    auto deviceProfile = alsa::getDeviceProfile(*audioPort);
    if (!deviceProfile.has_value()) {
        return ndk::ScopedAStatus::fromExceptionCode(EX_ILLEGAL_ARGUMENT);
    }
    auto proxy = alsa::readAlsaDeviceInfo(*deviceProfile);
    if (proxy.get() == nullptr) {
        return ndk::ScopedAStatus::fromExceptionCode(EX_ILLEGAL_STATE);
    }

    alsa_device_profile* profile = proxy.getProfile();
    std::vector<AudioChannelLayout> channels = alsa::getChannelMasksFromProfile(profile);
    std::vector<int> sampleRates = alsa::getSampleRatesFromProfile(profile);

    for (size_t i = 0; i < std::min(MAX_PROFILE_FORMATS, AUDIO_PORT_MAX_AUDIO_PROFILES) &&
                       profile->formats[i] != PCM_FORMAT_INVALID;
         ++i) {
        auto audioFormatDescription =
                alsa::c2aidl_pcm_format_AudioFormatDescription(profile->formats[i]);
        if (audioFormatDescription.type == AudioFormatType::DEFAULT) {
            LOG(WARNING) << __func__ << ": unknown pcm type=" << profile->formats[i];
            continue;
        }
        AudioProfile audioProfile = {.format = audioFormatDescription,
                                     .channelMasks = channels,
                                     .sampleRates = sampleRates};
        audioPort->profiles.push_back(std::move(audioProfile));
    }
    return ndk::ScopedAStatus::ok();
}

}  // namespace aidl::android::hardware::audio::core
➜  alsa git:(1a56e38edc) ✗
```

Utils.cpp

```bash
➜  alsa git:(1a56e38edc) ✗ cat Utils.cpp 
/*
 * Copyright (C) 2023 The Android Open Source Project
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

#include <map>
#include <set>

#define LOG_TAG "AHAL_AlsaUtils"
#include <Utils.h>
#include <aidl/android/media/audio/common/AudioFormatType.h>
#include <aidl/android/media/audio/common/PcmType.h>
#include <android-base/logging.h>
#include <audio_utils/primitives.h>
#include <cutils/compiler.h>

#include "Utils.h"
#include "core-impl/utils.h"

using aidl::android::hardware::audio::common::getChannelCount;
using aidl::android::media::audio::common::AudioChannelLayout;
using aidl::android::media::audio::common::AudioDeviceAddress;
using aidl::android::media::audio::common::AudioFormatDescription;
using aidl::android::media::audio::common::AudioFormatType;
using aidl::android::media::audio::common::AudioIoFlags;
using aidl::android::media::audio::common::AudioPortExt;
using aidl::android::media::audio::common::PcmType;

namespace aidl::android::hardware::audio::core::alsa {

const float kUnityGainFloat = 1.0f;

DeviceProxy::DeviceProxy() : mProfile(nullptr), mProxy(nullptr, alsaProxyDeleter) {}

DeviceProxy::DeviceProxy(const DeviceProfile& deviceProfile)
    : mProfile(new alsa_device_profile), mProxy(new alsa_device_proxy, alsaProxyDeleter) {
    profile_init(mProfile.get(), deviceProfile.direction);
    mProfile->card = deviceProfile.card;
    mProfile->device = deviceProfile.device;
    memset(mProxy.get(), 0, sizeof(alsa_device_proxy));
}

void DeviceProxy::alsaProxyDeleter(alsa_device_proxy* proxy) {
    if (proxy != nullptr) {
        proxy_close(proxy);
        delete proxy;
    }
}

namespace {

using AudioChannelCountToMaskMap = std::map<unsigned int, AudioChannelLayout>;
using AudioFormatDescToPcmFormatMap = std::map<AudioFormatDescription, enum pcm_format>;
using PcmFormatToAudioFormatDescMap = std::map<enum pcm_format, AudioFormatDescription>;

AudioChannelLayout getInvalidChannelLayout() {
    static const AudioChannelLayout invalidChannelLayout =
            AudioChannelLayout::make<AudioChannelLayout::Tag::invalid>(0);
    return invalidChannelLayout;
}

static AudioChannelCountToMaskMap make_ChannelCountToMaskMap(
        const std::set<AudioChannelLayout>& channelMasks) {
    AudioChannelCountToMaskMap channelMaskToCountMap;
    for (const auto& channelMask : channelMasks) {
        channelMaskToCountMap.emplace(getChannelCount(channelMask), channelMask);
    }
    return channelMaskToCountMap;
}

#define DEFINE_CHANNEL_LAYOUT_MASK(n) \
    AudioChannelLayout::make<AudioChannelLayout::Tag::layoutMask>(AudioChannelLayout::LAYOUT_##n)

const AudioChannelCountToMaskMap& getSupportedChannelOutLayoutMap() {
    static const std::set<AudioChannelLayout> supportedOutChannelLayouts = {
            DEFINE_CHANNEL_LAYOUT_MASK(MONO),
            DEFINE_CHANNEL_LAYOUT_MASK(STEREO),
    };
    static const AudioChannelCountToMaskMap outLayouts =
            make_ChannelCountToMaskMap(supportedOutChannelLayouts);
    return outLayouts;
}

const AudioChannelCountToMaskMap& getSupportedChannelInLayoutMap() {
    static const std::set<AudioChannelLayout> supportedInChannelLayouts = {
            DEFINE_CHANNEL_LAYOUT_MASK(MONO),
            DEFINE_CHANNEL_LAYOUT_MASK(STEREO),
    };
    static const AudioChannelCountToMaskMap inLayouts =
            make_ChannelCountToMaskMap(supportedInChannelLayouts);
    return inLayouts;
}

#undef DEFINE_CHANNEL_LAYOUT_MASK
#define DEFINE_CHANNEL_INDEX_MASK(n) \
    AudioChannelLayout::make<AudioChannelLayout::Tag::indexMask>(AudioChannelLayout::INDEX_MASK_##n)

const AudioChannelCountToMaskMap& getSupportedChannelIndexLayoutMap() {
    static const std::set<AudioChannelLayout> supportedIndexChannelLayouts = {
            DEFINE_CHANNEL_INDEX_MASK(1),  DEFINE_CHANNEL_INDEX_MASK(2),
            DEFINE_CHANNEL_INDEX_MASK(3),  DEFINE_CHANNEL_INDEX_MASK(4),
            DEFINE_CHANNEL_INDEX_MASK(5),  DEFINE_CHANNEL_INDEX_MASK(6),
            DEFINE_CHANNEL_INDEX_MASK(7),  DEFINE_CHANNEL_INDEX_MASK(8),
            DEFINE_CHANNEL_INDEX_MASK(9),  DEFINE_CHANNEL_INDEX_MASK(10),
            DEFINE_CHANNEL_INDEX_MASK(11), DEFINE_CHANNEL_INDEX_MASK(12),
            DEFINE_CHANNEL_INDEX_MASK(13), DEFINE_CHANNEL_INDEX_MASK(14),
            DEFINE_CHANNEL_INDEX_MASK(15), DEFINE_CHANNEL_INDEX_MASK(16),
            DEFINE_CHANNEL_INDEX_MASK(17), DEFINE_CHANNEL_INDEX_MASK(18),
            DEFINE_CHANNEL_INDEX_MASK(19), DEFINE_CHANNEL_INDEX_MASK(20),
            DEFINE_CHANNEL_INDEX_MASK(21), DEFINE_CHANNEL_INDEX_MASK(22),
            DEFINE_CHANNEL_INDEX_MASK(23), DEFINE_CHANNEL_INDEX_MASK(24),
    };
    static const AudioChannelCountToMaskMap indexLayouts =
            make_ChannelCountToMaskMap(supportedIndexChannelLayouts);
    return indexLayouts;
}

#undef DEFINE_CHANNEL_INDEX_MASK

AudioFormatDescription make_AudioFormatDescription(AudioFormatType type) {
    AudioFormatDescription result;
    result.type = type;
    return result;
}

AudioFormatDescription make_AudioFormatDescription(PcmType pcm) {
    auto result = make_AudioFormatDescription(AudioFormatType::PCM);
    result.pcm = pcm;
    return result;
}

const AudioFormatDescToPcmFormatMap& getAudioFormatDescriptorToPcmFormatMap() {
    static const AudioFormatDescToPcmFormatMap formatDescToPcmFormatMap = {
            {make_AudioFormatDescription(PcmType::INT_16_BIT), PCM_FORMAT_S16_LE},
            {make_AudioFormatDescription(PcmType::FIXED_Q_8_24), PCM_FORMAT_S24_LE},
            {make_AudioFormatDescription(PcmType::INT_24_BIT), PCM_FORMAT_S24_3LE},
            {make_AudioFormatDescription(PcmType::INT_32_BIT), PCM_FORMAT_S32_LE},
            {make_AudioFormatDescription(PcmType::FLOAT_32_BIT), PCM_FORMAT_FLOAT_LE},
    };
    return formatDescToPcmFormatMap;
}

static PcmFormatToAudioFormatDescMap make_PcmFormatToAudioFormatDescMap(
        const AudioFormatDescToPcmFormatMap& formatDescToPcmFormatMap) {
    PcmFormatToAudioFormatDescMap result;
    for (const auto& formatPair : formatDescToPcmFormatMap) {
        result.emplace(formatPair.second, formatPair.first);
    }
    return result;
}

const PcmFormatToAudioFormatDescMap& getPcmFormatToAudioFormatDescMap() {
    static const PcmFormatToAudioFormatDescMap pcmFormatToFormatDescMap =
            make_PcmFormatToAudioFormatDescMap(getAudioFormatDescriptorToPcmFormatMap());
    return pcmFormatToFormatDescMap;
}

void applyGainToInt16Buffer(void* buffer, const size_t bufferSizeBytes, const float gain,
                            int channelCount) {
    const uint16_t unityGainQ4_12 = u4_12_from_float(kUnityGainFloat);
    const uint16_t vl = u4_12_from_float(gain);
    const uint32_t vrl = (vl << 16) | vl;
    int numFrames = 0;
    if (channelCount == 2) {
        numFrames = bufferSizeBytes / sizeof(uint32_t);
        if (numFrames == 0) {
            return;
        }
        uint32_t* intBuffer = (uint32_t*)buffer;
        if (CC_UNLIKELY(vl > unityGainQ4_12)) {
            do {
                int32_t l = mulRL(1, *intBuffer, vrl) >> 12;
                int32_t r = mulRL(0, *intBuffer, vrl) >> 12;
                l = clamp16(l);
                r = clamp16(r);
                *intBuffer++ = (r << 16) | (l & 0xFFFF);
            } while (--numFrames);
        } else {
            do {
                int32_t l = mulRL(1, *intBuffer, vrl) >> 12;
                int32_t r = mulRL(0, *intBuffer, vrl) >> 12;
                *intBuffer++ = (r << 16) | (l & 0xFFFF);
            } while (--numFrames);
        }
    } else {
        numFrames = bufferSizeBytes / sizeof(uint16_t);
        if (numFrames == 0) {
            return;
        }
        int16_t* intBuffer = (int16_t*)buffer;
        if (CC_UNLIKELY(vl > unityGainQ4_12)) {
            do {
                int32_t mono = mul(*intBuffer, static_cast<int16_t>(vl)) >> 12;
                *intBuffer++ = clamp16(mono);
            } while (--numFrames);
        } else {
            do {
                int32_t mono = mul(*intBuffer, static_cast<int16_t>(vl)) >> 12;
                *intBuffer++ = static_cast<int16_t>(mono & 0xFFFF);
            } while (--numFrames);
        }
    }
}

void applyGainToInt32Buffer(int32_t* typedBuffer, const size_t bufferSizeBytes, const float gain) {
    int numSamples = bufferSizeBytes / sizeof(int32_t);
    if (numSamples == 0) {
        return;
    }
    if (CC_UNLIKELY(gain > kUnityGainFloat)) {
        do {
            float multiplied = (*typedBuffer) * gain;
            if (multiplied > INT32_MAX) {
                *typedBuffer++ = INT32_MAX;
            } else if (multiplied < INT32_MIN) {
                *typedBuffer++ = INT32_MIN;
            } else {
                *typedBuffer++ = multiplied;
            }
        } while (--numSamples);
    } else {
        do {
            *typedBuffer++ = (*typedBuffer) * gain;
        } while (--numSamples);
    }
}

void applyGainToFloatBuffer(float* floatBuffer, const size_t bufferSizeBytes, const float gain) {
    int numSamples = bufferSizeBytes / sizeof(float);
    if (numSamples == 0) {
        return;
    }
    if (CC_UNLIKELY(gain > kUnityGainFloat)) {
        do {
            *floatBuffer++ = std::clamp((*floatBuffer) * gain, -kUnityGainFloat, kUnityGainFloat);
        } while (--numSamples);
    } else {
        do {
            *floatBuffer++ = (*floatBuffer) * gain;
        } while (--numSamples);
    }
}

}  // namespace

std::ostream& operator<<(std::ostream& os, const DeviceProfile& device) {
    return os << "<" << device.card << "," << device.device << ">";
}

AudioChannelLayout getChannelLayoutMaskFromChannelCount(unsigned int channelCount, int isInput) {
    return findValueOrDefault(
            isInput ? getSupportedChannelInLayoutMap() : getSupportedChannelOutLayoutMap(),
            channelCount, getInvalidChannelLayout());
}

AudioChannelLayout getChannelIndexMaskFromChannelCount(unsigned int channelCount) {
    return findValueOrDefault(getSupportedChannelIndexLayoutMap(), channelCount,
                              getInvalidChannelLayout());
}

unsigned int getChannelCountFromChannelMask(const AudioChannelLayout& channelMask, bool isInput) {
    switch (channelMask.getTag()) {
        case AudioChannelLayout::Tag::layoutMask: {
            return findKeyOrDefault(
                    isInput ? getSupportedChannelInLayoutMap() : getSupportedChannelOutLayoutMap(),
                    static_cast<unsigned>(getChannelCount(channelMask)), 0u /*defaultValue*/);
        }
        case AudioChannelLayout::Tag::indexMask: {
            return findKeyOrDefault(getSupportedChannelIndexLayoutMap(),
                                    static_cast<unsigned>(getChannelCount(channelMask)),
                                    0u /*defaultValue*/);
        }
        case AudioChannelLayout::Tag::none:
        case AudioChannelLayout::Tag::invalid:
        case AudioChannelLayout::Tag::voiceMask:
        default:
            return 0;
    }
}

std::vector<AudioChannelLayout> getChannelMasksFromProfile(const alsa_device_profile* profile) {
    const bool isInput = profile->direction == PCM_IN;
    std::vector<AudioChannelLayout> channels;
    for (size_t i = 0; i < AUDIO_PORT_MAX_CHANNEL_MASKS && profile->channel_counts[i] != 0; ++i) {
        auto layoutMask =
                alsa::getChannelLayoutMaskFromChannelCount(profile->channel_counts[i], isInput);
        if (layoutMask.getTag() == AudioChannelLayout::Tag::layoutMask) {
            channels.push_back(layoutMask);
        }
        auto indexMask = alsa::getChannelIndexMaskFromChannelCount(profile->channel_counts[i]);
        if (indexMask.getTag() == AudioChannelLayout::Tag::indexMask) {
            channels.push_back(indexMask);
        }
    }
    return channels;
}

std::optional<DeviceProfile> getDeviceProfile(
        const ::aidl::android::media::audio::common::AudioDevice& audioDevice, bool isInput) {
    if (audioDevice.address.getTag() != AudioDeviceAddress::Tag::alsa) {
        LOG(ERROR) << __func__ << ": not alsa address: " << audioDevice.toString();
        return std::nullopt;
    }
    auto& alsaAddress = audioDevice.address.get<AudioDeviceAddress::Tag::alsa>();
    if (alsaAddress.size() != 2 || alsaAddress[0] < 0 || alsaAddress[1] < 0) {
        LOG(ERROR) << __func__
                   << ": malformed alsa address: " << ::android::internal::ToString(alsaAddress);
        return std::nullopt;
    }
    return DeviceProfile{.card = alsaAddress[0],
                         .device = alsaAddress[1],
                         .direction = isInput ? PCM_IN : PCM_OUT,
                         .isExternal = !audioDevice.type.connection.empty()};
}

std::optional<DeviceProfile> getDeviceProfile(
        const ::aidl::android::media::audio::common::AudioPort& audioPort) {
    if (audioPort.ext.getTag() != AudioPortExt::Tag::device) {
        LOG(ERROR) << __func__ << ": port id " << audioPort.id << " is not a device port";
        return std::nullopt;
    }
    auto& devicePort = audioPort.ext.get<AudioPortExt::Tag::device>();
    return getDeviceProfile(devicePort.device, audioPort.flags.getTag() == AudioIoFlags::input);
}

std::optional<struct pcm_config> getPcmConfig(const StreamContext& context, bool isInput) {
    struct pcm_config config;
    config.channels = alsa::getChannelCountFromChannelMask(context.getChannelLayout(), isInput);
    if (config.channels == 0) {
        LOG(ERROR) << __func__ << ": invalid channel=" << context.getChannelLayout().toString();
        return std::nullopt;
    }
    config.format = alsa::aidl2c_AudioFormatDescription_pcm_format(context.getFormat());
    if (config.format == PCM_FORMAT_INVALID) {
        LOG(ERROR) << __func__ << ": invalid format=" << context.getFormat().toString();
        return std::nullopt;
    }
    config.rate = context.getSampleRate();
    if (config.rate == 0) {
        LOG(ERROR) << __func__ << ": invalid sample rate=" << config.rate;
        return std::nullopt;
    }
    return config;
}

std::vector<int> getSampleRatesFromProfile(const alsa_device_profile* profile) {
    std::vector<int> sampleRates;
    for (int i = 0; i < std::min(MAX_PROFILE_SAMPLE_RATES, AUDIO_PORT_MAX_SAMPLING_RATES) &&
                    profile->sample_rates[i] != 0;
         i++) {
        sampleRates.push_back(profile->sample_rates[i]);
    }
    return sampleRates;
}

DeviceProxy openProxyForAttachedDevice(const DeviceProfile& deviceProfile,
                                       struct pcm_config* pcmConfig, size_t bufferFrameCount) {
    if (deviceProfile.isExternal) {
        LOG(FATAL) << __func__ << ": called for an external device, address=" << deviceProfile;
    }
    DeviceProxy proxy(deviceProfile);
    if (!profile_fill_builtin_device_info(proxy.getProfile(), pcmConfig, bufferFrameCount)) {
        LOG(FATAL) << __func__ << ": failed to init for built-in device, address=" << deviceProfile;
    }
    if (int err = proxy_prepare_from_default_config(proxy.get(), proxy.getProfile()); err != 0) {
        LOG(FATAL) << __func__ << ": fail to prepare for device address=" << deviceProfile
                   << " error=" << err;
        return DeviceProxy();
    }
    if (int err = proxy_open(proxy.get()); err != 0) {
        LOG(ERROR) << __func__ << ": failed to open device, address=" << deviceProfile
                   << " error=" << err;
        return DeviceProxy();
    }
    return proxy;
}

DeviceProxy openProxyForExternalDevice(const DeviceProfile& deviceProfile,
                                       struct pcm_config* pcmConfig, bool requireExactMatch) {
    if (!deviceProfile.isExternal) {
        LOG(FATAL) << __func__ << ": called for an attached device, address=" << deviceProfile;
    }
    auto proxy = readAlsaDeviceInfo(deviceProfile);
    if (proxy.get() == nullptr) {
        return proxy;
    }
    if (int err = proxy_prepare(proxy.get(), proxy.getProfile(), pcmConfig, requireExactMatch);
        err != 0) {
        LOG(ERROR) << __func__ << ": fail to prepare for device address=" << deviceProfile
                   << " error=" << err;
        return DeviceProxy();
    }
    if (int err = proxy_open(proxy.get()); err != 0) {
        LOG(ERROR) << __func__ << ": failed to open device, address=" << deviceProfile
                   << " error=" << err;
        return DeviceProxy();
    }
    return proxy;
}

DeviceProxy readAlsaDeviceInfo(const DeviceProfile& deviceProfile) {
    DeviceProxy proxy(deviceProfile);
    if (!profile_read_device_info(proxy.getProfile())) {
        LOG(ERROR) << __func__ << ": unable to read device info, device address=" << deviceProfile;
        return DeviceProxy();
    }
    return proxy;
}

void resetTransferredFrames(DeviceProxy& proxy, uint64_t frames) {
    if (proxy.get() != nullptr) {
        proxy.get()->transferred = frames;
    }
}

AudioFormatDescription c2aidl_pcm_format_AudioFormatDescription(enum pcm_format legacy) {
    return findValueOrDefault(getPcmFormatToAudioFormatDescMap(), legacy, AudioFormatDescription());
}

pcm_format aidl2c_AudioFormatDescription_pcm_format(const AudioFormatDescription& aidl) {
    return findValueOrDefault(getAudioFormatDescriptorToPcmFormatMap(), aidl, PCM_FORMAT_INVALID);
}

void applyGain(void* buffer, float gain, size_t bufferSizeBytes, enum pcm_format pcmFormat,
               int channelCount) {
    if (channelCount != 1 && channelCount != 2) {
        LOG(WARNING) << __func__ << ": unsupported channel count " << channelCount;
        return;
    }
    if (!getPcmFormatToAudioFormatDescMap().contains(pcmFormat)) {
        LOG(WARNING) << __func__ << ": unsupported pcm format " << pcmFormat;
        return;
    }
    if (std::abs(gain - kUnityGainFloat) < 1e-6) {
        return;
    }
    switch (pcmFormat) {
        case PCM_FORMAT_S16_LE:
            applyGainToInt16Buffer(buffer, bufferSizeBytes, gain, channelCount);
            break;
        case PCM_FORMAT_FLOAT_LE: {
            float* floatBuffer = (float*)buffer;
            applyGainToFloatBuffer(floatBuffer, bufferSizeBytes, gain);
        } break;
        case PCM_FORMAT_S24_LE:
            // PCM_FORMAT_S24_LE buffer is composed of signed fixed-point 32-bit Q8.23 data with
            // min and max limits of the same bit representation as min and max limits of
            // PCM_FORMAT_S32_LE buffer.
        case PCM_FORMAT_S32_LE: {
            int32_t* typedBuffer = (int32_t*)buffer;
            applyGainToInt32Buffer(typedBuffer, bufferSizeBytes, gain);
        } break;
        case PCM_FORMAT_S24_3LE: {
            int numSamples = bufferSizeBytes / (sizeof(uint8_t) * 3);
            if (numSamples == 0) {
                return;
            }
            std::unique_ptr<int32_t[]> typedBuffer(new int32_t[numSamples]);
            memcpy_to_i32_from_p24(typedBuffer.get(), (uint8_t*)buffer, numSamples);
            applyGainToInt32Buffer(typedBuffer.get(), numSamples * sizeof(int32_t), gain);
            memcpy_to_p24_from_i32((uint8_t*)buffer, typedBuffer.get(), numSamples);
        } break;
        default:
            LOG(FATAL) << __func__ << ": unsupported pcm format " << pcmFormat;
            break;
    }
}

}  // namespace aidl::android::hardware::audio::core::alsa
➜  alsa git:(1a56e38edc) ✗

```

