# Ghi chú phân tích APK SmartLauncher

## Mục lục

- [1. Thông tin ứng dụng](#1-thông-tin-ứng-dụng)
- [2. Permission](#2-permission)
- [3. Activity khởi chạy](#3-activity-khởi-chạy)
- [4. Service, receiver và provider](#4-service-receiver-và-provider)
- [5. Cấu trúc smali](#5-cấu-trúc-smali)
- [6. Cấu trúc resource](#6-cấu-trúc-resource)
- [7. Nhận xét sơ bộ](#7-nhận-xét-sơ-bộ)

## 1. Thông tin ứng dụng

- **Package:** `ginlemon.flowerfree`
- **Version name:** `6.6 build 024`
- **Version code:** `660240`
- **Min SDK:** `24`
- **Target SDK:** `36`
- Nguồn: `AndroidManifest.xml` và `apktool.yml`.

## 2. Permission

Các permission được khai báo trong Manifest:

- `com.android.vending.BILLING`
- `android.permission.FOREGROUND_SERVICE`
- `android.permission.FOREGROUND_SERVICE_MEDIA_PROJECTION`
- `android.permission.EXPAND_STATUS_BAR`
- `android.permission.ACCESS_HIDDEN_PROFILES`
- `android.permission.CALL_PHONE`
- `android.permission.READ_CALENDAR`
- `android.permission.WRITE_EXTERNAL_STORAGE`
- `android.permission.READ_MEDIA_AUDIO`
- `android.permission.READ_MEDIA_VIDEO`
- `android.permission.READ_MEDIA_IMAGES`
- `android.permission.VIBRATE`
- `android.permission.SET_WALLPAPER`
- `android.permission.SET_WALLPAPER_HINTS`
- `android.permission.INTERNET`
- `android.permission.ACCESS_NETWORK_STATE`
- `android.permission.READ_CONTACTS`
- `android.permission.WRITE_SETTINGS`
- `android.permission.QUERY_ALL_PACKAGES`
- `android.permission.USE_FINGERPRINT`
- `android.permission.USE_BIOMETRIC`
- `android.permission.REQUEST_DELETE_PACKAGES`
- `android.permission.READ_EXTERNAL_STORAGE`
- `android.permission.ACCESS_FINE_LOCATION`
- `android.permission.ACCESS_COARSE_LOCATION`
- `android.permission.BIND_NOTIFICATION_LISTENER_SERVICE`
- `android.permission.POST_NOTIFICATIONS`
- `android.permission.WAKE_LOCK`
- `com.google.android.c2dm.permission.RECEIVE`
- `android.permission.RECEIVE_BOOT_COMPLETED`
- `com.google.android.finsky.permission.BIND_GET_INSTALL_REFERRER_SERVICE`
- `ginlemon.flowerfree.DYNAMIC_RECEIVER_NOT_EXPORTED_PERMISSION`

## 3. Activity khởi chạy

Activity có intent-filter `MAIN` + `LAUNCHER`:

- `ginlemon.flower.HomeScreen`

## 4. Service, receiver và provider

### Service

- `ginlemon.flower.wallpaperScreenshot.WallpaperFrameCaptureService`
- `ginlemon.flower.library.PanelActionService`
- `ginlemon.flower.core.cloudMessaging.SlFirebaseMessagingService`
- `ginlemon.notifications.listener.NotificationListener`
- `com.google.firebase.components.ComponentDiscoveryService`
- `com.google.firebase.sessions.SessionLifecycleService`
- `com.google.firebase.messaging.FirebaseMessagingService`
- `androidx.work.impl.background.systemjob.SystemJobService`
- `androidx.work.impl.foreground.SystemForegroundService`
- `androidx.room.MultiInstanceInvalidationService`
- `com.google.android.gms.measurement.AppMeasurementService`
- `com.google.android.gms.measurement.AppMeasurementJobService`
- `com.google.android.datatransport.runtime.backends.TransportBackendDiscovery`
- `com.google.android.datatransport.runtime.scheduling.jobscheduling.JobInfoSchedulerService`

### Receiver

- `ginlemon.flower.feedrss.presentation.inAppRead.ReadingModeReceiver`
- `com.google.firebase.iid.FirebaseInstanceIdReceiver`
- `androidx.work.impl.utils.ForceStopRunnable$BroadcastReceiver`
- `androidx.work.impl.background.systemalarm.RescheduleReceiver`
- `androidx.work.impl.diagnostics.DiagnosticsReceiver`
- `com.google.android.gms.measurement.AppMeasurementReceiver`
- `androidx.profileinstaller.ProfileInstallReceiver`
- `com.google.android.datatransport.runtime.scheduling.jobscheduling.AlarmManagerSchedulerBroadcastReceiver`

### Provider

- `ginlemon.flowerfree.feedopml.SmartLauncherFeedOpmlProvider`
- `ginlemon.flower.provider.KustomProvider`
- `ginlemon.flower.preferences.SettingsProvider`
- `ginlemon.flower.billing.LicenseStatusProvider`
- `androidx.core.content.FileProvider`
- `ginlemon.flower.wallpaper.SetWallpaperProvider`
- `com.google.firebase.provider.FirebaseInitProvider`
- `leakcanary.internal.MainProcessAppWatcherInstaller`
- `com.squareup.picasso.PicassoProvider`

## 5. Cấu trúc smali

Có 3 thư mục dex đã giải mã, tổng cộng **18.753 file `.smali`**:

| Thư mục | Số file |
|---|---:|
| `smali/` | 5.962 |
| `smali_classes2/` | 12.031 |
| `smali_classes3/` | 760 |

Các package cấp cao nhất:

- `smali/`: `androidx/`, `com/`, `dagger/`, `ginlemon/`, `kotlin/`, `kotlinx/`, `org/`
- `smali_classes2/`: `android/`, `androidx/`, `com/`, `ginlemon/`, `javax/`, `kotlin/`, `kotlinx/`, `leakcanary/`, `me/`, `net/`, `okhttp3/`, `org/`
- `smali_classes3/`: `org/`, `tech/`

Các package con dưới `ginlemon/` được tìm thấy: `flower/`, `flowerfree/`, `iconretrieving/`, `launcher/`, `library/`, `notifications/`, `weatherproviders/`. Riêng `ginlemon/flower/` có **391 file smali**.

## 6. Cấu trúc resource

Các nhóm thư mục chính trong `res/`:

- `layout/` cùng các biến thể `layout-land/`, `layout-sw600dp/`, `layout-v26/`, `layout-watch/` (**179 file** gộp).
- `values/` và các biến thể theo locale, kích thước, giao diện và API (**123 thư mục, 233 file** gộp).
- `drawable/` cùng các biến thể density, night/API, RTL, nodpi và watch (**17 thư mục, 988 file** gộp).
- Nhóm khác: `anim/`, `animator/`, `color/` (gồm `color-night/`, `color-v26/`), `font/`, `interpolator/`, `raw/`, `xml/`, `xml-v35/`.

## 7. Nhận xét sơ bộ

- **Obfuscation:** có dấu hiệu obfuscation ở một phần mã. Trong toàn bộ smali có nhiều lớp và hàm tên rất ngắn; riêng `ginlemon/flower/`, 12/391 tên lớp dài tối đa 2 ký tự và 878/2.234 tên hàm dài tối đa 2 ký tự. Tuy vậy, phần lớn tên package/lớp tính năng của ứng dụng vẫn có ý nghĩa, nên không phải toàn bộ mã đều bị làm rối. Tên ngắn cũng có thể xuất hiện từ mã sinh tự động hoặc thư viện.
- **Billing:** có dấu hiệu rõ ràng sử dụng Google Play Billing: permission `com.android.vending.BILLING`, metadata cho Billing Library **8.3.0**, package `com.android.billingclient` và các `ProxyBillingActivity`.
- **Quảng cáo:** chưa thấy package SDK quảng cáo phổ biến như AdMob trong các đường dẫn smali đã kiểm tra. Có 4 lớp thuộc `com.google.android.gms.ads.identifier` (Advertising ID); chỉ riêng thành phần này không đủ để kết luận app tích hợp SDK phân phối quảng cáo.
- Đây là nhận xét tĩnh sơ bộ từ Manifest và cấu trúc smali/resource, không khẳng định hành vi lúc chạy.
