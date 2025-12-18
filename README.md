# rpicam-apps (Raspberry Pi 5 / Ubuntu 24.04 LTS 兼容版)

这是对上游 `raspberrypi/rpicam-apps` 的兼容性改动版本，目标是让代码在 **Raspberry Pi 5 + Ubuntu 24.04 LTS**（系统自带较旧的 `libcamera` 头文件/ABI 组合）环境中可以正常编译通过并完成基础功能。

## 适用环境

- 硬件：Raspberry Pi 5
- 系统：Ubuntu 24.04 LTS（aarch64）
- `libcamera`：系统头文件显示为 `0.2.0`（见 `/usr/include/libcamera/libcamera/version.h`）

> [!IMPORTANT]
> 这里的 `0.2.0` 指的是 Ubuntu 24.04 下通过 `sudo apt install libcamera-dev` 默认安装到系统里的版本（相对较旧）。
> 本仓库通过修改源码实现了对该版本的编译支持，但经过实测：**仅通过 apt 安装的 libcamera 版本，可能无法与 rpicam-apps 很好对接**。
> 如果你编译安装后的 `rpicam-apps` 仍然“找不到摄像头”，优先尝试 **本地编译并安装 libcamera**，然后再编译安装本仓库的 `rpicam-apps`，通常能解决问题。
> 详细步骤见 `附：OV5647 自动对焦摄像头的完整配置过程`。

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

## 附：OV5647 自动对焦摄像头的完整配置过程（Raspberry Pi 5 + Ubuntu 24.04 LTS + Rpicam with tensorflow-lite）

需严格按照以下步骤进行。

### part1：本地编译 libcamera

不要从 apt 系统安装 `libcamera*` 软件包。

在构建 libcamera 之前，请安装以下软件包：

```bash
sudo apt install -y build-essential
sudo apt install -y libboost-dev
sudo apt install -y libgnutls28-dev openssl libtiff-dev pybind11-dev
sudo apt install -y qtbase5-dev libqt5core5a libqt5widgets5t64
sudo apt install -y meson cmake
sudo apt install -y python3-yaml python3-ply
#sudo apt install -y libglib2.0-dev libgstreamer-plugins-base1.0-dev
```

完成上述步骤后，将 libcamera 代码库克隆到本地，并使用以下命令进行构建：

```bash
git clone https://github.com/raspberrypi/libcamera.git
cd libcamera
meson setup build --buildtype=release -Dpipelines=rpi/vc4,rpi/pisp -Dipas=rpi/vc4,rpi/pisp -Dv4l2=true -Dgstreamer=enabled -Dtest=false -Dlc-compliance=disabled -Dcam=disabled -Dqcam=disabled -Ddocumentation=disabled -Dpycamera=enabled
ninja -C build install
sudo ninja -C build install
```

### part2（可选）：获取 tensorflow-lite

方法来自：https://github.com/prepkg/tensorflow-lite-raspberrypi

执行以下命令：

```bash
wget https://github.com/prepkg/tensorflow-lite-raspberrypi/releases/latest/download/tensorflow-lite_64.deb
sudo apt install -y ./tensorflow-lite_64.deb
```

### part3：从源代码安装 rpicam-apps

```bash
git clone https://github.com/igttttma/rpicam-apps-rpi5-ubuntu2404.git
cd rpicam-apps-rpi5-ubuntu2404
sudo apt install -y cmake libboost-program-options-dev libdrm-dev libexif-dev
sudo apt install -y ffmpeg libavcodec-extra libavcodec-dev libavdevice-dev libpng-dev libpng-tools libepoxy-dev
sudo apt install -y qt5-qmake qtmultimedia5-dev
```

```bash
meson setup build -Denable_libav=enabled -Denable_drm=enabled -Denable_egl=enabled -Denable_qt=enabled -Denable_opencv=disabled -Denable_tflite=enabled -Denable_hailo=disabled
```

这里注明，完成了 part2 后，才可设置 `-Denable_tflite=enabled`，如不需要可设置为 `disabled`。

```bash
meson compile -C build
sudo meson install -C build
sudo ldconfig
```

### part4：设置自动对焦

下载 `ov5647_af.json` 并移动到 `/usr/local/share/libcamera/ipa/rpi/pisp`：

```bash
wget -O /tmp/ov5647_af.json https://raw.githubusercontent.com/ArduCAM/libcamera/arducam/src/ipa/rpi/pisp/data/ov5647_af.json
sudo install -D -m 644 /tmp/ov5647_af.json /usr/local/share/libcamera/ipa/rpi/pisp/ov5647_af.json
```

### part5：修改配置文件

```bash
sudo nano /boot/config.txt
```

Find the line: `camera_auto_detect=1`, update it to:

```
camera_auto_detect=0
```

Find the line: `[all]`, add the following item under it:

```
dtoverlay=ov5647,cam0
```

Save and reboot.

重启，大功告成。

## 免责声明

本仓库不是 Raspberry Pi 官方发布版本，仅用于在特定环境（Raspberry Pi 5 + Ubuntu 24.04 LTS）下解决编译兼容性问题。若你使用的是 Raspberry Pi OS 或更新的 `libcamera`/FFmpeg 组合，建议优先使用上游仓库。

## License

The source code is made available under the simplified [BSD 2-Clause license](https://spdx.org/licenses/BSD-2-Clause.html).
