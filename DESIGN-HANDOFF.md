# Banana Loop Android / Fire TV — Design Handoff

Documentation only. Describes **1.0.88** (`versionCode` 89) as shipped. No UI, color, or behavior changes in this package.

The app source is **not** published as a GitHub source tree. Downloadable archives (also under `/opt/cursor/artifacts/`):

| Archive | Size | SHA-256 |
| --- | --- | --- |
| `banana-loop-apk-source.zip` | 32 955 076 bytes | `2529f5bf3389f64e4bdd4ab3384b1b8b345a54829f0a6c6f0906c3e076d47fdb` |
| `banana-loop-apk-assets.zip` | 633 924 bytes | `1f195055097b63eb301f274b1ff9a05577fdff676fd4682e38ada4e9ea670770` |

Latest runnable APK (item 7):  
https://github.com/raymoe34/banana-loop-apk/raw/main/BananaLoop-1.0.88-debug.apk

The GitHub repo `raymoe34/banana-loop-apk` holds **APK binaries only**, not this source tree.

---

## 2) Tech stack

| Layer | Choice |
| --- | --- |
| Language | Kotlin 2.0.21, JVM 17 |
| App framework | Android Gradle Plugin 8.7.3, AndroidX AppCompat / Material 3 |
| UI | **XML layouts + Views**. No Jetpack Compose. |
| Catalog UI | **Android `WebView`** loads the remote catalog site. Native chrome (URL bar, tabs, dialogs) sits around it. |
| CSS | Not a web bundle. Catalog restyle is injected JS/CSS in `app/src/main/assets/tv-shell.js` (Fire TV) and `phone-chrome.js` (phone header hide). Native look is Android XML drawables + `values/colors.xml`. |
| Bundler | **Gradle** only. No Webpack / Vite / Metro. |
| Playback | AndroidX Media3 ExoPlayer 1.4.1 (`exoplayer` + `exoplayer-hls` + OkHttp data source) |
| HTTP | OkHttp 4.12, Gson |
| License API | `https://bananaloop.mankis.de` |
| Target platforms | Phone, tablet (`sw600dp`), **Fire TV / Android TV** (`LEANBACK_LAUNCHER`, `values-television`, `layout-television`) |
| `minSdk` / `targetSdk` / `compileSdk` | 24 / 35 / 35 |
| ABIs | `armeabi-v7a`, `arm64-v8a` (no x86) |
| `applicationId` | `app.bananaloop` |

There is **no** PWA, **no** `index.html`, **no** web `manifest.json` in this repo.

---

## 3) How it runs on Fire TV

**Native Android APK** with Leanback launcher — not a browser, not a PWA.

Boot path:

1. `SplashActivity` (LEANBACK + LAUNCHER)
2. `Devices.isTelevision()` → `TvCatalogActivity` (landscape, 10-foot)
3. Catalog HTML in `WebView` (`CatalogController`)
4. Movies/series → `PlayerActivity` (Media3)
5. Live → `LivePlayerActivity` (bare HLS ExoPlayer)

| What | Path |
| --- | --- |
| Manifest | `app/src/main/AndroidManifest.xml` |
| Leanback feature | `android.software.leanback` required=`false` |
| Launcher | `MAIN` + `LAUNCHER` + `LEANBACK_LAUNCHER` on `SplashActivity` |
| Banner / logo | `@drawable/tv_banner` |
| App icon | `@mipmap/ic_launcher`, `@mipmap/ic_launcher_round` |
| TV catalog activity | `app/src/main/java/app/bananaloop/catalog/TvCatalogActivity.kt` |
| TV catalog layout | `app/src/main/res/layout/activity_tv_catalog.xml` |
| TV chrome | `app/src/main/res/layout/include_tv_chrome.xml` |
| Web index.html / PWA manifest | **None** |
| Injected catalog CSS/JS | `app/src/main/assets/tv-shell.js`, `tv-search-nav.js`, `catalog-locale.js`, `interceptor.js`, `live-guard.js`, `media-lock.js`, `trial-nag-guard.js` |

Phone/tablet use `CatalogActivity` instead of `TvCatalogActivity`. Detection: `app/src/main/java/app/bananaloop/DeviceKind.kt`.

Sideload: install the debug APK with `adb install -r BananaLoop-1.0.88-debug.apk` (see README). Apps & Games → My Apps → Banana Loop.

---

## 4) Screens and reusable UI

### Activities (full screens)

| Screen | Devices | Layout | Controller |
| --- | --- | --- | --- |
| Splash (yellow bounce) | all | `layout/activity_splash.xml`, TV: `layout-television/activity_splash.xml` | `splash/SplashActivity.kt` |
| Catalog + chrome | phone | `layout/activity_catalog.xml` | `catalog/CatalogActivity.kt` + `CatalogController.kt` |
| Catalog two-pane | tablet | `layout-sw600dp/activity_catalog.xml` | same |
| Catalog 10-foot | Fire TV | `layout/activity_tv_catalog.xml` | `catalog/TvCatalogActivity.kt` + `CatalogController.kt` |
| Movie/series player | phone | `layout/activity_player.xml` | `player/PlayerActivity.kt` |
| Movie/series player | Fire TV | `layout-television/activity_player.xml` | same |
| Live player | all | `layout/activity_live_player.xml` | `player/LivePlayerActivity.kt` |

### Catalog WebView “pages” (not native)

Remote site inside the WebView. Native tabs only navigate these paths: `/`, `/movies`, `/series`, `/live`, `/search`. Not `/einstellungen` or `/manager` (those are Electron shell routes and 404 on the catalog host).

### Overlays / dialogs (native)

| UI | Layout | Kotlin |
| --- | --- | --- |
| Onboarding (first source) | `layout/include_onboarding.xml`, TV: `layout-television/include_onboarding.xml` | `CatalogController` |
| Search field | `layout/include_search.xml` | `CatalogController.toggleSearch` |
| Phone shelves (Empfehlen / Quellen / Einstellungen) | `layout/include_shelf.xml` | `CatalogController` |
| Overflow (…) | `layout/dialog_overflow.xml` | `catalog/OverflowMenu.kt` |
| Einstellungen (TV shell) | `layout/dialog_settings.xml` | `catalog/SettingsDialog.kt` |
| Quellen-Manager (TV shell) | built in code | `catalog/SourcesDialog.kt` |
| Premium / trial ended | `layout/dialog_paywall.xml`, TV: `layout-television/dialog_paywall.xml` | `license/PaywallDialog.kt` |
| Konto | `layout/dialog_account.xml` | `license/AccountDialog.kt` |
| Bonus / referral | `layout/dialog_bonus.xml` | `license/BonusDialog.kt` |
| Player Quellen list | `layout/item_source.xml`, TV: `layout-television/item_source.xml` | `player/SourceAdapter.kt` |
| Loader | `layout/include_banana_loader.xml` | `CatalogController` |

### Navigation chrome

| Block | Layout |
| --- | --- |
| Phone URL bar | `layout/include_urlbar.xml` |
| Phone top tabs (Suche / Start / Filme / Serien / Live) | `layout/include_nav.xml` |
| Phone bottom bar (Start / Empfehlen / Quellen / Einstellungen) | `layout/include_nav_phone.xml` |
| Fire TV Windows-like source bar + tabs | `layout/include_tv_chrome.xml` |
| Wordmark (tablet rail) | `layout/include_wordmark.xml` |

### Reusable building blocks (drawables / styles)

Buttons and chrome:

- `drawable/play_button.xml`, `play_button_round.xml`, `play_circle.xml`
- `drawable/btn_ghost.xml`, `btn_remove.xml`
- `drawable/onboarding_go.xml` (yellow CTA + white focus ring)
- `drawable/chrome_nav_btn.xml`, `chrome_gift_btn.xml`
- `drawable/nav_item_bg.xml`
- Style `Widget.BananaLoop.NavItem`, `Widget.BananaLoop.TvTab` in `values/themes.xml`

Cards / panels / fields:

- `drawable/panel_bg.xml` — dialog/sheet surface
- `drawable/best_offer_card.xml`, `plan_card.xml`
- `drawable/surface_field.xml`, `url_field.xml`
- `drawable/quellen_modal.xml`, `quellen_capsule.xml`
- `drawable/source_row.xml`
- `drawable/chip_profile.xml`, `chip_profile_win.xml`, `chip_premium.xml`, `chip_ghost.xml`
- `drawable/loop_capsule.xml`, `loader_capsule.xml`
- `drawable/sheet_handle.xml`

Lists:

- Bookmark/source rows: built in `CatalogController.renderBookmarks` / `SourcesDialog`
- Player sources: `RecyclerView` + `item_source.xml` + `SourceAdapter.kt`
- Catalog rows/posters: **website DOM** inside WebView (not native RecyclerView)

Focus rings (see §5):

- `drawable/focus_ring.xml`
- `drawable/tv_tab_bg.xml`
- `drawable/player_focus.xml`
- `drawable/onboarding_go.xml`
- `drawable/chrome_nav_btn.xml`

Tokens: `values/colors.xml`, `values/dimens.xml`, TV overrides in `values-television/`.

---

## 5) Keyboard / D-Pad focus

**Yes — visible focus exists** on Fire TV. Phone is touch-first (same Views, no D-Pad routing).

### Visible focus drawables

| File | What you see |
| --- | --- |
| `res/drawable/focus_ring.xml` | Gold `#FFC93C` 3dp stroke on focused fields/buttons |
| `res/drawable/tv_tab_bg.xml` | Red `#E10600` underline + light fill on focused TV tabs |
| `res/drawable/onboarding_go.xml` | White ring + scale on “Hinzufügen & loslegen” |
| `res/drawable/chrome_nav_btn.xml` | Gold stroke on Back / Forward / … / brand |
| `res/drawable/player_focus.xml` | Cool highlight on Fire TV player controls |
| `res/drawable/chrome_gift_btn.xml` | Focus state on Bonus |

Color token: `focus_ring` / `banana_yellow` in `values/colors.xml`. Stroke width: `focus_stroke` in `values/dimens.xml` (TV: `values-television/dimens.xml`).

### Native D-Pad implementation

| File | Role |
| --- | --- |
| `catalog/TvCatalogActivity.kt` | `dispatchKeyEvent`: Up / Down / OK / Enter / A |
| `catalog/CatalogController.kt` | `onTvDpadUp`, `onTvDpadDown`, `onTvChromeOk`, `onOnboardingDpadDown`, `onOnboardingOk`, `offerCatalogOk`, `focusTvTab`, `armTvConfirm`, `armTvGoldFocus`, `setWebTakesKeys` |
| `catalog/TvDialog.kt` | Makes dialog Buttons/EditTexts focusable; OK → `performClick()` |
| `license/PaywallDialog.kt` | `TvDialog.prepare` + `nextFocus*` + initial `requestFocus()` on email |
| `license/AccountDialog.kt`, `BonusDialog.kt` | same helper |
| `catalog/OverflowMenu.kt`, `SettingsDialog.kt`, `SourcesDialog.kt` | same helper |
| `player/PlayerActivity.kt` | D-Pad on skip/play/seek; TV chrome show/hide |
| `player/LivePlayerActivity.kt` | Surface not focusable; OK shows chrome / Back |

XML `android:focusable="true"` and `android:nextFocusUp/Down/Left/Right` are set on TV chrome (`include_tv_chrome.xml`) and TV paywall (`layout-television/dialog_paywall.xml`).

### WebView catalog focus (injected)

| File | Role |
| --- | --- |
| `assets/tv-shell.js` | `outline: 3px solid #E10600` on focused posters, rows, buttons |
| `assets/tv-search-nav.js` | D-Pad through search results, series episodes, source rows; `__bananaAtTop` so header only opens from the topmost item |

`offerCatalogOk()` calls `__bananaTvActivateFocused` / `__bananaTvClickFocused` then always falls through to the WebView.

---

## 6) Assets

All icons/logos used at runtime are Android resources (vector XML + PNG banners/launchers). There is no separate image CDN.

See `/opt/cursor/artifacts/banana-loop-apk-assets.zip` (`ASSETS.txt` inside). Vectors live in `app/src/main/res/drawable/*.xml`. Raster banners: `drawable*/tv_banner.png`. Launchers: `mipmap-*/ic_launcher*.png`.

Master marketing icon (not packaged in the APK): `bl-icons/AppIcon-1024.png`.

---

## 7) Test URL / runnable build

| | |
| --- | --- |
| Latest APK | **1.0.88** debug |
| Direct | https://github.com/raymoe34/banana-loop-apk/raw/main/BananaLoop-1.0.88-debug.apk |
| Repo (binaries only) | https://github.com/raymoe34/banana-loop-apk |
| SHA-256 | `dc07b70d9f60e53280c7dc4fca6fc3b612debe5b904dd4875e88f4d6e7d8e1cb` |
| Size | 14 623 796 bytes |
| `versionCode` | 89 |

No GitHub Release. Sideload with adb or Downloader on the Fire TV. Build from source: README.
