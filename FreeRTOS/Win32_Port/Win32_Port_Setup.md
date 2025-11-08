## 🧩 Step 1 — Clone Correctly (with Kernel)

Run this command from **PowerShell** or **Command Prompt**:

```bash
git clone --recurse-submodules https://github.com/FreeRTOS/FreeRTOS.git
```

That `--recurse-submodules` flag is the key — it automatically pulls in:

```
FreeRTOS-Kernel (Source/)
FreeRTOS-Plus
FreeRTOS-Test
```

---

## ✅ Step 2 — Verify Folder Structure

After cloning, you should have:

```
FreeRTOS/
 ├── FreeRTOS-Plus/
 ├── FreeRTOS-Kernel/
 │    ├── include/
 │    ├── portable/
 │    ├── tasks.c
 │    ├── queue.c
 │    ├── timers.c
 │    └── ...
 ├── Demo/
 │    └── WIN32-MSVC/
 │         └── WIN32.sln
 └── LICENSE.md
```

> 🧠 Note: Newer FreeRTOS versions renamed `Source/` → `FreeRTOS-Kernel/`.

If you still see a `Source/` reference inside `Demo/WIN32-MSVC`, don’t worry — we’ll adjust include paths to the new folder name.

---

## ⚙️ Step 3 — Fix Include Paths for New Structure

In **Visual Studio**, open:

```
FreeRTOS\Demo\WIN32-MSVC\WIN32.sln
```

Then right-click the `RTOSDemo` project → **Properties** →
`Configuration Properties → C/C++ → General → Additional Include Directories`

Replace with:

```
..\..\FreeRTOS-Kernel\include;
..\..\FreeRTOS-Kernel\portable\MSVC-MingW;
..\Common\include;
.
```

---

## ⚙️ Step 4 — Verify & Build

1. Set **Configuration = Debug**, **Platform = Win32**
2. Build → Clean Solution
3. Build → Rebuild Solution

If it builds successfully, you’ll see something like:

```
1>------ Build started: Project: RTOSDemo, Configuration: Debug Win32 ------
1>Compiling...
1>main_blinky.c
1>Linking...
1>Build succeeded.
```

Then press **F5** to run the Blinky demo.

---

## 🧠 Optional: If You Already Have a Broken Clone

If you don’t want to re-clone everything, you can reinitialize submodules manually:

```bash
cd C:\Users\atalayp\Downloads\FreeRTOS
git submodule deinit -f --all
git submodule update --init --recursive
```

But honestly, a fresh clone with `--recurse-submodules` is the cleanest and fastest fix.

---

Once you confirm your build works, we’ll continue with:
✅ Analyzing `main_blinky.c` → how tasks are created and how the scheduler starts.

Would you like me to walk through what happens inside `main_blinky.c` once it starts running (line by line, conceptually)?

---

### ✅ Recommended Repos

| # | Project                                                            | Description                                                                                 | Why it’s worth investigating                                       |
| - | ------------------------------------------------------------------ | ------------------------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| 1 | iot‑reference‑stm32u5 (by FreeRTOS) ([GitHub][1])                  | Runs on STM32 U5 board; cloud-connected, using FreeRTOS kernel + libraries.                 | Full stack: RTOS + IoT + secure update. Good for advanced use.     |
| 2 | X‑CUBE‑FREERTOS (by ST) ([GitHub][2])                              | Official STM32Cube expansion for FreeRTOS. Contains many ready-to-run applicative examples. | Professional codebase, many boards supported. Great structure.     |
| 3 | FreeRTOS‑STM32‑HAL‑Examples by “kowalski100” ([GitHub][3])         | Example tutorials on STM32F4 Discovery board using HAL + FreeRTOS.                          | Simpler than full IoT stacks — nice for focusing on RTOS patterns. |
| 4 | FreeRTOS_Gesture_Data_Collection (DevHeadsCommunity) ([GitHub][4]) | Uses an MPU-6050 sensor + STM32, FreeRTOS to orchestrate data capture & processing.         | Sensor loop + RTOS tasks = real embedded pattern you can adapt.    |
| 5 | STM32WBA‑BLE‑HeartRate‑FreeRTOS (stm32-hotspot org) ([GitHub][5])  | BLE heart-rate demo on STM32WBA with FreeRTOS + Cube examples.                              | Wireless + RTOS example — good for connectivity + tasks interplay. |

[1]: https://github.com/FreeRTOS/iot-reference-stm32u5?utm_source=chatgpt.com "FreeRTOS/iot-reference-stm32u5 - GitHub"
[2]: https://github.com/STMicroelectronics/x-cube-freertos?utm_source=chatgpt.com "STMicroelectronics/x-cube-freertos - GitHub"
[3]: https://github.com/kowalski100/FreeRTOS-STM32-HAL-Examples?utm_source=chatgpt.com "kowalski100/FreeRTOS-STM32-HAL-Examples - GitHub"
[4]: https://github.com/DevHeadsCommunity/FreeRTOS_Gesture_Data_Collection/blob/master/README.md?utm_source=chatgpt.com "Gesture Data Collection with MPU-6050 and STM32 using FreeRTOS"
[5]: https://github.com/stm32-hotspot/STM32WBA-BLE-HeartRate-FreeRTOS?utm_source=chatgpt.com "stm32-hotspot/STM32WBA-BLE-HeartRate-FreeRTOS - GitHub"
