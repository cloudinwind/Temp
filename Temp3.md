```bash
➜  cloudinwindblog git:(master) ✗ cat aa.diff
diff --git a/.gitignore b/.gitignore
index 3a8f11a..1cb8111 100644
--- a/.gitignore
+++ b/.gitignore
@@ -36,4 +36,7 @@ Firefly-docs/

 cache/
 package/
-.obsidian/
\ No newline at end of file
+.obsidian/
+
+# 说说同步输出（由 scripts/fetch-talks.ts 生成）
+src/content/talks.json
\ No newline at end of file
diff --git a/astro.config.mjs b/astro.config.mjs
index eb93f5f..802e4a8 100644
--- a/astro.config.mjs
+++ b/astro.config.mjs
@@ -196,6 +196,9 @@ export default defineConfig({
                                if (pathname === "/gallery/" && !siteConfig.pages.gallery) {
                                        return false;
                                }
+                               if (pathname === "/talks/" && !siteConfig.pages.talks) {
+                                       return false;
+                               }

                                return true;
                        },
diff --git a/package.json b/package.json
index 4218da0..d8c4a68 100644
--- a/package.json
+++ b/package.json
@@ -6,7 +6,7 @@
     "dev": "astro dev",
     "start": "astro dev",
     "check": "astro check",
-    "build": "node scripts/generate-icons.js && npx tsx scripts/generate-lqips.ts && astro build && pagefind --site dist",
+    "build": "node scripts/generate-icons.js && npx tsx scripts/generate-lqips.ts && npx tsx scripts/fetch-talks.ts && astro build && pagefind --site dist",
     "preview": "astro preview",
     "astro": "astro",
     "type-check": "tsc --noEmit --isolatedDeclarations",
@@ -15,7 +15,8 @@
     "lint": "biome check --write ./src",
     "preinstall": "npx only-allow pnpm",
     "icons": "node scripts/generate-icons.js",
-    "lqips": "npx tsx scripts/generate-lqips.ts"
+    "lqips": "npx tsx scripts/generate-lqips.ts",
+    "fetch-talks": "npx tsx scripts/fetch-talks.ts"
   },
   "dependencies": {
     "@astrojs/check": "^0.9.9",
diff --git a/src/config/navBarConfig.ts b/src/config/navBarConfig.ts
index bb4cdeb..f1f29b1 100644
--- a/src/config/navBarConfig.ts
+++ b/src/config/navBarConfig.ts
@@ -40,6 +40,11 @@ const getDynamicNavBarConfig = (): NavBarConfig => {
                links.push(LinkPreset.Guestbook);
        }

+       // 根据配置决定是否添加说说，在siteConfig关闭pages.talks时导航栏不显示说说
+       if (siteConfig.pages.talks) {
+               links.push(LinkPreset.Talks);
+       }
+
        // 我的及其子菜单
        links.push({
                name: "我的",
diff --git a/src/config/siteConfig.ts b/src/config/siteConfig.ts
index fdc7e1f..681420f 100644
--- a/src/config/siteConfig.ts
+++ b/src/config/siteConfig.ts
@@ -146,6 +146,8 @@ export const siteConfig: SiteConfig = {
                bangumi: true,
                // 相册页面开关
                gallery: true,
+               // 说说页面开关（GoToSocial/Mastodon 同步），需配置 .env 中的 GTS_INSTANCE_URL 与 GTS_ACCESS_TOKEN
+               talks: true,
        },

        // 分类导航栏开关，在首页和归档页顶部显示分类快捷导航
diff --git a/src/constants/link-presets.ts b/src/constants/link-presets.ts
index 09b91db..d15c438 100644
--- a/src/constants/link-presets.ts
+++ b/src/constants/link-presets.ts
@@ -53,4 +53,9 @@ export const LinkPresets: { [key in LinkPreset]: NavBarLink } = {
                url: "/categories/",
                icon: "material-symbols:folder-open-rounded",
        },
+       [LinkPreset.Talks]: {
+               name: i18n(I18nKey.talks),
+               url: "/talks/",
+               icon: "material-symbols:chat-bubble",
+       },
 };
diff --git a/src/env.d.ts b/src/env.d.ts
index 3eb535c..3cfd48c 100644
--- a/src/env.d.ts
+++ b/src/env.d.ts
@@ -4,6 +4,9 @@
 declare global {
        interface ImportMetaEnv {
                readonly MEILI_MASTER_KEY: string;
+               readonly GTS_INSTANCE_URL?: string;
+               readonly GTS_ACCESS_TOKEN?: string;
+               readonly GTS_TALKS_LIMIT?: string;
        }

        interface ITOCManager {
diff --git a/src/i18n/i18nKey.ts b/src/i18n/i18nKey.ts
index 0173168..82118ab 100644
--- a/src/i18n/i18nKey.ts
+++ b/src/i18n/i18nKey.ts
@@ -320,6 +320,12 @@ enum I18nKey {
        passwordSubmit = "passwordSubmit",
        passwordError = "passwordError",
        passwordProtectedRss = "passwordProtectedRss",
+
+       // 说说页面（GoToSocial/Mastodon 同步）
+       talks = "talks",
+       talksDescription = "talksDescription",
+       talksEmpty = "talksEmpty",
+       talksViewOriginal = "talksViewOriginal",
 }

 export default I18nKey;
diff --git a/src/i18n/languages/en.ts b/src/i18n/languages/en.ts
index c947bb5..08421f0 100644
--- a/src/i18n/languages/en.ts
+++ b/src/i18n/languages/en.ts
@@ -334,4 +334,9 @@ export const en: Translation = {
        [Key.passwordError]: "Incorrect password, please try again.",
        [Key.passwordProtectedRss]:
                "This article is encrypted. Please visit the website to view it.",
+
+       [Key.talks]: "Talks",
+       [Key.talksDescription]: "Short thoughts synced from the Fediverse",
+       [Key.talksEmpty]: "No talks yet — check back soon!",
+       [Key.talksViewOriginal]: "View original",
 };
diff --git a/src/i18n/languages/ja.ts b/src/i18n/languages/ja.ts
index 990b8a8..7c439fe 100644
--- a/src/i18n/languages/ja.ts
+++ b/src/i18n/languages/ja.ts
@@ -333,4 +333,9 @@ export const ja: Translation = {
        [Key.passwordError]: "パスワードが間違っています。もう一度お試しください。",
        [Key.passwordProtectedRss]:
                "この記事は暗号化されています。ウェブサイトにアクセスしてご覧ください。",
+
+       [Key.talks]: "つぶやき",
+       [Key.talksDescription]: "Fediverse から同期したつぶやき",
+       [Key.talksEmpty]: "まだつぶやきがありません。また後で確認してください～",
+       [Key.talksViewOriginal]: "原文を見る",
 };
diff --git a/src/i18n/languages/ru.ts b/src/i18n/languages/ru.ts
index ed00a59..55bfe29 100644
--- a/src/i18n/languages/ru.ts
+++ b/src/i18n/languages/ru.ts
@@ -335,4 +335,9 @@ export const ru: Translation = {
        [Key.passwordError]: "Неверный пароль, попробуйте снова.",
        [Key.passwordProtectedRss]:
                "Эта статья зашифрована. Пожалуйста, посетите сайт для просмотра.",
+
+       [Key.talks]: "Заметки",
+       [Key.talksDescription]: "Короткие мысли, синхронизированные из Fediverse",
+       [Key.talksEmpty]: "Заметок пока нет — загляните позже!",
+       [Key.talksViewOriginal]: "Открыть оригинал",
 };
diff --git a/src/i18n/languages/zh_CN.ts b/src/i18n/languages/zh_CN.ts
index d95c8a8..5efdb38 100644
--- a/src/i18n/languages/zh_CN.ts
+++ b/src/i18n/languages/zh_CN.ts
@@ -323,4 +323,9 @@ export const zh_CN: Translation = {
        [Key.passwordSubmit]: "解锁",
        [Key.passwordError]: "密码错误，请重试。",
        [Key.passwordProtectedRss]: "本文已加密保护，请访问网站查看。",
+
+       [Key.talks]: "说说",
+       [Key.talksDescription]: "这里是我同步自联邦宇宙的碎碎念",
+       [Key.talksEmpty]: "还没有说说，过段时间再来看看吧～",
+       [Key.talksViewOriginal]: "查看原文",
 };
diff --git a/src/i18n/languages/zh_TW.ts b/src/i18n/languages/zh_TW.ts
index bcd5399..3c517ee 100644
--- a/src/i18n/languages/zh_TW.ts
+++ b/src/i18n/languages/zh_TW.ts
@@ -325,4 +325,9 @@ export const zh_TW: Translation = {
        [Key.passwordSubmit]: "解鎖",
        [Key.passwordError]: "密碼錯誤，請重試。",
        [Key.passwordProtectedRss]: "本文已加密保護，請訪問網站查看。",
+
+       [Key.talks]: "說說",
+       [Key.talksDescription]: "這裡是我同步自聯邦宇宙的碎碎念",
+       [Key.talksEmpty]: "還沒有說說，過段時間再來看看吧～",
+       [Key.talksViewOriginal]: "查看原文",
 };
diff --git a/src/types/config.ts b/src/types/config.ts
index c2d37e0..8a418a0 100644
--- a/src/types/config.ts
+++ b/src/types/config.ts
@@ -89,6 +89,7 @@ export type SiteConfig = {
                guestbook: boolean; // 留言板页面开关
                bangumi: boolean;
                gallery: boolean; // 相册页面开关
+               talks: boolean; // 说说页面开关（GoToSocial/Mastodon 同步）
        };

        // 分类导航栏开关
@@ -184,6 +185,7 @@ export enum LinkPreset {
        Gallery = 7,
        Tags = 8,
        Categories = 9,
+       Talks = 10,
 }

 export type NavBarLink = {
➜  cloudinwindblog git:(master) ✗

```
