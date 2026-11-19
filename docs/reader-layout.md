# 阅读页布局规则

只适用于本工程阅读页（`src/reader.mjava` / `src/ui.mjava`）。通用 iApp 知识见工作区
`.claude/skills/iapp-development/references/`，这里不重复。

## 全屏姿态

- 进阅读页调用 `rd_fullscreen()`：`hide(systemBars())`，状态栏和导航栏都隐藏；
  叠 `IMMERSIVE_STICKY | HIDE_NAVIGATION | LAYOUT_HIDE_NAVIGATION` 兜底。
- 边缘滑动可临时唤出系统栏；API 30 / MIUI 上是否能自动收回 ❓（未单独复验）。
- 阅读容器的 `safe` padding 四边置 0，WebView 铺满整屏；`overlay`（目录等原生浮层）
  仍保留左右/底部避让。

## 边距全部来自 WebView CSS

| 位置 | 竖屏 | 横屏 |
|---|---|---|
| 顶边距 | 状态栏/挖孔高度 + 8dp | 与左右边距相等 |
| 左右边距 | 18px | `max(18, 挖孔宽度换算成 CSS px)` |
| 底边距 | `76px + 导航条高度` | 同左（导航栏隐藏时导航条高度为 0） |

- 顶/底边距必须放在 `body` 的 padding 上：它属于滚动内容，下滑时会移出屏幕；
  容器 padding 是滚动区域外的固定留白，达不到这个效果。
- 段距用 `margin-top`，不用 `margin-bottom`；注入前删掉服务端正文末尾的
  `<p></p>` / `<p> </p>` 等空段落，否则最后一行和操作栏之间会多一段外边距。
- 挖孔在横屏时位于左侧或右侧，用 `displayCutout` 的 `left/right` 取宽度；
  上/左/右三边统一取该值，保证等距且正文不压摄像头。

## WebView 约束

- **JavaScript 未开启**（`setJavaScriptEnabled` 从未设置）：`evaluateJavascript` 返回 null，
  HTML 里的 `<script>` 不执行。所有布局/样式走 CSS，不要用 JS 动态改 padding。
- 旋转后不能靠 JS 更新边距，改为重新渲染当前页并恢复滚动比例（见下）。

## 旋转

- Android 11 / iApp 5.0.1015 实测：旋转**不会重建**页面（同进程、无新的 `loading`）。
- `rd_watchConfig()` 注册 `ComponentCallbacks.onConfigurationChanged`：
  - 第一次回调记录滚动比例 `scrollY / maxScroll`，后续回调不覆盖；
  - 用时间戳 token 去重，只让最后一次回调执行重渲染；
  - 重渲染后等内容高度连续两次稳定，再按比例恢复滚动位置。

## 底部操作栏

- 高度 = `48dp + 导航条高度`，底部 padding = 导航条高度，按钮留在上面 48dp。
- 显示时背景覆盖导航条区域（导航栏透明时）；隐藏时整条 GONE，正文透出到导航条下层。
- 自动弹出距离 `rd_endZone()` = `(body 底边距) - (操作栏高度)`，只在最后一行已位于
  操作栏上方时弹出，避免盖住正文。

## 外观抽屉（Aa）

- `BottomSheetDialog` 默认 `PEEK_HEIGHT_AUTO`：横屏时
  `parentHeight - parentWidth*9/16` 为负，会回落到 `peekHeightMin` 并停在 collapsed。
- `show()` 后取 `BottomSheetBehavior`：`setSkipCollapsed(true)` + `setState(STATE_EXPANDED)`。
- `design_bottom_sheet` 资源 id 用 `activity.getPackageName()` 查（Material 资源已并进宿主包）。

## 验证状态

- 红米 Note 8 / API 30：全屏、顶部/底部/挖孔边距、操作栏贴文、Aa 展开、旋转恢复 ✅ 2026-11-19。
- 腾讯应用宝容器 display 2 / API 33（无系统栏、无挖孔的全面屏）：阅读页正常 ✅ 2026-11-19。
- 打包 APK：❓ 未验；怎么验：打包后装到红米，检查四边边距、横竖屏旋转与 Aa 抽屉。
  打包版 `getRootWindowInsets()` 可能返回 null，`ui_insets` 会走 `status_bar_height` 兜底，
  顶部边距可能与开发模式不同。
