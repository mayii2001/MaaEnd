# 手机端（ADB 真机）适配说明

本分支 `phone-adb-adapt` 基于上游 [MaaEnd/MaaEnd](https://github.com/MaaEnd/MaaEnd) `v2`，叠加了自用的 ADB 真机（手机端）适配改动。本文档说明分支内容与配套本地机制。

## 一、分支相对上游 v2 的改动

### 1. 合并上游 PR #4305：领取奖励适配 ADB

- 来源：上游 PR [#4305](https://github.com/MaaEnd/MaaEnd/pull/4305)（feat: 领取奖励适配ADB，@overflow65537）
- 内容：ProtocolPass / Tasks / LevelUpOperator / Weapon / Crafting 等任务的 pipeline 源文件改动，以及 `resource_adb` 下的新增图片与 JSON（如 `DailyRewards/ProtocolPass.json`、`DailyRewards/Tasks.json`、`LevelUpOperator.json` 等）
- 合并无冲突

### 2. `__ScenePrivateEndVisit` OCR ROI 修正

- 文件：`assets/resource/pipeline/SceneManager/SceneWorld.json`
- 改动：OCR 的 `roi` 由 `[-260,-100,260,100]` 改为 `[600,340,620,160]`
- 原因：「结束拜访」按钮实际出现在画面中下部（真机 1280x720 下约 x850-1010、y400-435），原 ROI 位于右下角，识别不到该按钮

### 3. 好友列表直达分支

- 文件：`assets/resource/pipeline/SceneManager/SceneMenu.json`
- 改动：
  - 新增节点 `__ScenePrivateMenuFriendsEnterMenuFriendsListEntry`（DirectHit，无 recognition；`next` 为 `["__ScenePrivateMenuFriendsEnterMenuFriendsListSuccess", "__ScenePrivateMenuFriendsEnterMenuFriendsList"]`；`pre_delay`/`post_delay`/`rate_limit` 均为 0）
  - `__ScenePrivateWorldEnterMenuFriendsList` 的 anchor `__ScenePrivateMenuListEnterMenuFriendsNextAnchor` 由 `__ScenePrivateMenuFriendsEnterMenuFriendsList` 改为指向新节点
- 原因：游戏会记住子页面，当直接落在好友列表页时，入口 tab 处于选中态，与模板（未选中态）匹配失败导致卡死；新增 DirectHit 节点后，已在列表页则直接判定成功，否则再点击列表入口

### 4. 进拍照改为轮盘拖动方案（替代图标模板点击）

- 文件：
  - `assets/resource_adb/pipeline/EnvironmentMonitoring.json`（新增）：覆盖 `EnvironmentMonitoringTakePhotoDirectly`，`next` 仅保留 `EnterCameraModeLongPress`，跳过 `EnterCameraModeClick` 的图标模板点击路径，直接走 TouchDown → TouchMove（拖向 10-11 点方向）→ TouchUp 的按住拖动链
  - `assets/tasks/setting/Keymap.json`：移除 `KeymapGeneral` 的 `pipeline_override`（`EnterCameraModeLongPress` 的 KeyDown、`EnterCameraModeKeyUp` 的 KeyUp 两项，置为 `{}`），hotkeys 定义保留
- 原因：
  - 快捷道具轮盘按钮图标随最后使用的功能变化，不固定，图标模板方案不可靠；且实测普通（相机机身）与 swirl 两种外观互相匹配分仅 0.53（阈值 0.7），双模板也无法覆盖所有形态
  - 正确交互是按住该槽位向 10-11 点方向拖动，出现拍照选项后释放。原版 `resource_adb` 已有 TouchDown/TouchMove/TouchUp 拖动链（`EnterCameraModeLongPress` → `EnterCameraModeChooseCamera` → `EnterCameraModeKeyUp`），但 Keymap 的任务级 override 会把 `EnterCameraModeLongPress`/`EnterCameraModeKeyUp` 顶成 KeyDown/KeyUp 键盘事件，真机不响应注入键盘，故移除该 override，并让 `EnvironmentMonitoringTakePhotoDirectly` 跳过 Click 路径直接进入拖动链
- 备注：`EnterCameraModeButton.png`（swirl 形态）与 `EnterCameraModeButton2.png`（普通形态）的双形态模板文件保留，但已被本轮盘方案取代，仅作备用

## 二、本地运行机制（不入库，仅说明）

以下机制只存在于本地运行目录，用于在 app 更新后自动恢复真机适配，不随本分支提交：

- `config/overrides/` 目录存放 pipeline / 图片覆盖文件，同时支持 `resource_adb` 与 `tasks` 目录
- `tools/apply_local_overrides.py` 在 `startup.bat` 启动时将覆盖深合并到 `resource_adb`（及 `tasks`），app 更新重置资源后自动恢复
- `tools/calibrate_roi.py` 用真机截图做模板匹配以校准 ROI 并写回 `config/overrides`；已有配置且自检分 ≥ 0.7 则跳过，只校准一次
- `tools/calibrate_hud.py` 批量 HUD 校准：`calibrate_targets.json` 维护校准目标，支持绿幕模板、每目标 threshold、apply 附加写入——用槽位外圈环形模板 `QuickToolSlotRing.png`（绿幕遮挡内圈，与图标无关）定位快捷道具槽位后，同步写入 `EnterCameraModeLongPress`/`EnterCameraModeChooseCamera` 的 target 坐标，适配自定义 HUD 布局

## 三、其他本地改动（不入库）

- `finish.bat` 改进：恢复分辨率后执行 `am kill-all` 并重启 launcher，解决部分应用缓存旧显示度量导致页面比例不恢复的问题

## 四、1.25 倍 UI 缩放导致的模板识别问题

- **现象**：PC 版模板在 ADB 真机（720x1280 @320dpi，游戏横屏 1280x720）上模板匹配分普遍只有 0.5 左右（实测 `EnterCameraModeButton` PC 模板在真机截图上 0.51~0.54，阈值 0.7/0.8/0.9 均不达标），位置找得对但分数不达标。
- **原因**：真机上游戏 UI 按约 1.25 倍渲染。对比：PC 模板 `EnterCameraModeButton.png` 为 20x15，真机同按钮约 24x18 ≈ 1.2~1.25 倍；上游 `resource_adb` 自带模板也多为 PC 模板的 1.25 倍放大版。尺寸不一致导致 NCC 相关性下降。
- **对策**（本分支已采用）：
  1. ADB 专用模板必须用真机截图裁取（如 `EnterCameraModeButton.png` 从真机 1280x720 截图裁 `[1162,270,24,18]`），裁取后在多张真机截图上验证分数 ≥ 0.9、无目标画面 ≤ 0.6；
  2. 注意区分「缩放不匹配」与「图标本来不同」：轮盘槽位按钮外观随功能变化，属于后者，不能用补模板解决（见第 4 节轮盘拖动方案）；
  3. 本地 `tools/calibrate_hud.py` 校准时的匹配分同样受 1.25 倍影响，PC 模板校准会失败，所以校准模板也要用真机裁取的版本。
- **验证方法**：用 numpy 手写 NCC（`TM_CCOEFF_NORMED` 等价）在 `on_error` 截图上复算分数，即可近似 MAA 的 TemplateMatcher 结果（已验证两者一致）。

## 五、vivo 虚拟屏导致 ADB 截图失败（MaaToolkit.dll 补丁）

- **现象**：部分 vivo 手机存在系统虚拟屏 `vivo_rms_screen`（见上游 issue [MaaEnd/MaaEnd#4289](https://github.com/MaaEnd/MaaEnd/issues/4289)，已关闭未修），ADB 连接后截图失败。
- **根因**：MaaFramework v5.12.1 的 `AdbDeviceFinder` 检测到相关属性后误判设备为 Androws 模拟器，强制使用 `screencap -d 90000`，在真机上无法出图。
- **修复方式**：本分支附带修复版 DLL `patches/maafw/MaaToolkit.dll`（含新逻辑标记串 `sys.tencent.imei is set but device looks like a physical phone, skip Androws detection`，官方 v5.12.1 正式版没有），替换安装目录 `maafw/MaaToolkit.dll` 即可。详见 `patches/README.md`。
- **本地自动打补丁机制**（不入库，仅说明）：`config/overrides/maafw/MaaToolkit.dll` + `tools/apply_local_overrides.py` 随 `startup.bat` 执行，仅当目标 DLL 缺少修复标记串时才覆盖（app 更新带来已修复新版时不降级），已经过「官方版 → apply → 恢复修复版」的模拟验证。
