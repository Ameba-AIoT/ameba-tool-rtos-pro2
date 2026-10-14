# Enabling HDR Mode on AmebaPro2

## Table of Contents
1. [IQ Binary File Overview](#iq-binary-file-overview)
2. [Enable HDR Mode](#enable-hdr-mode)
   - [Step 1: Check Sensor Support](#step-1-check-sensor-support)
   - [Step 2: Set Sensor Max FPS](#step-2-set-sensor-max-fps)
   - [Step 3: Check IQ Min FPS](#step-3-check-iq-min-fps)
   - [Step 4: Enable HDR in the Example](#step-4-enable-hdr-in-the-example)
   - [Step 5: Build and Flash](#step-5-build-and-flash)
   - [Step 6: Verify](#step-6-verify)
3. [Troubleshooting](#troubleshooting)

---

# IQ Binary File Overview
On AmebaPro2, Image Quality (**IQ**) parameters are stored in a standalone binary file called `iq_<sensor_id>.bin`. IQ parameter profiles are categorized into two types:

- **Complete IQ (Full IQ):** A complete package containing multiple IQ parameter sets for different exposure modes and operating conditions.
- **Partial IQ:** An individual IQ parameter set optimized for a specific exposure mode and operating condition, such as Linear Day or HDR Night.

## IQ Table Index Mapping

A Complete IQ file can contain up to **six Partial IQ tables**, indexed from **0 to 5**. The tables are divided into two groups according to the exposure mode:

| Index | Group | Exposure Mode | Scene Mode | Usage |
|---:|---|---|---|---|
| **0** | Group 0 | Linear | Day | Default IQ table for Linear mode under daylight conditions |
| **1** | Group 0 | Linear | Night | IQ table used when Linear mode switches to night conditions |
| **2** | Group 0 | Linear | Other | Reserved or user-defined IQ table for other Linear-mode scenarios |
| **3** | Group 1 | HDR | Day | Default IQ table for HDR mode under daylight conditions |
| **4** | Group 1 | HDR | Night | IQ table used when HDR mode switches to night conditions |
| **5** | Group 1 | HDR | Other | Reserved or user-defined IQ table for other HDR-mode scenarios |

Note: Index for each table could be adjustable. But in general case, we suggest user to use setting listed as above.

---

# Enable HDR Mode

> **Example used:** `media_uvcd` (source: [`component/example/media_uvcd/example_media_uvc.c`]((https://github.com/Ameba-AIoT/ameba-rtos-pro2/blob/main/component/example/media_uvcd/example_media_uvc.c))).

This example is used with the [RealCam IQ Tuning Tool](https://github.com/Ameba-AIoT/ameba-tool-rtos-pro2/tree/main/IQ%20Tuning).

**Reference sensor:** GC4663

## Step 1: Check Sensor Support
Open [`project/realtek_amebapro2_v0_example/inc/sensor.h`](https://github.com/Ameba-AIoT/ameba-rtos-pro2/blob/main/project/realtek_amebapro2_v0_example/inc/sensor.h) and confirm your sensor is marked `v` in the **ISP HDR** column.

Select the sensor with macro, e.g. `<USE_SENSOR = SENSOR_GC4663>`.

## Step 2: Set Sensor Max FPS
In `sensor_params_t sensor_params[]` (in `sensor.h`), change the sensor's FPS
to its **HDR mode** limit, as noted in the comment next to the entry:

| Resolution | HDR Max FPS |
|---|---|
| 2K (2560*1440) | 20 |
| FHD (1920*1080) | 30 |

for example, when using GC4663
```c
// Before (Linear)
[SENSOR_GC4663] = {2560, 1440, 24},
// After (HDR, 2K)
[SENSOR_GC4663] = {2560, 1440, 20},
```

## Step 3: Check IQ Min FPS
The sensor HDR max FPS (Step 2) **must be greater than or equal to** the
`dyn_min_fps` of **both** of these IQ tables:

| IQ Index | Mode | Requirement |
|---|---|---|
| 0 | Linear Day | `dyn_min_fps` <= HDR max FPS |
| 3 | HDR Day | `dyn_min_fps` <= HDR max FPS |

Note: the ISP loads index 0 during init before switching to HDR

You can check `dyn_min_fps` in RealCam ISP Tuning Pro under `AE \ AE Attribute Extension`.

The SDK default `iq_gc4663.bin` already meets this requirement.
Use it for your first test before trying a custom IQ.

## Step 4: Enable HDR
In [`video_api.c`](https://github.com/Ameba-AIoT/ameba-rtos-pro2/blob/main/component/video/driver/RTL8735B/video_api.c), set `init_items.init_hdr_mode` value to 1:

```c
init_items.init_hdr_mode = 1; // 0: Linear, 1: HDR
```
## Step 5: Build and Flash
Run these commands from `project/realtek_amebapro2_v0_example/GCC-RELEASE/build`:

```bash
cmake .. -G"Unix Makefiles" -DCMAKE_TOOLCHAIN_FILE=../toolchain.cmake -DEXAMPLE=media_uvcd
cmake --build . --target flash -j12
```

Flash `flash_ntz.bin` to the board using `uartfwburn` image tool.

## Step 6: Verify
Check the UART log after boot:

1. **IQ timestamp:** this confirms the correct IQ bin is built in.
   The SDK default GC4663 IQ shows:
   ```text
   iq timestamp: 2026/09/21 17:34:49
   ```
2. **HDR active:** look for `[VOE]change sensor mode => 2560x1440@20.000000fps - 'hdr line 2to1' `.
3. **Stream:** open RealCam and confirm the video stream is visible.

---
    
# Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| No stream in RealCam after enabling HDR | Sensor max FPS exceeds the HDR limit | Recheck Step 2 |
| No stream in RealCam after enabling HDR | `dyn_min_fps` of IQ index 0 or 3 is greater than sensor max FPS | Recheck Step 3 |
| IQ timestamp does not match | Wrong or old IQ bin built in | Clean the build folder and rebuild, try to perform chip erase if the issue remains |
