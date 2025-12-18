# rpicam-apps (Raspberry Pi 5 / Ubuntu 24.04 LTS 兼容版)

这是对上游 `raspberrypi/rpicam-apps` 的兼容性改动版本，目标是让代码在 **Raspberry Pi 5 + Ubuntu 24.04 LTS**（系统自带较旧的 `libcamera` 头文件/ABI 组合）环境中可以正常编译通过并完成基础功能。

## 适用环境

- 硬件：Raspberry Pi 5
- 系统：Ubuntu 24.04 LTS（aarch64）
- `libcamera`：系统头文件显示为 `0.2.0`（见 `/usr/include/libcamera/libcamera/version.h`）

## 与上游不同之处（本仓库主要改动）

本 fork 的核心是“向后兼容”，因此有些上游新增功能会被降级或跳过。

- `libav`/FFmpeg 兼容：放宽 `libavcodec` 版本门槛，并对音频通道布局、重采样等 API 做新旧分支兼容（`ch_layout` vs `channel_layout/channels`，`swr_alloc_set_opts2` vs `swr_alloc_set_opts`）。
- `libcamera controls` 兼容：在旧版 `libcamera` 中，部分控制项处于 `controls::draft` 或不存在；本版本对 AE 状态等元数据读取做了兼容处理。
- 像素格式兼容：移除旧版 `libcamera::formats` 中不存在的像素格式（例如部分 PiSP/高位深 RGB/Mono 格式），相关保存/识别逻辑做降级处理。
- 同步/厂商扩展控制降级：若系统头文件不包含对应的 `controls::rpi::*` 项，则同步相关逻辑会被关闭，避免编译失败。

## 构建

如果你已经在本机安装了构建依赖并创建了 `build/`：

```bash
meson compile -C build
```

首次配置可参考上游官方文档（构建依赖以你的发行版为准）：
https://www.raspberrypi.com/documentation/computers/camera_software.html#building-libcamera-and-rpicam-apps

## 免责声明

本仓库不是 Raspberry Pi 官方发布版本，仅用于在特定环境（Raspberry Pi 5 + Ubuntu 24.04 LTS）下解决编译兼容性问题。若你使用的是 Raspberry Pi OS 或更新的 `libcamera`/FFmpeg 组合，建议优先使用上游仓库。

## License

The source code is made available under the simplified [BSD 2-Clause license](https://spdx.org/licenses/BSD-2-Clause.html).
