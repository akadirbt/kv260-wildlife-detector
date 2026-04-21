# KV260 Wildlife Detector — Deployment Problem Report

**Platform:** Xilinx/AMD Kria KV260 · Vitis AI 2022.1 · YOLOv5s  
**Summary:** From dead camera to 67.5 FPS — 15 problems, all documented.

---

## Problem Index

| # | Problem | Category | Status |
|---|---------|----------|--------|
| 1 | Ghost Device — `/dev/video0` Does Not Exist | Firmware / FPGA | ✅ Resolved |
| 2 | Corrupted FPGA Overlay (`.dtbo`) — Invalid `interrupt-parent` | Firmware / Driver | ✅ Resolved |
| 3 | CSI Camera Entity Not Initialized on Second `loadapp` | Kernel / Driver | ✅ Resolved |
| 4 | CMA Buffer Allocation Failure — 4K Sensor vs 1080p System | Memory / DMA | ✅ Resolved |
| 5 | Broken Pipe and Unknown Format — `media-ctl` Pad Mismatch | GStreamer / Media Pipeline | ✅ Resolved |
| 6 | RTSP Stream 26-Second Lag | GStreamer / Encoder | ✅ Resolved |
| 7 | DPU Factory Empty — Runner Cannot Be Created | Vitis AI / DPU | ✅ Resolved |
| 8 | `xir::Subgraph` API Mismatch — `toposort_child_subgraph` Not Found | Vitis AI API | ✅ Resolved |
| 9 | C++ Build Dependencies Missing — `opencv4` and `glog` Not Found | Build / Toolchain | ✅ Resolved |
| 10 | `pkg-config` Cannot Find `opencv4` Inside Docker | Build / Toolchain | ✅ Resolved |
| 11 | Docker Container Loses Installed Packages After Restart | Docker / Environment | ✅ Resolved |
| 12 | `/dev/video0` Cannot Be Opened with `CAP_V4L2` or `v4l2src` | Camera / ISP Pipeline | ✅ Resolved |
| 13 | RTSP Bottleneck — FPS Capped at ~30 | Performance | ✅ Resolved |
| 14 | CPU Resize Bottleneck — FPS Capped at 44 | Performance | ✅ Resolved |
| 15 | `frame_id` Deduplication Check Drops FPS to 3.5 | Performance / Logic | ✅ Resolved |

---

## FPS Progression

| Stage | FPS | Key Change |
|-------|-----|------------|
| Camera dead (broken `.dtbo`) | **0** | No camera, no frames at all |
| Camera alive (fixed `.dtbo` + `media-ctl`) | — | Frames flowing, no DPU yet |
| Python + RTSP + DPU | **1–5** | First inference, Python overhead |
| Python DPU (fixed) | **22–30** | Docker + xclbin correct |
| C++ + RTSP | **~32** | Rewritten in C++ |
| C++ + `mediasrcbin` (RTSP bypassed) | **44** | Direct ISP read |
| C++ + GStreamer hardware scale | **67** | `videoscale` in pipeline |

---

## Problem Details

### #1 — Ghost Device: `/dev/video0` Does Not Exist
**Category:** Firmware / FPGA

**Symptom:** `/dev/video0` is not visible. `ls /dev/video*` returns nothing. The camera does not exist to the OS.

**Root Cause:** On the KV260, the AR1335 sensor and AP1302 ISP pass through programmable logic (FPGA fabric) acting as a gateway. Until the FPGA is programmed with the correct bitstream (overlay), this gateway is closed and the kernel has no visibility of any camera device.

**Attempts:**
1. Checked `dmesg` for camera — no entries at all
2. `v4l2-ctl --list-devices` — empty output
3. Rebooted — device still absent (overlay not loaded at boot)

**Solution:** Program the FPGA fabric with the `kv260-smartcam` overlay using `xmutil`. Without this step, no camera-related commands will work at all.

```bash
# Load the smartcam bitstream onto the FPGA
sudo xmutil unloadapp        # unload any existing overlay first
sudo xmutil loadapp kv260-smartcam

# Verify camera device appeared
ls /dev/video*               # should now show /dev/video0
v4l2-ctl --list-devices      # should list AP1302 ISP
```

> **NOTE:** The overlay must be loaded once per boot. It is not loaded automatically at startup. Always run `unloadapp` before `loadapp` to prevent entity conflicts.

---

### #2 — Corrupted FPGA Overlay (`.dtbo`) — Invalid `interrupt-parent`
**Category:** Firmware / Driver

**Symptom:** `dmesg` reports: `Entity type for entity 80000000.csiss was not initialized!`. Camera produces no frames. `xmutil loadapp` returns success but the ISP pipeline is non-functional.

**Root Cause:** The `kv260-smartcam.dtbo` file had a corrupted `interrupt-parent` value (`0xffffffff`) in the `csiss` node definition. This prevented the `xilinx-csi2rxss` driver from probing correctly. The overlay loaded without visible error but the camera entity was silently never initialised.

**Attempts:**
1. Manual bind of `xilinx-csi2rxss` driver → `Device or resource busy`
2. Unbind + rebind loop → `duplicate subdev` error
3. Repeated unload/reload cycles → same error
4. Found the invalid `0xffffffff` `interrupt-parent` value in `dmesg`

**Solution:** Download the correct `.dtsi` from the official Xilinx `kria-apps-firmware` GitHub (`xlnx_rel_v2022.1` branch), recompile with `dtc`, and replace the corrupted file.

```bash
# Download correct .dtsi from Xilinx GitHub
wget -O /tmp/kv260-smartcam.dtsi \
  https://raw.githubusercontent.com/Xilinx/kria-apps-firmware/\
xlnx_rel_v2022.1/boards/kv260/smartcam/kv260-smartcam.dtsi

# Compile to binary overlay
dtc -@ -O dtb -o /tmp/kv260-smartcam-new.dtbo /tmp/kv260-smartcam.dtsi

# Backup original and replace
sudo cp /lib/firmware/xilinx/kv260-smartcam/kv260-smartcam.dtbo \
        /lib/firmware/xilinx/kv260-smartcam/kv260-smartcam.dtbo.bak
sudo cp /tmp/kv260-smartcam-new.dtbo \
        /lib/firmware/xilinx/kv260-smartcam/kv260-smartcam.dtbo

sudo reboot
sudo xmutil loadapp kv260-smartcam
sudo dmesg | grep -i 'ap1302'   # should show: ap1302 detected
```

> **NOTE:** The fixed overlay only works correctly on the **first** `loadapp` after each reboot. A second unload/load cycle without rebooting will reproduce the entity error.

---

### #3 — CSI Camera Entity Not Initialized on Second `loadapp`
**Category:** Kernel / Driver

**Symptom:** After running `xmutil unloadapp && xmutil loadapp kv260-smartcam` a second time without rebooting, the command returns `Entity type not initialized`. Camera becomes inaccessible.

**Root Cause:** The Linux kernel media framework registers ISP pipeline entities the first time the overlay is loaded. These registrations persist in kernel memory even after `unloadapp`. A second `loadapp` attempts to re-register already-registered entities, which the kernel rejects as a conflict.

**Solution:** Load the overlay exactly once per boot. If the error appears, the only reliable fix is a full reboot.

```bash
# Correct procedure — once per boot only
sudo xmutil unloadapp 2>/dev/null || true
sleep 1
sudo xmutil loadapp kv260-smartcam

# If 'Entity type not initialized' appears:
sudo reboot   # only reliable fix
```

> **NOTE:** This is a known platform limitation of the KV260. There is no software workaround — a reboot is required.

---

### #4 — CMA Buffer Allocation Failure — 4K Sensor vs 1080p System
**Category:** Memory / DMA

**Symptom:** GStreamer pipeline fails with `Buffer pool activation failed` or `Failed to allocate required memory`. No frames captured.

**Root Cause:** The AR1335 is a 13MP (4K) sensor. Its DMA engine attempts to write 4K-sized contiguous memory blocks. The system was configured for 1080p and the CMA region was too small. RAM fragmentation after extended runtime prevents finding large enough contiguous blocks even when total free memory is sufficient.

**Attempts:**
1. Reducing buffer count in GStreamer → still failed
2. Tried MMAP vs USERPTR io-mode → same failure
3. `/proc/meminfo` showed CMA region was only 256MB
4. Tried MJPG format → ISP does not support MJPG output

**Solution:** Two-part fix: (1) Force all pipeline stages to 1080p using `media-ctl`. (2) Expand the CMA region to 1GB via kernel boot parameters.

```bash
# Part 1: Force pipeline to 1080p at every stage
sudo media-ctl -d /dev/media0 -V '"ap1302.4-003c":2 [fmt:UYVY8_1X16/1920x1080]'
sudo media-ctl -d /dev/media0 -V '"80000000.csiss":0 [fmt:UYVY8_1X16/1920x1080]'
sudo media-ctl -d /dev/media0 -V '"80000000.csiss":1 [fmt:UYVY8_1X16/1920x1080]'

# Part 2: Expand CMA region to 1GB
# Edit /boot/firmware/cmdline.txt — append cma=1024M to the kernel parameters line
sudo nano /boot/firmware/cmdline.txt
sudo reboot

# Verify CMA after reboot:
cat /proc/meminfo | grep -i Cma
# CmaTotal should show ~1048576 kB
```

> **NOTE:** Without `media-ctl` format locking, the ISP may attempt 4K output even if 1080p is requested through V4L2. Both fixes are needed.

---

### #5 — Broken Pipe and Unknown Format — `media-ctl` Pad Mismatch
**Category:** GStreamer / Media Pipeline

**Symptom:** `v4l2-ctl` streaming fails with `Broken pipe`. `media-ctl -p` shows `unknown` in format fields for one or more pipeline pads. No frames captured.

**Root Cause:** The ISP media pipeline has multiple pads (sensor output, ISP input, ISP output, CSI receiver input, CSI receiver output). Each pad must agree on the same pixel format and resolution. If any single pad is unset or mismatched, the pipeline cannot negotiate a stream.

**Attempts:**
1. Set only the ISP output format → CSI pads still showed `unknown`
2. Used `v4l2-ctl` to set format on `/dev/video0` → pads remained `unknown`
3. Tried GStreamer without `media-ctl` pre-configuration → `Broken pipe` immediately

**Solution:** Explicitly set the format on **every pad** in the pipeline before starting capture. All three stages must be set to the same format (`UYVY8_1X16` at `1920x1080`). Then set the V4L2 video node to `NV12`.

```bash
# Step 1: Lock ISP output pad
sudo media-ctl -d /dev/media0 -V '"ap1302.4-003c":2 [fmt:UYVY8_1X16/1920x1080]'

# Step 2: Lock CSI receiver input pad (must match ISP output)
sudo media-ctl -d /dev/media0 -V '"80000000.csiss":0 [fmt:UYVY8_1X16/1920x1080]'

# Step 3: Lock CSI receiver output pad (DMA path)
sudo media-ctl -d /dev/media0 -V '"80000000.csiss":1 [fmt:UYVY8_1X16/1920x1080]'

# Step 4: Set V4L2 capture node to NV12 (DPU-friendly)
sudo v4l2-ctl -d /dev/video0 --set-fmt-video=width=1920,height=1080,pixelformat=NV12

# Verify — no 'unknown' should remain:
media-ctl -d /dev/media0 -p

# Test streaming:
v4l2-ctl -d /dev/video0 --stream-mmap --stream-count=100
```

> **NOTE:** This manual `media-ctl` configuration is the single most critical step in the entire bringup process. Consider adding these commands to `wildlife_start.sh` for clean bringup on a fresh board.

---

### #6 — RTSP Stream 26-Second Lag
**Category:** GStreamer / Encoder

**Symptom:** RTSP stream viewed with `ffplay` has up to 26 seconds of delay. Real-time monitoring is impossible.

**Root Cause:** Default `smartcam` encoder parameters (`periodicity-idr=270`, `initial-delay=100`, `cpb-size=200`) are optimised for recording quality, not low latency. A 270-frame IDR interval means the decoder must wait up to 9 seconds for a keyframe. Combined with `ffplay`'s receive buffer, total lag reached 26 seconds.

**Attempts:**
1. UDP/RTP stream → SPS/PPS header missing
2. Custom `omxh264enc` GStreamer pipeline → SPS/PPS issues
3. `periodicity-idr=1 + cpb-size=0` → encoder crash
4. `gop-length=5` → lag stayed at 5s
5. `ffplay -max_delay 0` → lag increased

**Solution:** Optimise encoder parameters via `--encodeEnhancedParam`. Result: 26s → 0.03s lag.

```bash
# Low-latency smartcam RTSP startup
sudo docker exec -d $CID bash -c 'smartcam --mipi -t rtsp -n -p 8554 \
  -W 1920 -H 1080 -r 30 \
  --encodeEnhancedParam "periodicity-idr=3 initial-delay=0 \
  cpb-size=50 gdr-mode=disabled b-frames=0 num-slices=8"'

# Low-latency viewer (Windows)
ffplay -rtsp_transport tcp -fflags nobuffer -flags low_delay \
  -framedrop -vf setpts=0 -sync ext rtsp://192.168.1.172:8554/test
```

> **NOTE:** `cpb-size=0` causes encoder crash. Minimum usable value is 50. `num-slices=8` encodes each frame in parallel slices — critical for low latency.

---

### #7 — DPU Factory Empty — Runner Cannot Be Created
**Category:** Vitis AI / DPU

**Symptom:** Both C++ and Python fail with `Check failed: !get_factory().empty()` when creating a `vart::Runner`. DPU does not execute at all.

**Root Cause:** The xclbin firmware had not been loaded onto the DPU device. The Xilinx smartcam Docker container loads the xclbin automatically at startup. Without starting the container first, the DPU core is unprogrammed and the Vitis AI factory finds no registered runner implementations.

**Attempts:**
1. `sudo xbutil program` with xclbin manually → runner still failed
2. Added `-lvart-dpu-runner -lvart-dpu-controller` linker flags → same error
3. Used `vitis::ai::GraphRunner` API → factory still empty
4. Ran binary inside Docker → shared library version mismatch

**Solution:** Always start the Docker container before running the detector. The container loads the xclbin onto the FPGA as part of its initialisation.

```bash
# Start Docker first (this programs the xclbin onto the DPU)
CID=$(sudo docker run -dit --privileged --network host \
  -v /dev:/dev -v /sys:/sys -v /tmp:/tmp \
  -v /etc/vart.conf:/etc/vart.conf \
  -v /lib/firmware/xilinx:/lib/firmware/xilinx \
  -v /home/ubuntu:/home/ubuntu -v /run:/run \
  xilinx/smartcam-dev:latest bash)

# Set firmware path inside container
export XLNX_VART_FIRMWARE=/lib/firmware/xilinx/kv260-smartcam/kv260-smartcam.xclbin

/home/ubuntu/dpu_detector
```

> **NOTE:** To make `XLNX_VART_FIRMWARE` permanent:  
> `echo 'XLNX_VART_FIRMWARE=/lib/firmware/xilinx/kv260-smartcam/kv260-smartcam.xclbin' | sudo tee -a /etc/environment`

---

### #8 — `xir::Subgraph` API Mismatch — `toposort_child_subgraph` Not Found
**Category:** Vitis AI API

**Symptom:** Compilation fails: `struct xir::Subgraph has no member named toposort_child_subgraph`.

**Root Cause:** `toposort_child_subgraph()` appears in newer Vitis AI documentation but does not exist in the Vitis AI 2022.1 C++ API on the KV260. The method was added in a later release.

**Solution:** Use `get_children()` which is the correct method for Vitis AI 2022.1. Alternatively, use the `GraphRunner` API which abstracts away subgraph traversal entirely.

```cpp
// Wrong (newer Vitis AI API — not available on KV260 2022.1):
// root->toposort_child_subgraph();

// Correct for Vitis AI 2022.1:
root->get_children();

// Better: use GraphRunner to avoid subgraph API entirely
auto runner = vitis::ai::GraphRunner::create_graph_runner(graph.get(), attrs.get());
```

---

### #9 — C++ Build Dependencies Missing — `opencv4` and `glog` Not Found
**Category:** Build / Toolchain

**Symptom:** Compilation fails: `fatal error: glog/logging.h: No such file or directory`. `pkg-config` cannot find `opencv4`.

**Root Cause:** The base Ubuntu 22.04 system and the base Xilinx Docker image include only runtime libraries, not development headers. The `-dev` packages are not installed by default.

**Solution:** Install the `-dev` packages which include all headers and `pkg-config` files needed for compilation.

```bash
sudo apt-get update
DEBIAN_FRONTEND=noninteractive apt-get install -y \
  g++ libopencv-dev libgoogle-glog-dev
```

---

### #10 — `pkg-config` Cannot Find `opencv4` Inside Docker
**Category:** Build / Toolchain

**Symptom:** Even after installing `libopencv-dev`, compilation fails: `Package opencv4 was not found in the pkg-config search path`.

**Root Cause:** On arm64 (aarch64), the `opencv4.pc` file is installed at `/usr/lib/aarch64-linux-gnu/pkgconfig` — not in the default `PKG_CONFIG_PATH` inside Docker.

**Solution:** Explicitly pass `PKG_CONFIG_PATH` including the aarch64 directory.

```bash
PKG_CONFIG_PATH=/usr/lib/aarch64-linux-gnu/pkgconfig:/usr/share/pkgconfig \
g++ -O3 -std=c++17 -o /home/ubuntu/dpu_detector /home/ubuntu/dpu_detector.cpp \
  $(PKG_CONFIG_PATH=/usr/lib/aarch64-linux-gnu/pkgconfig pkg-config --cflags --libs opencv4) \
  $(PKG_CONFIG_PATH=/usr/lib/aarch64-linux-gnu/pkgconfig pkg-config --cflags --libs gstreamer-1.0) \
  -lvitis_ai_library-graph_runner -lvart-runner -lxir -lglog -lpthread
```

---

### #11 — Docker Container Loses Installed Packages After Restart
**Category:** Docker / Environment

**Symptom:** After starting a fresh Docker container, `g++` and `libopencv-dev` are missing. Every new container requires reinstalling all build tools.

**Root Cause:** Docker containers are stateless. Packages installed with `apt-get` inside a running container are lost when the container is removed. The base `xilinx/smartcam:2022.1` image does not include C++ build tools.

**Solution:** After installing packages, commit the container state as a new Docker image.

```bash
# Install packages inside running container
sudo docker exec $CID bash -c \
  'DEBIAN_FRONTEND=noninteractive apt-get update && \
   apt-get install -y g++ libopencv-dev libgoogle-glog-dev'

# Commit as new reusable image
sudo docker commit $CID xilinx/smartcam-dev:latest

# Use new image from now on (not xilinx/smartcam:2022.1)
```

---

### #12 — `/dev/video0` Cannot Be Opened with `CAP_V4L2` or `v4l2src`
**Category:** Camera / ISP Pipeline

**Symptom:** `cap_.open("/dev/video0", cv::CAP_V4L2)` returns false. GStreamer `v4l2src` reports `Buffer pool activation failed`. No frames captured.

**Root Cause:** `/dev/video0` is the output of a complex ISP media pipeline. It cannot be opened with plain V4L2 calls because the full pipeline (sensor → ISP → scaler → output) must be running first. Plain `v4l2src` and `cv::CAP_V4L2` do not configure this pipeline.

**Attempts:**
1. `cv::CAP_V4L2` direct open → returns false
2. GStreamer `v4l2src` → buffer pool activation failed
3. Tried YUYV, NV12, MJPG formats → same failure
4. Raw `v4l2` streaming works only after `media-ctl` configuration

**Solution:** Use GStreamer `mediasrcbin` which understands the Xilinx ISP pipeline and configures all entities automatically. Must be run inside the Xilinx Docker container.

```cpp
// Wrong:
// cap_.open("/dev/video0", cv::CAP_V4L2);

// Correct — mediasrcbin inside Docker:
std::string pipeline =
    "mediasrcbin media-device=/dev/media0"
    " ! video/x-raw,format=NV12,width=1920,height=1080"
    " ! videoscale"
    " ! video/x-raw,format=NV12,width=416,height=416"
    " ! videoconvert ! video/x-raw,format=BGR"
    " ! appsink max-buffers=1 drop=true sync=false";
cap_.open(pipeline, cv::CAP_GSTREAMER);
```

> **NOTE:** `mediasrcbin` is a Xilinx-specific GStreamer plugin available only inside the Docker container. The detector must run inside Docker.

---

### #13 — RTSP Bottleneck — FPS Capped at ~30
**Category:** Performance

**Symptom:** DPU takes only 13 ms per frame (theoretical 77 FPS) but actual throughput is ~30 FPS. FPS matches RTSP stream rate exactly.

**Root Cause:** Pipeline was: Camera → H.264 encode → RTSP → TCP → decode → OpenCV → DPU. The encode/decode round-trip introduced ~20 ms overhead and hard-capped throughput at the RTSP framerate.

**Solution:** Bypass RTSP entirely. Read frames directly from the ISP using `mediasrcbin` inside Docker.

```bash
# Kill RTSP process to free the camera device
sudo docker exec $CID pkill -f smartcam
sleep 2
# Detector now reads directly from ISP — no RTSP overhead
# Result: ~30 FPS → 44 FPS
```

---

### #14 — CPU Resize Bottleneck — FPS Capped at 44
**Category:** Performance

**Symptom:** After RTSP bypass, FPS stalls at 44. Timing: `pre=5.6 ms, dpu=13.0 ms, post=0.3 ms`. Preprocess is the new ceiling.

**Root Cause:** `cv::resize()` on the ARM CPU scales 1920×1080 → 416×416 every frame. On ARM Cortex-A53 this takes ~5.5 ms, making total pipeline ~19 ms instead of the target 14.7 ms.

**Attempts:**
1. `cv::INTER_LINEAR` → `cv::INTER_NEAREST` → 5.6 ms → 5.1 ms, marginal gain
2. Considered NEON SIMD intrinsics — complex and fragile

**Solution:** Move resize into the GStreamer pipeline using `videoscale`. Runs through hardware-accelerated media paths: 5.6 ms → 1.4 ms.

```cpp
"mediasrcbin media-device=/dev/media0"
" ! video/x-raw,format=NV12,width=1920,height=1080"
" ! videoscale"
" ! video/x-raw,format=NV12,width=416,height=416"  // scaled in hardware here
" ! videoconvert ! video/x-raw,format=BGR"
" ! appsink max-buffers=1 drop=true sync=false"
// Result: 44 FPS → 67 FPS
```

---

### #15 — `frame_id` Deduplication Check Drops FPS to 3.5
**Category:** Performance / Logic

**Symptom:** After adding `frame_id` duplicate detection, FPS drops from 67 to 3.5. Timing still shows 14.7 ms total — DPU is idle most of the time.

**Root Cause:** The inference loop runs at 67 FPS but the camera delivers new frames at ~30 FPS. The `frame_id` check skips inference when no new frame arrived, throttling the DPU to the camera's actual delivery rate (~3.5 FPS in practice due to thread scheduling).

**Solution:** Remove the `frame_id` equality check. Process every frame including duplicates. Redundant inference is harmless for wildlife detection and keeps the DPU fully utilised.

```cpp
// Removed — caused 3.5 FPS:
// if (current_frame_id == last_frame_id) {
//     std::this_thread::sleep_for(std::chrono::milliseconds(1));
//     continue;
// }
// Result: 3.5 FPS → 67 FPS restored
```

---

## Final System State

| Parameter | Value |
|-----------|-------|
| FPS | **67.5 FPS** (stable) |
| End-to-end latency | ~15 ms |
| Preprocess | 1.4 ms (GStreamer hardware scale) |
| DPU inference | 13.0 ms (FPGA, YOLOv5s INT8) |
| Postprocess / NMS | 0.3 ms |
| CPU usage | ~10–15% (ARM Cortex-A53) |
| Camera | AR1335 + AP1302 ISP, 1920×1080 NV12 |
| DPU input | 416×416 BGR (scaled in GStreamer) |
| Docker image | `xilinx/smartcam-dev:latest` |
| Startup | `sudo bash /home/ubuntu/wildlife_start.sh` |

---

## Important Files

| File Path | Description |
|-----------|-------------|
| `/home/ubuntu/wildlife_detector.xmodel` | YOLOv5s DPU model (416×416, INT8) |
| `/home/ubuntu/anchor_info.json` | YOLO anchor boxes from training |
| `/home/ubuntu/dpu_detector` | C++ detector binary |
| `/home/ubuntu/dpu_detector.cpp` | C++ detector source code |
| `/home/ubuntu/wildlife_start.sh` | Single-command startup script |
| `/lib/firmware/xilinx/kv260-smartcam/kv260-smartcam.dtbo` | Fixed FPGA overlay (from Xilinx GitHub) |
| `/lib/firmware/xilinx/kv260-smartcam/kv260-smartcam.dtbo.bak` | Backup of original corrupted overlay |
| `/lib/firmware/xilinx/kv260-smartcam/kv260-smartcam.xclbin` | DPU xclbin loaded by Docker |
| `/etc/vart.conf` | Vitis AI runtime config |

---

## Next Steps

- Train YOLOv5s at 640×640 with additional small/distant animal images to improve squirrel and possum mAP (currently 0.687 and 0.661)
- Re-quantize and compile the 640×640 model with Vitis AI toolchain → new `.xmodel`. Change only `INPUT_W`/`INPUT_H` in `dpu_detector.cpp`
- Add HDMI output using `kmssink` GStreamer element to display bounding boxes on a local monitor (estimated FPS impact: -5 to -8 FPS)
- Add `media-ctl` pipeline configuration commands to `wildlife_start.sh` for clean bringup on a fresh board
- Add `XLNX_VART_FIRMWARE` to `/etc/environment` to make it permanent across reboots
- Adjust `CONF_THRESH` from 0.5 to ~0.344 based on F1 curve analysis for better recall at field conditions

---

## References

| # | Document | URL | Relevant To |
|---|----------|-----|-------------|
| 1 | KV260 SmartCam App Deployment (2022.1) | xilinx.github.io/kria-apps-docs/kv260/2022.1/build/html/docs/smartcamera/docs/app_deployment.html | #1, #2, #6 |
| 2 | KV260 SmartCam Known Issues (2022.1) | xilinx.github.io/kria-apps-docs/kv260/2022.1/build/html/docs/smartcamera/docs/issue-sc.html | #2, #7 |
| 3 | KV260 SmartCam Software Architecture — Platform | xilinx.github.io/kria-apps-docs/kv260/2022.1/build/html/docs/smartcamera/docs/sw_arch_platform.html | #3, #5, #9 |
| 4 | KV260 SmartCam Software Architecture — Accelerator | xilinx.github.io/kria-apps-docs/kv260/2022.1/build/html/docs/smartcamera/docs/sw_arch_accel.html | #9, #11 |
| 5 | Xilinx kria-apps-firmware GitHub (xlnx_rel_v2022.1) | github.com/Xilinx/kria-apps-firmware/tree/xlnx_rel_v2022.1 | #2 |
| 6 | Generation of Firmware Binaries — Kria KV260 | xilinx.github.io/kria-apps-docs/kv260/2022.1/build/html/docs/generating_custom_firmware.html | #2 |
| 7 | DTSI/DTBO Generation Example for SmartCam | xilinx.github.io/kria-apps-docs/creating_applications/2022.1/build/html/docs/dtsi_dtbo_generation_smartcam_example.html | #2 |
| 8 | Vitis AI — XLNX_VART_FIRMWARE | xilinx.github.io/inference-server/main/backends/vitis_ai.html | #7 |
| 9 | Kria SOM AI Customization — DPU and xclbin | xilinx.github.io/kria-apps-docs/creating_applications/2022.1/build/html/docs/AI_customization.html | #7 |
| 10 | Running SmartCam on KV260 Board | xilinx.github.io/kria-apps-docs/creating_applications/2022.1/build/html/docs/kria_vitis_acceleration_flow/running-smartcam-on-board.html | #1, #7 |
| 11 | Linux Kernel CMA — A Deep Dive (LWN.net) | lwn.net/Articles/486301 | #4 |
| 12 | CMA for Embedded Camera/VPU Drivers (Toradex) | developer.toradex.com/software/linux-resources/linux-features/contiguous-memory-allocator-cma-linux/ | #4 |

> Problems #5, #13, #14, #15 were solved through empirical debugging on hardware. No single Xilinx document covers these specific failure modes — they are documented here for the first time in the context of a custom DPU inference pipeline on KV260.
