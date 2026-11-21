# rpi_camera_ros2_ws

A ready-to-build ROS 2 Jazzy workspace for using Raspberry Pi cameras (CSI) with ROS 2.

It contains the upstream [libcamera](https://gitlab.freedesktop.org/camera/libcamera) and [camera_ros](https://github.com/christianrauch/camera_ros) packages in `src/`. Clone it, build it, then follow this guide to configure and calibrate your camera.

## Why this repo?

The ROS-packaged `libcamera` often doesn't work with Raspberry Pi cameras. The upstream `libcamera` has to be built from source in the same workspace as `camera_ros`. This repo packages that setup and documents the path from a fresh install to a calibrated camera.

## Tested environment

| Item | Value |
|------|-------|
| Board | Raspberry Pi 4 (4 GB) |
| Camera | Camera Module 1 |
| OS | Ubuntu 24.04 |
| ROS 2 | Jazzy |


## Repository layout

```
rpi_camera_ros2_ws/
├── README.md
├── .gitignore
├── .git/
├── .gitattributes
└── src/
    ├── libcamera/     # upstream libcamera
    └── camera_ros/    # ROS 2 camera node
```

Config files (camera parameters, calibration) are not included. You create them in the steps below because they depend on your camera and setup.

---

## 1. Prerequisites

Install the base tools (skip any you already have):

```bash
sudo apt install -y git python3-colcon-common-extensions python3-rosdep python3-colcon-meson
```

One-time rosdep setup (skip if already done):

```bash
sudo rosdep init
rosdep update
```

Install the libcamera build dependencies (only needed if the build complains):

```bash
sudo apt install -y meson ninja-build pkg-config python3-yaml python3-ply \
  python3-jinja2 openssl libyaml-dev libssl-dev libudev-dev libatomic1
```

Make sure ROS's own libcamera and camera_ros packages are **not** installed, since they conflict with the source build:

```bash
sudo apt remove ros-jazzy-libcamera ros-jazzy-camera-ros
```

## 2. (Optional) Check that the Pi detects the camera

This step is a quick sanity check before building. It installs the system version of libcamera's command-line tools:

```bash
sudo apt install libcamera-tools libcamera-ipa
cam --list
```

You should see your camera listed. If you get "command not found", the tools aren't installed. If the list is empty, check the ribbon cable, the camera connector, and your Pi's camera configuration before continuing.

## 3. Clone and build

```bash
git clone --recurse-submodules https://github.com/Joshua-Ayanlade/rpi_camera_ros2_ws.git ~/camera_ws
cd ~/camera_ws
```

> If you forgot `--recurse-submodules`, run `git submodule update --init --recursive`.

Source ROS 2 and install dependencies:

```bash
source /opt/ros/jazzy/setup.bash

rosdep install -y \
  --from-paths src \
  --ignore-src \
  --rosdistro jazzy \
  --skip-keys=libcamera
```

Build:

```bash
colcon build --event-handlers=console_direct+
source install/setup.bash
```

The first build compiles libcamera and can take a while on a Pi. If it runs out of memory, build with fewer parallel jobs (`colcon build --parallel-workers 1`) or add swap space.

## 4. Test the camera

```bash
ros2 run camera_ros camera_node
```

In another terminal (with the workspace sourced), confirm images are being published:

```bash
ros2 topic list
ros2 topic hz /camera/image_raw
```

To confirm the camera is detected by the libcamera you just built, you can also run its `cam` tool if it was built (the path may vary):

```bash
~/camera_ws/install/libcamera/bin/cam --list
```

## 5. Configure the camera

Create a parameter file, for example `~/camera_ws/camera_params.yaml`:

```yaml
camera:
  ros__parameters:
    camera: 0            # camera index (see `cam --list`)
    format: XRGB8888     # pixel format
    width: 640
    height: 480
```

Start the camera with it:

```bash
ros2 run camera_ros camera_node --ros-args --params-file ~/camera_ws/camera_params.yaml
```

Run `ros2 run camera_ros camera_node --ros-args -p help:=true` or see the [camera_ros documentation](https://github.com/christianrauch/camera_ros) for the other parameters (formats, resolutions, camera controls).

## 6. Calibrate the camera

Calibration produces the intrinsic parameters (camera matrix and distortion coefficients) that many ROS packages need, such as vision, SLAM, and AprilTag.

### What you need

A printed checkerboard (or a checkerboard shown on a screen). Measure the side length of one square accurately, because the calibration depends on it.

> `--size 8x6` means **8x6 inner corners**, which is a board of 9x7 squares. Count corners, not squares.

### Install the calibration tool

```bash
sudo apt install ros-jazzy-camera-calibration
```

### Run the calibration

Terminal 1: start the camera:

```bash
ros2 run camera_ros camera_node --ros-args --params-file ~/camera_ws/camera_params.yaml
```

Terminal 2: start the calibration GUI:

```bash
ros2 run camera_calibration cameracalibrator \
  --size 8x6 \
  --square 0.025 \
  --no-service-check \
  --ros-args \
  --remap image:=/camera/image_raw \
  --remap camera:=/camera
```

Set `--square` to the **real size of one square in metres**:

| Board | `--square` |
|-------|-----------|
| Printed, 25 mm squares | `0.025` |
| Shown on a phone, 9 mm squares | `0.009` |

If you use a phone or monitor, measure the displayed squares with a ruler rather than trusting the nominal size.

### Capture tips

- Move the board around the frame, including corners and edges.
- Tilt it and change its distance.
- Keep it flat and fully visible, and avoid glare.
- Wait for the progress bars (X, Y, Size, Skew) in the GUI to fill up.

### Save the results

1. Click **Calibrate** and wait for the computation to finish.
2. Click **Save**. This writes `/tmp/calibrationdata.tar.gz`.
3. Click **Commit** to send the calibration to the camera node.

Extract the saved archive:

```bash
cd /tmp
tar -xvzf calibrationdata.tar.gz
```

(`-x` extract, `-v` verbose, `-z` decompress gzip, `-f` archive filename.)

The archive contains `ost.yaml` (the calibration file you'll use), `ost.txt`, and the captured images. Copy `ost.yaml` somewhere permanent:

```bash
mkdir -p ~/camera_ws/config
cp /tmp/ost.yaml ~/camera_ws/config/ost.yaml
```

## 7. Load the calibration

Add the calibration file path to your parameter file using `camera_info_url`:

```yaml
camera:
  ros__parameters:
    camera: 0
    format: XRGB8888
    width: 640
    height: 480
    camera_info_url: file:///home/<your-username>/camera_ws/config/ost.yaml
```

Use an absolute path with the `file://` prefix. Then restart the camera node.

If you launch the camera from a launch file, pass the same parameter there:

```python
Node(
    package='camera_ros',
    executable='camera_node',
    parameters=[{
        'camera': 0,
        'width': 640,
        'height': 480,
        'camera_info_url': 'file:///home/<your-username>/camera_ws/config/ost.yaml',
    }],
)
```

### Camera name mismatch warning

When you start the node, you may see a warning like:

```
[WARN] camera calibration file has name 'narrow_stereo' but expected 'imx708__base_axi_pcie_1000120000_rp1_i2c_88000_imx708_1a'
```

Open `ost.yaml` and change the `camera_name:` field to the name given in the warning. Restart the node, and the warning should disappear.

## Troubleshooting

| Problem | What to try |
|---------|-------------|
| `cam: command not found` | Install `libcamera-tools` (step 2) or use the `cam` built in the workspace. |
| No cameras listed | Check the ribbon cable and connector, reboot, and confirm the camera is enabled for your OS. |
| Permission errors on camera devices | Add your user to the `video` group: `sudo usermod -aG video $USER`, then log out and back in. |
| Build runs out of memory | Use `colcon build --parallel-workers 1` or add swap. |
| Build fails on missing libraries | Install the libcamera build dependencies from step 1. |
| `ros-jazzy-libcamera` conflicts | Remove it as shown in step 1, then rebuild from a clean `build/`, `install/`, `log/`. |
| Camera name warning | Edit `camera_name` in `ost.yaml` (step 7). |
| Calibration GUI never detects the board | Check that `--size` counts inner corners and the board is fully visible and well lit. |

## Credits

- [camera_ros](https://github.com/christianrauch/camera_ros) by Christian Rauch
- [libcamera](https://libcamera.org/)

## License

This repository only bundles the packages listed above. Each keeps its own license (libcamera: LGPL-2.1+, camera_ros: MIT). Check the license files in each package directory.
