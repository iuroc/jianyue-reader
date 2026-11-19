# 权限配置

## 最终集合

只保留 `android.permission.INTERNET`。应用数据全部写 `filesDir`，不读外部存储、不安装 APK。

`AndroidManifest.xml` 现已同步为：

```xml
<Permissions>android.permission.INTERNET</Permissions>
```

## iApp 的权限页才是打包来源

- iApp 工程设置里的权限页是一份**固定目录**；`AndroidManifest.xml` 的 `<Permissions>`
  只决定打开工程时勾选目录里的哪几项。
- 工程被打开过之后，打包以权限页的勾选状态为准；改源目录文件**不会**回写。
- 目录外的权限即使写进文件也会被忽略。实测 5.0.1015 识别 6 项：
  `INTERNET`、`REQUEST_INSTALL_PACKAGES`、`CAMERA`、`RECORD_AUDIO`、
  `READ_EXTERNAL_STORAGE`、`WRITE_EXTERNAL_STORAGE`；
  `ACCESS_FINE_LOCATION`、`READ_CONTACTS`、`POST_NOTIFICATIONS` 未识别。
  清单可能随版本变化，以目标机实测为准。

## 改动流程

1. 在 iApp 权限页勾选/取消权限（本工程：只勾 `INTERNET`，取消 `WRITE_EXTERNAL_STORAGE`）；
2. 重新打包并安装；
3. 核对实际请求：

   ```bash
   adb -s <serial> shell dumpsys package com.jianyue.reader | grep -A 12 "requested permissions"
   ```

4. 把设备侧 `/sdcard/Android/data/com.iapp.app/files/iApp/ProjectApp/com.jianyue.reader/AndroidManifest.xml`
   抄回本工程的 `AndroidManifest.xml`，提交；避免下次整目录推送覆盖。

## 权限闸门

- `perm.mjava` 只检查 `INTERNET`。
- `INTERNET` 是安装期权限，不能用 `requestPermissions` 临时申请；缺权限时
  `home.buildGate()` 只提示“安装包缺少网络权限，请重新打包安装”，不提供应用内授权按钮。
- 首次安装包若显示该闸门，说明 iApp 权限页没勾 `INTERNET`，按上面的流程重打包。

## 验证记录

- 权限探针 `com.demo.permprobe`：9 项声明 → iApp 权限页高亮 6 项，目录外 3 项被忽略 ✅ 2026-11-19。
- 已装 APK（23:46）缺 `INTERNET`，与 iApp 权限页当时只有 `WRITE` 一致 ✅ 2026-11-19。
- 2026-11-20 重新打包后 `dumpsys package com.jianyue.reader` 的 requested permissions 已含
  `android.permission.INTERNET: granted=true`，权限闸门不再出现 ✅。
  iApp 模板仍会附带 `POST_NOTIFICATIONS` / `MANAGE_EXTERNAL_STORAGE` / `QUERY_ALL_PACKAGES`，
  不影响本应用。
