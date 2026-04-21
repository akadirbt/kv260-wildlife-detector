# KV260 Wildlife Detector

Real-time wildlife detection on the **Xilinx/AMD Kria KV260** FPGA board using a YOLOv5s neural network accelerated by the on-chip DPU (Deep Learning Processing Unit).

**67.5 FPS · ~15 ms latency · ~10% CPU · 9 animal classes**

---

## How It Works

```
AR1335 Camera (MIPI/CSI)
        │
        ▼
AP1302 ISP  ──►  GStreamer mediasrcbin
                        │
                   videoscale (HW)
                  1920×1080 → 416×416
                        │
                        ▼
                  DPU (FPGA fabric)
                  YOLOv5s INT8 — 13 ms
                        │
                        ▼
               Decode + NMS (ARM CPU)
                        │
                        ▼
              Terminal output + FPS stats
```

1. Camera captures frames via the MIPI/CSI interface through the AP1302 ISP
2. GStreamer reads the frame directly from the ISP pipeline and scales it down to 416×416 **in hardware** — CPU never touches a 1080p frame
3. Frame is fed into the DPU, which runs YOLOv5s on the FPGA fabric (~13 ms/frame)
4. Output tensors are decoded into bounding boxes using the YOLOv5 anchor formulas
5. Non-Maximum Suppression removes duplicate boxes
6. Results are printed to terminal with per-frame timing statistics

---

## Performance

| Metric | Value |
|--------|-------|
| FPS | **67.5** (stable) |
| End-to-end latency | ~15 ms |
| Preprocess (GStreamer HW scale) | 1.4 ms |
| DPU inference | 13.0 ms |
| Postprocess + NMS | 0.3 ms |
| CPU usage | ~10–15% (ARM Cortex-A53) |

---

## Hardware

- **Board:** Xilinx/AMD Kria KV260 Vision AI Starter Kit
- **Camera:** ON Semiconductor AR1335 (13MP, 1/3.2") + AP1302 ISP — connected to **J7 (IAS0)**
- **DPU:** DPUCZDX8G_ISA1_B3136 (on-chip FPGA fabric)
- **Platform:** Vitis AI 2022.1 / Ubuntu 22.04 (PetaLinux)

---

## Detected Classes

`bear` · `coyote` · `deer` · `fox` · `possum` · `raccoon` · `skunk` · `squirrel` · `turkey`

---

## Prerequisites

- KV260 board running PetaLinux / Ubuntu 22.04
- AR1335 camera module connected to **J7 (IAS0)** — not J8
- `kv260-smartcam` overlay installed (`/lib/firmware/xilinx/kv260-smartcam/`)
- Fixed `.dtbo` (corrupted `interrupt-parent` issue — see [Deployment Problems](DEPLOYMENT_PROBLEMS.md#2))
- Docker with `xilinx/smartcam-dev:latest` image (committed after installing build tools)
- `XLNX_VART_FIRMWARE` pointing to the xclbin

---

## Quick Start

### 1. Boot and load FPGA overlay (once per boot)

```bash
sudo xmutil unloadapp 2>/dev/null || true
sleep 1
sudo xmutil loadapp kv260-smartcam
```

### 2. Configure ISP pipeline (once per boot)

```bash
sudo media-ctl -d /dev/media0 -V '"ap1302.4-003c":2 [fmt:UYVY8_1X16/1920x1080]'
sudo media-ctl -d /dev/media0 -V '"80000000.csiss":0 [fmt:UYVY8_1X16/1920x1080]'
sudo media-ctl -d /dev/media0 -V '"80000000.csiss":1 [fmt:UYVY8_1X16/1920x1080]'
sudo v4l2-ctl -d /dev/video0 --set-fmt-video=width=1920,height=1080,pixelformat=NV12
```

### 3. Start Docker container

```bash
CID=$(sudo docker run -dit --privileged --network host \
  -v /dev:/dev -v /sys:/sys -v /tmp:/tmp \
  -v /etc/vart.conf:/etc/vart.conf \
  -v /lib/firmware/xilinx:/lib/firmware/xilinx \
  -v /home/ubuntu:/home/ubuntu -v /run:/run \
  xilinx/smartcam-dev:latest bash)
```

### 4. Run the detector

```bash
sudo docker exec -it $CID bash
export XLNX_VART_FIRMWARE=/lib/firmware/xilinx/kv260-smartcam/kv260-smartcam.xclbin
/home/ubuntu/dpu_detector
```

Or use the startup script:

```bash
sudo bash /home/ubuntu/wildlife_start.sh
```

---

## Build

Inside the Docker container:

```bash
PKG_CONFIG_PATH=/usr/lib/aarch64-linux-gnu/pkgconfig \
g++ -O3 -std=c++17 -o dpu_detector dpu_detector.cpp \
  $(PKG_CONFIG_PATH=/usr/lib/aarch64-linux-gnu/pkgconfig pkg-config --cflags --libs opencv4) \
  $(PKG_CONFIG_PATH=/usr/lib/aarch64-linux-gnu/pkgconfig pkg-config --cflags --libs gstreamer-1.0) \
  -lvitis_ai_library-graph_runner -lvart-runner -lxir -lglog -lpthread
```

Build dependencies (install once, then commit the container — see [Problem #11](DEPLOYMENT_PROBLEMS.md#11)):

```bash
DEBIAN_FRONTEND=noninteractive apt-get install -y g++ libopencv-dev libgoogle-glog-dev
sudo docker commit $CID xilinx/smartcam-dev:latest
```

---

## Configuration

All tunable parameters are at the top of `dpu_detector.cpp`:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `MODEL_PATH` | `wildlife_detector.xmodel` | Path to compiled DPU model |
| `ANCHOR_PATH` | `anchor_info.json` | YOLO anchor boxes from training |
| `INPUT_W / INPUT_H` | `416` | Neural network input resolution |
| `CONF_THRESH` | `0.5` | Minimum detection confidence |
| `NMS_THRESH` | `0.6` | NMS overlap threshold |
| `NUM_CLASSES` | `9` | Number of animal classes |

To switch to a 640×640 model, change only `INPUT_W`/`INPUT_H` and provide the new `.xmodel`.

---

## Required Files on Board

| File | Description |
|------|-------------|
| `wildlife_detector.xmodel` | YOLOv5s compiled for DPU (INT8) |
| `anchor_info.json` | Anchor boxes exported from training |
| `kv260-smartcam.dtbo` | Fixed FPGA overlay |
| `kv260-smartcam.xclbin` | DPU bitstream loaded by Docker |
| `/etc/vart.conf` | Vitis AI runtime configuration |

---

## Deployment Problems

Bringup took solving 15 distinct problems spanning firmware corruption, kernel driver conflicts, DMA memory allocation, GStreamer pipeline configuration, Vitis AI runtime setup, C++ toolchain, and performance bottlenecks. Every problem, root cause, failed attempt, and solution is documented in detail:

**[→ DEPLOYMENT_PROBLEMS.md](DEPLOYMENT_PROBLEMS.md)**

Notable issues:
- Corrupted `.dtbo` with invalid `interrupt-parent = 0xffffffff` silently breaking the ISP pipeline
- `v4l2src` / `CAP_V4L2` not working — must use `mediasrcbin` for Xilinx ISP pipelines
- `frame_id` deduplication logic dropping FPS from 67 to 3.5
- Hardware `videoscale` in GStreamer pipeline being the difference between 44 and 67 FPS

---

## Next Steps

- [ ] Train at 640×640 for better small/distant animal detection (squirrel mAP: 0.687 → target 0.80+)
- [ ] Add HDMI output via `kmssink` for on-device bounding box display
- [ ] Add `media-ctl` commands to `wildlife_start.sh` for clean bringup on a fresh board
- [ ] Add `XLNX_VART_FIRMWARE` to `/etc/environment` for permanent setup
- [ ] Tune `CONF_THRESH` to ~0.344 based on F1 curve for better field recall

---

## Platform

| Component | Version |
|-----------|---------|
| Vitis AI | 2022.1 |
| PetaLinux / Ubuntu | 22.04 |
| Docker image | `xilinx/smartcam-dev:latest` |
| DPU architecture | DPUCZDX8G_ISA1_B3136 |
| Model | YOLOv5s, INT8 quantized, 416×416 |
