## 🧠 What Is an Event Group?

An **Event Group** (or **Event Flags**) is essentially a **collection of bits**, each representing a *distinct event or state flag* in your system.
You can think of it as a **bitfield shared among tasks**, managed by the kernel.

Each bit = 1 → means event occurred
Each bit = 0 → means event not yet occurred

---

### 🧩 Example Use Case (real-world style)

Let’s imagine a **Smart Refrigerator** control system (😉 relevant to your field):

We have a **Controller Task** that can only start system operation when:

1. Temperature Sensor is initialized ✅
2. Fan is ready ✅
3. Compressor is ready ✅

Each of these initializations occurs in a different task or interrupt, and once they’re all complete, the controller can proceed.

This is a *perfect* use case for **Event Groups**.

---

## ⚙️ Key FreeRTOS API Functions

| Function                 | Description                            |
| ------------------------ | -------------------------------------- |
| `xEventGroupCreate()`    | Create an event group                  |
| `xEventGroupSetBits()`   | Set bits (e.g., signal event occurred) |
| `xEventGroupClearBits()` | Clear bits (e.g., reset event flags)   |
| `xEventGroupWaitBits()`  | Wait for one or more bits to be set    |
| `xEventGroupGetBits()`   | Read current bits                      |

---

## 💡 Code Example — Multi-Event Synchronization

```c
#include "FreeRTOS.h"
#include "task.h"
#include "event_groups.h"
#include <stdio.h>

/* Event bit definitions */
#define BIT_TEMPERATURE_READY   (1 << 0)
#define BIT_FAN_READY           (1 << 1)
#define BIT_COMPRESSOR_READY    (1 << 2)

/* Event group handle */
EventGroupHandle_t xSystemEventGroup;

/* --- Controller Task --- */
void vControllerTask(void *pv)
{
    const EventBits_t xBitsToWaitFor = BIT_TEMPERATURE_READY |
                                       BIT_FAN_READY |
                                       BIT_COMPRESSOR_READY;

    printf("Controller waiting for all subsystems...\n");

    /* Wait for ALL bits to be set before proceeding */
    EventBits_t xResultBits = xEventGroupWaitBits(
        xSystemEventGroup,
        xBitsToWaitFor,
        pdFALSE,        // Don't clear bits after waiting
        pdTRUE,         // Wait for ALL bits
        portMAX_DELAY); // Wait indefinitely

    if ((xResultBits & xBitsToWaitFor) == xBitsToWaitFor)
    {
        printf("✅ All subsystems ready. Starting control loop...\n");
    }

    for (;;)
    {
        printf("Controller running system control...\n");
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

/* --- Subsystem Tasks --- */
void vTempTask(void *pv)
{
    vTaskDelay(pdMS_TO_TICKS(1000));
    printf("Temperature sensor initialized.\n");
    xEventGroupSetBits(xSystemEventGroup, BIT_TEMPERATURE_READY);
    vTaskDelete(NULL);
}

void vFanTask(void *pv)
{
    vTaskDelay(pdMS_TO_TICKS(2000));
    printf("Fan system initialized.\n");
    xEventGroupSetBits(xSystemEventGroup, BIT_FAN_READY);
    vTaskDelete(NULL);
}

void vCompressorTask(void *pv)
{
    vTaskDelay(pdMS_TO_TICKS(3000));
    printf("Compressor initialized.\n");
    xEventGroupSetBits(xSystemEventGroup, BIT_COMPRESSOR_READY);
    vTaskDelete(NULL);
}

/* --- Main --- */
int main(void)
{
    xSystemEventGroup = xEventGroupCreate();

    xTaskCreate(vControllerTask, "Controller", 1000, NULL, 2, NULL);
    xTaskCreate(vTempTask, "Temp", 1000, NULL, 1, NULL);
    xTaskCreate(vFanTask, "Fan", 1000, NULL, 1, NULL);
    xTaskCreate(vCompressorTask, "Compressor", 1000, NULL, 1, NULL);

    vTaskStartScheduler();

    for (;;);
}
```

---

### 🧾 Output

```
Controller waiting for all subsystems...
Temperature sensor initialized.
Fan system initialized.
Compressor initialized.
✅ All subsystems ready. Starting control loop...
Controller running system control...
Controller running system control...
...
```

---

## ⚙️ Notes

* You can wait for **any bit** (`pdFALSE`) or **all bits** (`pdTRUE`).
* Multiple tasks can set or wait on the same event group.
* Bits can also be cleared automatically after waiting (by setting `xClearOnExit = pdTRUE`).
* Event groups are *not interrupt-safe*. From ISRs, use **Direct-to-Task Notifications** instead.

---

## ⚙️ Scenario — “System Initialization & Ready Notification”

We’ll simulate a simplified **embedded device startup** with the following:

* `SystemControllerTask` waits for initialization events from subsystems:

  * Sensor
  * Communication
  * Display
* When *all are ready*, controller sets a **SYSTEM_READY** bit.
* Then, **UI Task** waits for this SYSTEM_READY flag to start working.

---

## 💡 Code Example

```c
#include "FreeRTOS.h"
#include "task.h"
#include "event_groups.h"
#include <stdio.h>

/* --- Event bit definitions --- */
#define BIT_SENSOR_READY       (1 << 0)
#define BIT_COMM_READY         (1 << 1)
#define BIT_DISPLAY_READY      (1 << 2)
#define BIT_SYSTEM_READY       (1 << 3)

/* --- Event group handle --- */
EventGroupHandle_t xSystemEvents;

/* --- Subsystem Tasks --- */
void vSensorTask(void *pv)
{
    vTaskDelay(pdMS_TO_TICKS(1000));  // Simulate init
    printf("Sensor initialized.\n");
    xEventGroupSetBits(xSystemEvents, BIT_SENSOR_READY);
    vTaskDelete(NULL);
}

void vCommTask(void *pv)
{
    vTaskDelay(pdMS_TO_TICKS(2000));  // Simulate init
    printf("Communication module initialized.\n");
    xEventGroupSetBits(xSystemEvents, BIT_COMM_READY);
    vTaskDelete(NULL);
}

void vDisplayTask(void *pv)
{
    vTaskDelay(pdMS_TO_TICKS(1500));  // Simulate init
    printf("Display module initialized.\n");
    xEventGroupSetBits(xSystemEvents, BIT_DISPLAY_READY);
    vTaskDelete(NULL);
}

/* --- Controller Task --- */
void vSystemControllerTask(void *pv)
{
    const EventBits_t xBitsToWaitFor = BIT_SENSOR_READY |
                                       BIT_COMM_READY |
                                       BIT_DISPLAY_READY;

    printf("System Controller waiting for all subsystems...\n");

    /* Wait until all 3 bits are set */
    xEventGroupWaitBits(
        xSystemEvents,
        xBitsToWaitFor,
        pdFALSE,   // Don’t clear bits
        pdTRUE,    // Wait for all bits
        portMAX_DELAY);

    printf("✅ All subsystems ready. System is starting...\n");

    /* Signal to other tasks that the system is fully ready */
    xEventGroupSetBits(xSystemEvents, BIT_SYSTEM_READY);

    for (;;)
    {
        printf("Controller performing system-level monitoring...\n");
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

/* --- UI Task --- */
void vUITask(void *pv)
{
    printf("UI Task waiting for SYSTEM_READY flag...\n");

    xEventGroupWaitBits(
        xSystemEvents,
        BIT_SYSTEM_READY,
        pdFALSE,   // Don't clear bit
        pdTRUE,    // Wait for all bits (just one bit here)
        portMAX_DELAY);

    printf("UI Task started. System is ready!\n");

    for (;;)
    {
        printf("UI updating display...\n");
        vTaskDelay(pdMS_TO_TICKS(1500));
    }
}

/* --- Main Function --- */
int main(void)
{
    /* Create Event Group */
    xSystemEvents = xEventGroupCreate();

    /* Create Tasks */
    xTaskCreate(vSystemControllerTask, "Controller", 1000, NULL, 3, NULL);
    xTaskCreate(vSensorTask, "Sensor", 1000, NULL, 2, NULL);
    xTaskCreate(vCommTask, "Comm", 1000, NULL, 2, NULL);
    xTaskCreate(vDisplayTask, "Display", 1000, NULL, 2, NULL);
    xTaskCreate(vUITask, "UI", 1000, NULL, 1, NULL);

    vTaskStartScheduler();

    for (;;);
}
```

---

## 🧾 Expected Output

```
System Controller waiting for all subsystems...
UI Task waiting for SYSTEM_READY flag...
Sensor initialized.
Display module initialized.
Communication module initialized.
✅ All subsystems ready. System is starting...
UI Task started. System is ready!
Controller performing system-level monitoring...
UI updating display...
Controller performing system-level monitoring...
UI updating display...
...
```

---

## 🧠 Key Takeaways

| Concept                                                                                                                          | Meaning |
| -------------------------------------------------------------------------------------------------------------------------------- | ------- |
| Event Groups can signal in **both directions** — subsystems → controller → dependents.                                           |         |
| Tasks can **wait on different subsets** of bits, allowing flexible dependency management.                                        |         |
| Useful for **initialization sequencing**, **network connection states**, or **multi-component readiness**.                       |         |
| Unlike semaphores, **bits remain set** until explicitly cleared. This makes event groups great for *persistent state signaling*. |         |

