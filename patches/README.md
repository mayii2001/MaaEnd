# patches

本目录存放自用补丁文件，仅配合 `phone-adb-adapt` 分支使用，不向上游提 PR。

## maafw/MaaToolkit.dll

- **来源**：修复版 MaaFramework 构建（基于 v5.12.1 修改 AdbDeviceFinder 逻辑后自行编译），非官方发布版。
- **修复问题**：上游 issue [MaaEnd/MaaEnd#4289](https://github.com/MaaEnd/MaaEnd/issues/4289)（已关闭未修）。部分 vivo 手机存在系统虚拟屏 `vivo_rms_screen`，MaaFramework v5.12.1 的 `AdbDeviceFinder` 检测到 `sys.tencent.imei` 等属性后误判设备为 Androws 模拟器，强制使用 `screencap -d 90000` 截图，导致 ADB 截图失败。
- **修复方式**：新增判断逻辑——`sys.tencent.imei` 已设置但设备看起来是物理手机时跳过 Androws 检测（DLL 内包含标记串 `sys.tencent.imei is set but device looks like a physical phone, skip Androws detection`，官方 v5.12.1 正式版没有此字符串）。
- **用法**：用本文件替换安装目录下的 `maafw/MaaToolkit.dll`。
- **注意事项**：
  - 官方正式版 v5.12.1 未包含此修复；
  - 若未来官方新版 DLL 已含 `skip Androws detection` 字符串，则无需再打本补丁；
  - 文件校验：md5 `445728af3961fe79c6205718e73cdcc5`，大小 544768 字节。
