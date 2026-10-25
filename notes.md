# Ghi chú phân tích APK — Smart Launcher

> **Phạm vi:** chỉ đọc và ghi chú. **Không sửa** bất kỳ file nào trong APK đã decode.

## 0. Ghi chú về file đầu vào (quan trọng)

Yêu cầu ban đầu là giải nén `SmartLauncher_latest.apk`, nhưng **file này không tồn tại** trong thư mục làm việc. Thư mục chỉ có:

| File | Kích thước | Vai trò |
|---|---|---|
| `base.apk` | 23.9 MB | APK base (đầy đủ code + resources) ← **đã dùng làm đầu vào** |
| `split_config.arm64_v8a.apk` | 49 KB | Split ABI (native libs) |
| `split_config.vi.apk` | 127 KB | Split ngôn ngữ tiếng Việt |
| `split_config.xxhdpi.apk` | 70 KB | Split density màn hình |
| `SmartLauncher-splits.zip` | 14.4 MB | Gói zip chứa các split |
| `SmartLauncher_decoded/` | 225 MB | Bản decode cũ (từ `base.apk`) |

Vì `base.apk` là APK base duy nhất và bản decode cũ (`SmartLauncher_decoded/apktool.yml`) xác nhận cũng chính là Smart Launcher 6.6 build 023, nên `base.apk` được dùng thay thế.

Lệnh đã chạy (apktool cục bộ `apktool.jar`, vì `apktool` không có trong PATH):

```bash
java -jar apktool.jar d base.apk -o smartlauncher_decoded -f
```

Kết quả: exit code 0, có cảnh báo `Unresolved resource reference ... 0x7f...` (bình thường với APK dùng split/App Bundle).

**Lưu ý về split:** đây là APK từ Android App Bundle. `lib/` (native) nằm trong `split_config.arm64_v8a.apk`, chuỗi dịch (kể cả tiếng Việt) nằm trong `split_config.vi.apk`, ảnh mật độ cao nằm trong `split_config.xxhdpi.apk`. Bản decode chỉ từ `base.apk` nên **thiếu** các phần đó — khi rebuild phải nhớ điều này.

---

## Mục lục

1. [Thông tin chung](#1-thông-tin-chung)
2. [Version & SDK](#2-version--sdk)
3. [Danh sách permission](#3-danh-sách-permission)
4. [Activity chính (intent-filter LAUNCHER)](#4-activity-chính-intent-filter-launcher)
5. [Service](#5-service)
6. [Receiver](#6-receiver)
7. [Provider](#7-provider)
8. [Cấu trúc smali](#8-cấu-trúc-smali)
9. [Cấu trúc resource](#9-cấu-trúc-resource)
10. [Nhận xét sơ bộ: obfuscation](#10-nhận-xét-sơ-bộ-obfuscation)
11. [Nhận xét sơ bộ: quảng cáo / billing / tracking](#11-nhận-xét-sơ-bộ-quảng-cáo--billing--tracking)
12. [Các điểm đáng chú ý khác](#12-các-điểm-đáng-chú-ý-khác)
13. [Ghi chú cho bước mod sau](#13-ghi-chú-cho-bước-mod-sau)

---

## 1. Thông tin chung

| Thuộc tính | Giá trị |
|---|---|
| **Package name** | `ginlemon.flowerfree` |
| Tên hiển thị | Smart Launcher (`res/values/strings.xml` → `app_label`) |
| Application class | `ginlemon.flowerfree.SLApp` |
| Package prefix của code | `ginlemon.flower.*`, `ginlemon.launcher.*`, `ginlemon.library.*`, `ginlemon.notifications.*` |
| Theme mặc định | `@style/StaticDayNightStyle` |
| Icon | `@drawable/ic_launcher` (round icon dùng cùng drawable) |
| Thuộc tính `<application>` đáng chú ý | `android:persistent="true"`, `android:excludeFromRecents="true"`, `android:allowBackup="false"`, `android:requestLegacyExternalStorage="true"`, `android:requiredForAllUsers="true"`, `android:appComponentFactory="androidx.core.app.CoreComponentFactory"` |
| Số dex | 2 (`classes.dex` 11.3 MB, `classes2.dex` 5.2 MB) |
| Chữ ký | Không có chữ ký v1 (không có `.RSA`/`.SF` trong `META-INF`) → chỉ ký v2/v3. Có `original/stamp-cert-sha256` (Play Store source stamp) |

Nguồn: `smartlauncher_decoded/AndroidManifest.xml`.

---

## 2. Version & SDK

Từ `AndroidManifest.xml` và `apktool.yml`:

| Thuộc tính | Giá trị |
|---|---|
| **versionName** | `6.6 build 023` |
| **versionCode** | `660230` |
| minSdkVersion | `29` (Android 10) |
| targetSdkVersion | `36` (Android 16) |
| compileSdkVersion | `36` |
| apktool version dùng để decode | `3.0.3` |
| `resourcesInfo.packageId` | `127` (0x7f) |
| `installLocation` | `internalOnly` |
| Split types yêu cầu | `base__abi,base__density` |

Metadata Play Store trong manifest:

- `com.android.vending.splits.required = true`
- `com.android.vending.splits = @xml/splits0`
- `com.android.vending.derived.apk.id = 3`
- `com.android.stamp.source = https://play.google.com/store`
- `com.android.stamp.type = STAMP_TYPE_DISTRIBUTION_APK`

---

## 3. Danh sách permission

Tổng: **34** `<uses-permission>` + **2** `<permission>` tự định nghĩa + `<queries>` cho billing.

### 3.1 Permission hệ thống / bên thứ ba (34)

| # | Permission | Nhóm chức năng |
|---|---|---|
| 1 | `com.android.vending.BILLING` | Mua trong app (Play Billing) |
| 2 | `android.permission.FOREGROUND_SERVICE` | Foreground service |
| 3 | `android.permission.FOREGROUND_SERVICE_MEDIA_PROJECTION` | Chụp màn hình (wallpaper screenshot) |
| 4 | `android.permission.EXPAND_STATUS_BAR` | Mở rộng notification shade |
| 5 | `android.permission.ACCESS_HIDDEN_PROFILES` | Truy cập profile ẩn (work profile) |
| 6 | `android.permission.CALL_PHONE` | Gọi điện trực tiếp |
| 7 | `android.permission.READ_CALENDAR` | Đọc lịch (widget lịch) |
| 8 | `android.permission.WRITE_EXTERNAL_STORAGE` | Ghi bộ nhớ ngoài (legacy) |
| 9 | `android.permission.READ_MEDIA_AUDIO` | Đọc media audio |
| 10 | `android.permission.READ_MEDIA_VIDEO` | Đọc media video |
| 11 | `android.permission.READ_MEDIA_IMAGES` | Đọc media ảnh |
| 12 | `android.permission.VIBRATE` | Rung |
| 13 | `android.permission.SET_WALLPAPER` | Đặt hình nền |
| 14 | `android.permission.SET_WALLPAPER_HINTS` | Gợi ý kích thước hình nền |
| 15 | `android.permission.INTERNET` | Mạng |
| 16 | `android.permission.ACCESS_NETWORK_STATE` | Trạng thái mạng |
| 17 | `android.permission.READ_CONTACTS` | Đọc danh bạ |
| 18 | `android.permission.WRITE_SETTINGS` | Sửa cài đặt hệ thống |
| 19 | `android.permission.QUERY_ALL_PACKAGES` | Liệt kê toàn bộ app (bắt buộc với launcher) |
| 20 | `android.permission.USE_FINGERPRINT` | Vân tay (legacy) |
| 21 | `android.permission.USE_BIOMETRIC` | Sinh trắc học |
| 22 | `android.permission.REQUEST_DELETE_PACKAGES` | Yêu cầu gỡ app |
| 23 | `android.permission.READ_EXTERNAL_STORAGE` | Đọc bộ nhớ ngoài (legacy) |
| 24 | `android.permission.ACCESS_FINE_LOCATION` | Vị trí chính xác (thời tiết) |
| 25 | `android.permission.ACCESS_COARSE_LOCATION` | Vị trí tương đối |
| 26 | `android.permission.BIND_NOTIFICATION_LISTENER_SERVICE` | Đọc notification |
| 27 | `android.permission.POST_NOTIFICATIONS` | Gửi notification |
| 28 | `android.permission.WAKE_LOCK` | Giữ CPU thức |
| 29 | `com.google.android.c2dm.permission.RECEIVE` | Nhận FCM push |
| 30 | `android.permission.ACCESS_ADSERVICES_ATTRIBUTION` | Privacy Sandbox attribution |
| 31 | `android.permission.ACCESS_ADSERVICES_AD_ID` | Privacy Sandbox ad ID |
| 32 | `com.google.android.finsky.permission.BIND_GET_INSTALL_REFERRER_SERVICE` | Install referrer (Play) |
| 33 | `android.permission.RECEIVE_BOOT_COMPLETED` | Chạy sau khi khởi động |
| 34 | `ginlemon.flowerfree.DYNAMIC_RECEIVER_NOT_EXPORTED_PERMISSION` | Nội bộ app (signature) |

### 3.2 Permission tự định nghĩa (2)

| Permission | protectionLevel | Dùng cho |
|---|---|---|
| `ginlemon.flowerfree.permission.READ_FEED_OPML` | `signature` | Bảo vệ provider `ginlemon.flowerfree.feedopml` |
| `ginlemon.flowerfree.DYNAMIC_RECEIVER_NOT_EXPORTED_PERMISSION` | `signature` | Bảo vệ dynamic receiver nội bộ |

### 3.3 `<queries>` (bắt buộc cho billing trên Android 11+)

- `com.android.vending.billing.InAppBillingService.BIND`
- `com.google.android.apps.play.billingtestcompanion.BillingOverrideService.BIND`

### 3.4 `<uses-feature>` / `<uses-library>`

- `uses-feature` (đều `required="false"`): `touchscreen`, `faketouch`, `telephony`, `screen.portrait`, `ram.low`
- `uses-library` (đều `required="false"`): `org.apache.http.legacy`, `androidx.window.extensions`, `androidx.window.sidecar`, `android.ext.adservices`

---

## 4. Activity chính (intent-filter LAUNCHER)

Tổng số activity khai báo trong manifest: **73**. Nhưng chỉ **duy nhất 1** activity có `category.LAUNCHER`:

| Activity | Đặc điểm |
|---|---|
| **`ginlemon.flower.HomeScreen`** | `exported="true"`, `launchMode="singleTask"`, `label="@string/app_label"`, `theme="@style/Theme.Acrylic.Transparent"`, `taskAffinity=""`, `stateNotNeeded="true"`, `resumeWhilePausing="true"`, `resizeableActivity="true"` |

Intent-filter của `HomeScreen` (2 filter):

```
# Filter 1 — home + launcher
action    android.intent.action.MAIN
category  android.intent.category.HOME          ← thay thế launcher mặc định
category  android.intent.category.DEFAULT
category  android.intent.category.LAUNCHER      ← icon ngoài màn hình chính
category  android.intent.category.MONKEY
category  android.intent.category.LAUNCHER_APP

# Filter 2 — action riêng của app
action    ginlemon.flower.SL_ACTION
category  android.intent.category.DEFAULT
```

Kèm `meta-data android.app.shortcuts = @xml/shortcuts`.

### Entry point phụ (không có LAUNCHER nhưng đáng chú ý)

| Activity | Intent-filter | Ghi chú |
|---|---|---|
| `ginlemon.flower.preferences.PrefSectionActivity` | `MAIN` (không có LAUNCHER) | `exported="true"`, label "Preferences" — mở được từ ngoài |
| `ginlemon.flower.navigation.AppLinkRouterActivity` | `VIEW` http/https `applink.smartlauncher.net` (`/pw`, `/pw/ciliegia`), `autoVerify="true"` | Deep link / App Link |
| `ginlemon.flower.preferences.activities.ExportedActivities$RestoreBackup` | `VIEW` file/content `*.slbk` | Khôi phục backup, `exported="true"` |
| `ginlemon.flower.preferences.activities.ExportedActivities$FakeHome` | `MAIN` + `HOME` | `android:enabled="false"` — dùng để reset default home |
| `ginlemon.flower.preferences.activities.ExportedActivities$SetHomeActivity` | không filter | `exported="true"`, mở hộp thoại chọn launcher |
| `ginlemon.flower.preferences.activities.ExportedActivities$LaunchHsGallery` | không filter | `exported="true"` |
| `ginlemon.flower.preferences.activities.ExportedActivities$LaunchFAQ` | không filter | `exported="true"` |
| `ginlemon.flower.preferences.activities.ExportedActivities$FakeActivity` | `MAIN` + action `ginlemon.smartlauncher.SETTINGS_PROVIDER` | `exported="true"` |
| `ginlemon.flower.pinrequests.PinRequestHandlerActivity` | `CONFIRM_PIN_SHORTCUT`, `CONFIRM_PIN_APPWIDGET` | Ghim shortcut/widget |
| `ginlemon.flower.wallpaperCropUi.WallpaperCropActivity` | `CROP_AND_SET_WALLPAPER`, `ATTACH_DATA` (image/*) | Crop wallpaper |
| `ginlemon.flower.applyIconPack.ApplyIconPackActivity` | action `ginlemon.smartlauncher.setGSLTHEME` | Áp icon pack từ app khác |
| `ginlemon.flower.shortcutPicker.create.SLCreateShortcutActivity` | `CREATE_SHORTCUT` | Tạo shortcut |
| `ginlemon.flower.premium.paywall.classicpaywall.SingularProductPaywallActivity` | — | `exported="true"` (paywall) |
| `ginlemon.flower.premium.paywall.experimental.PaywallExperimentalActivity` | — | `exported="true"` (paywall) |
| `ginlemon.flower.premium.advantages.PremiumFeaturesActivity` | — | `exported="true"` |
| `ginlemon.flower.library.SLPatternTestActivity` | — | `exported="true"` — **activity test/debug lộ ra ngoài** |
| `ginlemon.flower.graphic.UIdemoActivity` | — | `exported="true"` — demo UI |
| `ginlemon.flower.appWidget.AppWidgetEncapsulationActivity` | — | `exported="false"` |
| `ginlemon.flower.preferences.prefMenu.PrefMenuActivity` | `APPLICATION_PREFERENCES` | Menu cài đặt |
| `ginlemon.flower.missions.MissionCompletedActivity` | action `ginlemon.smartlauncher.MISSION_COMPLETED` | `exported="true"` |
| `ginlemon.flower.PreventModificationsActivity` | actions `ginlemon.smartlauncher.BUBBLESTYLE`, `...CLOCKFONTS` | `exported="false"` |

### Activity của thư viện (khai báo trong manifest)

`androidx.compose.ui.tooling.PreviewActivity` (`exported="true"`), `com.android.billingclient.api.ProxyBillingActivity`, `...ProxyBillingActivityV2`, `com.google.android.gms.common.api.GoogleApiActivity`, `com.google.android.play.core.common.PlayCoreDialogWrapperActivity`.

### Nhóm activity theo tính năng (trong code app)

`preferences/*` (cài đặt), `premium/paywall/*` (4 paywall), `feedrss/*` (RSS reader), `widgets/*` (widget nội bộ), `pickers/*` (chọn icon/widget), `wallpaper*` (hình nền), `missions/*`, `shareYourHomescreen/*`, `onboarding/*`, `about/*`, `recovery/*`, `appWidget/*`.

---

## 5. Service

Tổng: **14**

| Service | exported | Ghi chú |
|---|---|---|
| `ginlemon.flower.wallpaperScreenshot.WallpaperFrameCaptureService` | false | `foregroundServiceType="mediaProjection"` — chụp màn hình |
| `ginlemon.flower.library.PanelActionService` | false | **AccessibilityService**, permission `BIND_ACCESSIBILITY_SERVICE`, meta-data `@xml/descr_accessibility_service` |
| `ginlemon.flower.core.cloudMessaging.SlFirebaseMessagingService` | false | Nhận FCM của app |
| `ginlemon.notifications.listener.NotificationListener` | false | **NotificationListenerService**, permission `BIND_NOTIFICATION_LISTENER_SERVICE` |
| `com.google.firebase.components.ComponentDiscoveryService` | false | Firebase (16 registrar: crashlytics, sessions, messaging, remoteconfig, analytics-connector, dynamiclinks, installations, abt, datatransport) |
| `com.google.firebase.sessions.SessionLifecycleService` | false | `enabled="false"` |
| `com.google.firebase.messaging.FirebaseMessagingService` | false | FCM base service |
| `com.google.android.gms.measurement.AppMeasurementService` | false | Google Analytics |
| `com.google.android.gms.measurement.AppMeasurementJobService` | false | Analytics job |
| `androidx.work.impl.background.systemjob.SystemJobService` | **true** | WorkManager, permission `BIND_JOB_SERVICE` |
| `androidx.work.impl.foreground.SystemForegroundService` | false | WorkManager foreground |
| `androidx.room.MultiInstanceInvalidationService` | false | Room DB |
| `com.google.android.datatransport.runtime.backends.TransportBackendDiscovery` | false | DataTransport (analytics transport) |
| `com.google.android.datatransport.runtime.scheduling.jobscheduling.JobInfoSchedulerService` | false | DataTransport job |

---

## 6. Receiver

Tổng: **8**

| Receiver | exported | Ghi chú |
|---|---|---|
| `ginlemon.flower.feedrss.presentation.inAppRead.ReadingModeReceiver` | false | action `ginlemon.flower.action.OPEN_READING_MODE` |
| `com.google.firebase.iid.FirebaseInstanceIdReceiver` | **true** | permission `com.google.android.c2dm.permission.SEND`, action `c2dm.intent.RECEIVE` |
| `com.google.android.gms.measurement.AppMeasurementReceiver` | false | Analytics |
| `androidx.work.impl.utils.ForceStopRunnable$BroadcastReceiver` | false | WorkManager |
| `androidx.work.impl.background.systemalarm.RescheduleReceiver` | false | `enabled="false"`, action `BOOT_COMPLETED` |
| `androidx.work.impl.diagnostics.DiagnosticsReceiver` | **true** | permission `DUMP` |
| `com.google.android.datatransport.runtime.scheduling.jobscheduling.AlarmManagerSchedulerBroadcastReceiver` | false | DataTransport |
| `androidx.profileinstaller.ProfileInstallReceiver` | **true** | permission `DUMP`, 4 action profileinstaller |

Ngoài manifest còn receiver khai báo động trong code: `ginlemon.flower.SLEventsReceiver`, `ginlemon.flower.UserProfileRepo$broadcastReceiver$1`.

---

## 7. Provider

Tổng: **9**

| Provider | Authority | exported | Ghi chú |
|---|---|---|---|
| `ginlemon.flowerfree.feedopml.SmartLauncherFeedOpmlProvider` | `ginlemon.flowerfree.feedopml` | **true** | `readPermission=ginlemon.flowerfree.permission.READ_FEED_OPML` (signature) — import/export OPML feed |
| `ginlemon.flower.provider.KustomProvider` | `ginlemon.flowerfree.kustomprovider` | **true** | Cầu nối Kustom (KWGT/KLWP) |
| `ginlemon.flower.preferences.SettingsProvider` | `ginlemon.flowerfree.settingsprovider` | **true** | Chia sẻ cài đặt cho app/widget khác |
| `ginlemon.flower.billing.LicenseStatusProvider` | `ginlemon.flowerfree.haspro` | **true** | **Công bố trạng thái premium ra ngoài** |
| `androidx.core.content.FileProvider` | `ginlemon.flowerfree.provider` | false | `grantUriPermissions=true`, meta-data `@xml/provider_paths` |
| `ginlemon.flower.wallpaper.SetWallpaperProvider` | `ginlemon.flowerfree.wallpaperprovider` | **true** | `writePermission=android.permission.SET_WALLPAPER` |
| `com.google.firebase.provider.FirebaseInitProvider` | `ginlemon.flowerfree.firebaseinitprovider` | false | `initOrder="100"`, `directBootAware="true"` |
| `leakcanary.internal.MainProcessAppWatcherInstaller` | `ginlemon.flowerfree.leakcanary-installer` | false | **LeakCanary còn trong bản release** (`enabled=@bool/leak_canary_watcher_auto_install`) |
| `com.squareup.picasso.PicassoProvider` | `ginlemon.flowerfree.com.squareup.picasso` | false | Picasso (tải ảnh) |

Ngoài ra có activity lồng trong provider: `ginlemon.flower.provider.KustomProvider$AutoFinishTransparentActivity`.

---

## 8. Cấu trúc smali

### 8.1 Tổng quan

| Thư mục | Số file `.smali` |
|---|---|
| `smali/` (classes.dex) | **12,340** |
| `smali_classes2/` (classes2.dex) | **5,890** |
| **Tổng** | **18,230** |

Không có `smali_classes3/` trở đi. **Không có thư mục `lib/`** (native libs nằm ở `split_config.arm64_v8a.apk`).

Đáng chú ý: **12,243** class nằm ngay ở package mặc định (root) — 9,192 file trong `smali/` và 3,051 file trong `smali_classes2/` — tên vô nghĩa kiểu `a0.smali`, `a00.smali`, `iqb.smali`, `lwb.smali`. Đây là kết quả R8 rename (xem mục 10).

### 8.2 Package cấp 1 — `smali/`

| Package | Số class |
|---|---|
| `com/` | 2,032 |
| `androidx/` | 792 |
| `ginlemon/` | 297 |
| `android/` | 19 |
| `dagger/` | 8 |
| *(root, không package)* | 9,192 |

### 8.3 Package cấp 1 — `smali_classes2/`

| Package | Số class |
|---|---|
| `org/` | 1,746 |
| `kotlinx/` | 675 |
| `ginlemon/` | 204 |
| `tech/` | 143 |
| `javax/` | 59 |
| `me/` | 7 |
| `kotlin/` | 2 |
| `leakcanary/` | 2 |
| `okhttp3/` | 1 |
| *(root, không package)* | 3,051 |

### 8.4 Package con của code app (`ginlemon`, tổng 501 class)

Các package lớn (tính cả 2 thư mục smali):

| Package | Số class |
|---|---|
| `ginlemon.flower.widgets` | 67 |
| `ginlemon.flower.preferences` | 42 |
| `ginlemon.flower.core` | 21 |
| `ginlemon.flower.pickers` | 20 |
| `ginlemon.flower.fontLoader` | 18 |
| `ginlemon.flower.premium` | 17 |
| `ginlemon.flower.navigation` | 16 |
| `ginlemon.flower.feedrss` | 16 |
| `ginlemon.flower.panels` | 15 |
| `ginlemon.flower.feedRssProvider` | 15 |
| `ginlemon.flower.appWidget` | 15 |
| `ginlemon.flower.library` | 12 |
| `ginlemon.flower.slcompanionbase` | 11 |
| `ginlemon.flower.settings` / `missions` | 6 / 6 |
| `ginlemon.flower.billing` / `about` | 5 / 5 |
| `ginlemon.flower.viewWidget`, `widgetUtils`, `whatsNew`, `wallpaperCropUi`, `shortcutPicker`, `shell`, `shareYourHomescreen`, `searchPanel`, `wallpaperScreenshot`, `wallpaperPicker`, `wallpaper`, `homePanel`, `supergrid` | 2–5 mỗi package |

Nhóm package khác của hãng: `ginlemon.iconretrieving`, `ginlemon.launcher`, `ginlemon.library`, `ginlemon.notifications.listener`, `ginlemon.weatherproviders`, `ginlemon.flowerfree.feedopml`.

Các class gốc đọc được tên: `HomeScreen`, `Hilt_HomeScreen`, `App`, `SLApp`, `DndLayer`, `LockerView`, `FloatingSurfaceView`, `PreventModificationsActivity`, `Hilt_PreventModificationsActivity`, `SLEventsReceiver`, `ColorTestActivity`, `DummyActivity`...

### 8.5 Thư viện bên thứ ba phát hiện trong smali

| Thư viện | Bằng chứng |
|---|---|
| AndroidX (appcompat, compose, work, room, biometric, browser, cardview, constraintlayout, window, profileinstaller, core) | `smali/androidx/*` (792 class) |
| Kotlin + coroutines | `kotlin/`, `kotlinx/coroutines` (675 class) |
| Hilt / Dagger (DI) | `dagger/` (8), class `Hilt_HomeScreen`, `Hilt_PreventModificationsActivity` |
| Google Play Services (measurement/analytics, location, cloud-messaging, basement, base) | `com/google/android/gms/*`, `unknown/play-services-*.properties` |
| Firebase (analytics, messaging, crashlytics, remoteconfig, dynamiclinks, installations, sessions, abt, datatransport) | `com/google/firebase/*`, `unknown/firebase-*.properties` |
| Google Play Billing 8.3.0 | `com/android/billingclient/api/*`, meta-data `com.google.android.play.billingclient.version=8.3.0` |
| Google Play Core | `com.google.android.play.core.*`, `PlayCoreDialogWrapperActivity` |
| Picasso | 128 file, `com.squareup.picasso.PicassoProvider` |
| okhttp3 / Okio | `okhttp3/`, `com/squareup/okio` |
| jsoup (parse HTML cho RSS reader) | `org/jsoup/*` |
| LeakCanary | `leakcanary/internal/*`, provider `leakcanary-installer` |
| androidx.work (WorkManager) | service/receiver như mục 5–6 |
| Room | `androidx/room/MultiInstanceInvalidationService` |
| `javax.measure` / `tech.units.indriya` | đơn vị đo (unit converter của widget) |
| `me.relex.circleindicator` | indicator cho viewpager |
| `org.jetbrains.annotations`, `org.checkerframework`, `org.jspecify` | annotation compile-time |
| `org.jf.dexlib2` | dexlib2 (xử lý dex runtime) |

### 8.6 `unknown/` — 34 entry

Không phải thư mục resource; đây là metadata META-INF bị apktool đẩy sang: các file `*.properties` phiên bản thư viện (`billing.properties`, `firebase-analytics.properties`, `play-services-ads-identifier.properties`, `play-services-location.properties`...), `client_analytics.proto`, `messaging_event.proto`, `kotlin-tooling-metadata.json`, `DebugProbesKt.bin`. Dùng để xác định chính xác dependency — hữu ích khi đối chiếu thư viện.

---

## 9. Cấu trúc resource

### 9.1 Các thư mục quan trọng

| Thư mục | Số file | Ghi chú |
|---|---|---|
| `res/layout/` | **170** | Layout chính (`activity_*`, `fragment_*`, `item_*`, widget...) |
| `res/drawable/` | 540 | Vector + shape + selector |
| `res/drawable-xxhdpi/` | 154 | Bitmap mật độ cao (phần lớn bitmap thật nằm ở split `xxhdpi`) |
| `res/drawable-anydpi/` | 86 | Adaptive icon, vector anydpi |
| `res/drawable-nodpi/` | 16 | Ảnh không scale |
| `res/drawable-{xhdpi,xxxhdpi,v26,watch,anydpi-v26}` | 2 / 2 / 2 / 1 / 1 | |
| **Tổng drawable (mọi qualifier)** | **804** | |
| `res/color/` (+`color-v26`, `color-night`) | 80 (+1+1) | Color state list |
| `res/anim/` | 62 | Animation cũ |
| `res/animator/` | 24 | Object animator |
| `res/interpolator/` | 13 | |
| `res/font/` | 15 | Font nhúng |
| `res/xml/` (+`xml-v35`) | 9 (+1) | Xem 9.3 |
| `res/raw/` | 2 | |
| `res/layout-land/` | 4 | Landscape |
| `res/layout-sw600dp/`, `layout-v26/`, `layout-watch/` | 2 mỗi loại | Tablet / API 26 / Wear |

### 9.2 `res/values/` (13 file)

`arrays.xml`, `attrs.xml`, `bools.xml`, `colors.xml` (**171** color), `dimens.xml`, `drawables.xml`, `fonts.xml`, `ids.xml`, `integers.xml`, `plurals.xml`, `public.xml`, `strings.xml` (**1,502** string), `styles.xml`.

Có **27** thư mục `values-*` nhưng **không có thư mục locale** (không có `values-vi`, `values-fr`...). Lý do: chuỗi dịch nằm trong `split_config.vi.apk` (và các split ngôn ngữ khác không có trong thư mục này). Các `values-*` ở đây chỉ là qualifier kích thước/hướng/API: `values-night`, `values-land`, `values-sw600dp`, `values-w320dp-land`, `values-v25..v31`, `values-ldrtl`, `values-watch`...

### 9.3 `res/xml/`

`_generated_res_locale_config.xml` (locale config), `appfilter.xml` (ánh xạ icon pack), `backup_rules.xml`, `descr_accessibility_service.xml`, `m3_button_group_child_size_change.xml`, `m3_split_button_inner_corner_size_state_list.xml`, `provider_paths.xml` (FileProvider), `shortcuts.xml` (app shortcuts), `splits0.xml` (khai báo split của App Bundle).

### 9.4 Thư mục khác trong bản decode

| Thư mục | Kích thước | Nội dung |
|---|---|---|
| `assets/` | 896 KB | `PublicSuffixDatabase.list`, `database/ginlemon.flower.db_v1.db` (DB mẫu), `dexopt/baseline.prof(m)`, `drawable.xml`, `google_fonts_data.json`, `licenses.html`, `open_source_licenses.json`, `readability.js` (cho chế độ đọc RSS) |
| `original/` | 68 KB | `AndroidManifest.xml` gốc (binary) + `stamp-cert-sha256` |
| `unknown/` | 1.3 MB | Metadata META-INF (mục 8.6) |

---

## 10. Nhận xét sơ bộ: obfuscation

**Kết luận: CÓ obfuscation bằng R8, nhưng KHÔNG obfuscate toàn bộ — code chính của app vẫn đọc được tên.**

Bằng chứng:

1. **R8 đã chạy:** 18,060 / 18,230 file smali có dòng `.source "r8-map-id-<sha256>"` — đây là dấu hiệu đặc trưng của R8 (Android Gradle Plugin) có bật mapping ID. File mapping (`mapping.txt`) **không** có trong APK.
2. **Tên class bị rename hàng loạt:** 12,243 class nằm ở package mặc định với tên vô nghĩa 1–3 ký tự (`a0`, `a00`, `a0b`, `iqb`, `lwb`, `zzc`...). Trong `smali/`: 9,189/9,192 class root có tên ≤ 3 ký tự. Đây là các class thư viện (GMS, Kotlin, Firebase, Play Core...) và một phần code app đã bị R8 đổi tên.
3. **Nhưng entry point & code app vẫn rõ nghĩa:** toàn bộ package `ginlemon.flower.*` (501 class) giữ nguyên tên đọc được — `HomeScreen`, `PremiumFeaturesActivity`, `PaywallExperimentalActivity`, `LicenseStatusProvider`, `PurchaseReEngagementWorker`, `PanelActionService`, `WidgetPickerActivity`... Tên method/field trong các class này cũng đọc được.
   → Đây là R8 với **keep rules** cho: Android component khai báo trong manifest, Hilt/Dagger generated, Room, Kotlin metadata. Không phải obfuscation "che giấu" cố ý.
4. **Không có packer / anti-tamper / string encryption:**
   - Không tìm thấy: `com/pairip` (PairIP của Google), `com/tencent`, `libjiagu`, `qihoo`, `stringfog`, `com/guard`, `SecShell` (0 hit).
   - Vẫn còn 2 dex bình thường, `AndroidManifest.xml` đọc trực tiếp được, không có class loader động giải mã dex.
   - → APK **không bị pack**, có thể `apktool b` lại và chỉnh sửa như bình thường.
5. **Tên hàm/biến trong code app:** dễ đọc (không bị đổi thành `a()`, `b()`); chỉ các lambda/anonymous class có tên kiểu `UserProfileRepo$broadcastReceiver$1` (bình thường với Kotlin).

Hệ quả cho việc mod: có thể đọc trực tiếp logic premium/billing trong `ginlemon/flower/billing/`, `ginlemon/flower/premium/` mà không cần deobfuscate.

---

## 11. Nhận xét sơ bộ: quảng cáo / billing / tracking

### 11.1 Quảng cáo (ads) — **KHÔNG có SDK quảng cáo**

| Kiểm tra | Kết quả |
|---|---|
| AppLovin (`applovin`) | 0 hit |
| Unity Ads (`unity3d`) | 0 hit |
| ironSource | 0 hit |
| Facebook Audience Network (`facebook/ads`) | 0 hit |
| MoPub / Vungle / Chartboost | 0 hit |
| AdMob đầy đủ (`com.google.android.gms.ads.AdView`, `InterstitialAd`...) | **Không có** — chỉ có 5 file thuộc `gms/ads/identifier` |

Chi tiết `com.google.android.gms.ads` chỉ gồm `AdvertisingIdClient` + `ads_identifier/R` — tức **thư viện lấy Advertising ID** (`play-services-ads-identifier`), phục vụ đo lường/analytics chứ không hiển thị quảng cáo. Không có layout nào chứa `AdView`/banner.

Chuỗi `admob` xuất hiện ở: `smali/bqb.smali` (cột DB `apps.admob_app_id` — có vẻ là dữ liệu widget/app nào đó) và `gms/measurement` (log nội bộ của SDK Google Analytics).

→ **Mô hình kiếm tiền là trả phí (freemium/premium), không phải quảng cáo.**

### 11.2 Billing / thanh toán — **CÓ, đầy đủ**

| Thành phần | Chi tiết |
|---|---|
| Permission | `com.android.vending.BILLING` |
| SDK | Google Play Billing Library **8.3.0** (`com.google.android.play.billingclient.version=8.3.0`), 23 file `com/android/billingclient/api/*` |
| Activity của SDK | `ProxyBillingActivity`, `ProxyBillingActivityV2` |
| `<queries>` | `com.android.vending.billing.InAppBillingService.BIND` |
| Code riêng của app | package `ginlemon.flower.billing`: `LicenseStatusProvider`, `PurchaseReEngagementWorker`, `periodicPromo/*`, `seasonalPromo/*` |
| Provider công bố trạng thái | `ginlemon.flowerfree.haspro` (exported=true) |
| Paywall (4 màn) | `SingularProductPaywallActivity`, `MultiProductPaywallActivity`, `ComposePaywallActivity`, `PaywallExperimentalActivity` + `PremiumFeaturesActivity`, `LeavingPremiumActivity` |
| Thư viện | `billing`, `billing-ktx` (trong `unknown/`) |

### 11.3 Tracking / analytics — **CÓ (Firebase + Google Analytics), nhưng có cờ tắt**

| Thành phần | Trạng thái |
|---|---|
| Firebase Analytics + GMS Measurement (`AppMeasurementService`, `AppMeasurementReceiver`, `AppMeasurementJobService`) | Có mặt |
| Firebase Crashlytics | Có mặt nhưng meta-data `firebase_crashlytics_collection_enabled = false` |
| Meta-data `firebase_analytics_collection_enabled` | `false` |
| Meta-data `google_analytics_automatic_screen_reporting_enabled` | `false` |
| Firebase Cloud Messaging (push) | Có (`SlFirebaseMessagingService`, `FirebaseInstanceIdReceiver`) |
| Firebase Remote Config / Dynamic Links / Installations / Sessions / ABT | Có |
| Install referrer | `BIND_GET_INSTALL_REFERRER_SERVICE` |
| Google Awareness API key | **Lộ trong manifest**: `com.google.android.awareness.API_KEY = AIzaSyDj9mJ2ea9RTxXFy0pnn4G3u7jE14RI7L0` |
| Ad ID / Privacy Sandbox | `ACCESS_ADSERVICES_AD_ID`, `ACCESS_ADSERVICES_ATTRIBUTION` |

Lưu ý: các cờ `*_collection_enabled=false` chỉ là meta-data mặc định trong manifest — SDK vẫn có thể bật lại bằng code/remote config; quyền Ad ID và `AdvertisingIdClient` vẫn tồn tại.

### 11.4 Quyền nhạy cảm đáng lưu ý

Launcher nên có nhiều quyền, nhưng đây là các quyền "mạnh" cần biết khi mod:

- `QUERY_ALL_PACKAGES` (bắt buộc với launcher — liệt kê mọi app đã cài)
- `BIND_NOTIFICATION_LISTENER_SERVICE` + service `NotificationListener` (đọc toàn bộ notification)
- Accessibility service `PanelActionService` (`BIND_ACCESSIBILITY_SERVICE`) — điều khiển UI toàn hệ thống
- `FOREGROUND_SERVICE_MEDIA_PROJECTION` (chụp màn hình qua MediaProjection)
- `READ_CONTACTS`, `READ_CALENDAR`, `ACCESS_FINE_LOCATION` (widget liên hệ/lịch/thời tiết)
- `WRITE_SETTINGS`, `EXPAND_STATUS_BAR`, `SET_WALLPAPER`, `CALL_PHONE`, `REQUEST_DELETE_PACKAGES`

---

## 12. Các điểm đáng chú ý khác

1. **App không bị pack → mod được.** Không có lớp bảo vệ nào ngoài R8.
2. **LeakCanary còn trong bản release**: provider `ginlemon.flowerfree.leakcanary-installer` và class `leakcanary.internal.MainProcessAppWatcherInstaller`. Provider bị tắt bởi `res/values/bools.xml` → `leak_canary_watcher_auto_install = false`, nên watcher không tự cài — nhưng code và provider vẫn nằm trong APK. Đây là dấu hiệu build chưa dọn sạch dependency debug.
3. **Activity debug/test lộ ra ngoài** (`exported="true"`): `ginlemon.flower.library.SLPatternTestActivity`, `ginlemon.flower.graphic.UIdemoActivity`, `androidx.compose.ui.tooling.PreviewActivity`.
4. **`LicenseStatusProvider` (authority `ginlemon.flowerfree.haspro`) là exported** — trạng thái premium có thể bị truy vấn từ app khác; đây là điểm hay bị khai thác khi mod premium (chỉ ghi chú, chưa đụng tới).
5. **`android:persistent="true"`** — app yêu cầu được hệ thống giữ chạy thường trực (đặc quyền thường chỉ app hệ thống mới có hiệu lực).
6. **`android:excludeFromRecents="true"`** và `allowBackup="false"` — launcher không hiện trong recent apps, không backup qua ADB.
7. **Có Split APK** → nếu mod và cài đặt, phải xử lý cả 4 file (base + 3 split) hoặc dùng cách merge thành APK đơn; nếu chỉ sửa base mà cài lên máy đã có bản Play Store sẽ lỗi chữ ký.
8. **Không có chữ ký v1** → khi rebuild, cần ký bằng apksigner với scheme v2/v3, chỉ dùng `jarsigner` sẽ không cài được trên Android 11+.

---

## 13. Ghi chú cho bước mod sau

Chưa thực hiện bất kỳ thay đổi nào. Khi cần mod, các vị trí đáng chú ý (đã xác định, chưa sửa):

| Mục tiêu | Vị trí |
|---|---|
| Logic premium | `smali/ginlemon/flower/billing/LicenseStatusProvider.smali`, `smali/ginlemon/flower/premium/**` |
| Paywall | `smali*/ginlemon/flower/premium/paywall/**` (4 activity) |
| Nhãn app | `res/values/strings.xml` → `app_label` = "Smart Launcher" |
| Icon | **Không có thư mục `res/mipmap*/`** — icon là `res/drawable-anydpi-v26/ic_launcher.xml` (adaptive) + `res/drawable-xxxhdpi/ic_launcher.png` |
| Entry chính | `smali/ginlemon/flower/HomeScreen.smali` |
| Điểm vào từ ngoài | `ginlemon.flower.preferences.PrefSectionActivity` (exported, MAIN) |
| LeakCanary | `res/values/bools.xml` → `leak_canary_watcher_auto_install` (đang là `false`) |

**Nhắc lại:** bản decode này thiếu native libs (`arm64_v8a` split), chuỗi tiếng Việt (`vi` split) và ảnh xxhdpi, nên test UI/chuỗi trên bản decode sẽ không đầy đủ.
