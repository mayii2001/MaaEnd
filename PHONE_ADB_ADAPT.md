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

### 4. `EnterCameraModeButton` 真机双形态模板

- 文件：
  - `assets/resource_adb/image/Common/Button/EnterCameraModeButton.png`（替换为 swirl 形态真机截图，24x18 RGBA，裁自 1280x720 真机截图 `[1162,270]`）
  - `assets/resource_adb/image/Common/Button/EnterCameraModeButton2.png`（新增，普通形态相机机身图标，24x18 RGBA，裁自真机截图 `[1162,271]`）
  - `assets/resource_adb/pipeline/Common/Button/EnterCameraModeButton.json`（`template` 字段更新为上述两张图）
- 原因：原模板是 PC 版相机机身图标，真机靠近拍摄目标时按钮渲染为圆形 swirl 提示图标，模板分仅 0.53（阈值 0.7），导致 `EnterCameraModeClick` 走不通、回退到真机不响应的按键路径。且真机上该按钮有普通（相机机身）与靠近目标（swirl）两种外观，两种外观互相匹配分也只有 0.53，单模板必然漏一种，故使用双模板

## 二、本地运行机制（不入库，仅说明）

以下机制只存在于本地运行目录，用于在 app 更新后自动恢复真机适配，不随本分支提交：

- `config/overrides/` 目录存放 pipeline / 图片覆盖文件
- `tools/apply_local_overrides.py` 在 `startup.bat` 启动时将覆盖深合并到 `resource_adb`，app 更新重置资源后自动恢复
- `tools/calibrate_roi.py` 用真机截图做模板匹配以校准 ROI 并写回 `config/overrides`；已有配置且自检分 ≥ 0.7 则跳过，只校准一次

## 三、其他本地改动（不入库）

- `finish.bat` 改进：恢复分辨率后执行 `am kill-all` 并重启 launcher，解决部分应用缓存旧显示度量导致页面比例不恢复的问题
