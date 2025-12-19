## Task Notifications

— A *lightweight, fast alternative* to queues and semaphores for communication between tasks (or between an ISR and a task).

---

### What Task Notifications Are

Every **FreeRTOS task** automatically has a built-in *notification value* (like a private integer mailbox).
This means **you don’t need to create a queue or semaphore** — the notification mechanism is already part of the TCB (Task Control Block).

You can:

* **Send a value** to another task (like posting an event or data),
* **Increment** or **set bits** atomically (depending on usage mode),
* **Block a task** until it’s notified,
* **Notify from ISR** with extremely low overhead.

Think of it as:

> “A zero-cost, zero-allocation signaling mechanism between tasks.”

---

### Common Use Cases

| Case                               | Example                                      | Mechanism                                     |
| ---------------------------------- | -------------------------------------------- | --------------------------------------------- |
| **Event signaling**                | ISR wakes a task when ADC sample ready       | `xTaskNotifyGiveFromISR()`                    |
| **Data passing (simple integer)**  | A task sends a measurement to another        | `xTaskNotify()` with `eSetValueWithOverwrite` |
| **Bitwise event flags**            | Multiple independent signals in one variable | `xTaskNotify()` with `eSetBits`               |
| **Counting semaphore replacement** | Notify counts number of produced items       | `xTaskNotifyGive()` + `ulTaskNotifyTake()`    |

---

## Practical Example — “Sensor → Logger” via Task Notifications

We’ll create:

* A **Sensor Task** that periodically produces temperature data.
* A **Logger Task** that waits for notifications to log when new data is available.

```c
#include "FreeRTOS.h"
#include "task.h"
#include <stdio.h>

TaskHandle_t xLoggerTaskHandle = NULL;
float g_temperature = 0.0f;

void vSensorTask(void *pvParameters)
{
    while (1)
    {
        // Generate new sensor data
        g_temperature = 25.0f + (rand() % 100) / 10.0f;  // 25.0–35.0°C range

        // Notify logger task that new data is available
        xTaskNotifyGive(xLoggerTaskHandle);

        printf("[Sensor] Measured: %.1f°C\n", g_temperature);

        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

void vLoggerTask(void *pvParameters)
{
    while (1)
    {
        // Block until sensor notifies
        ulTaskNotifyTake(pdTRUE, portMAX_DELAY);

        printf("→ [Logger] New temperature: %.1f°C\n", g_temperature);
    }
}

int main(void)
{
    xTaskCreate(vLoggerTask, "Logger", 1000, NULL, 2, &xLoggerTaskHandle);
    xTaskCreate(vSensorTask, "Sensor", 1000, NULL, 1, NULL);

    vTaskStartScheduler();
    return 0;
}
```

### Output:

```
[Sensor] Measured: 27.2°C
→ [Logger] New temperature: 27.2°C

[Sensor] Measured: 28.5°C
→ [Logger] New temperature: 28.5°C
```

---

### Key Concepts

* `xTaskNotifyGive()` — sends a notification to a specific task (like signaling a semaphore).
* `ulTaskNotifyTake(clearOnExit, timeout)` — waits for the notification (like taking from a semaphore).
* **Extremely lightweight:** about 20× faster than queue operations.
* **1-to-1 communication** only (one task waits for notification).

---

## Variants

| Function                                         | Purpose                                                         |
| ------------------------------------------------ | --------------------------------------------------------------- |
| `xTaskNotify()`                                  | Send notification with mode control (overwrite, set bits, etc.) |
| `xTaskNotifyFromISR()`                           | Same from an interrupt context                                  |
| `xTaskNotifyGive()` / `xTaskNotifyGiveFromISR()` | Increment a task’s notification count                           |
| `ulTaskNotifyTake()`                             | Wait and decrement the count                                    |
| `xTaskNotifyWait()`                              | Wait and optionally clear/set bits                              |

---

## Comparison vs Queue

| Feature     | Queue                               | Task Notification                     |
| ----------- | ----------------------------------- | ------------------------------------- |
| Memory      | Dynamic/static (needs allocation)   | Built-in (no extra memory)            |
| Speed       | Slower                              | Very fast                             |
| Many-to-one | ✓                                   | ❌ (1-to-1 only)                      |
| Data Size   | Structs, any type                   | Usually int/bitmask                   |
| Use Case    | Multi-producer, structured messages | ISR signals, simple data, sync events |

---

> “If a task receives multiple notifications, how does it know *which event* or *which variable/data* to process?”

Let’s unpack that — because it’s one of the **most important nuances** of using task notifications effectively.

---

## How Task Notifications Carry Information

There are actually **two categories** of FreeRTOS task notifications:

| Type                           | Description                                                                                                          | Function Pair                              |
| ------------------------------ | -------------------------------------------------------------------------------------------------------------------- | ------------------------------------------ |
| **Simple count notifications** | Used like a semaphore counter. Only increments. No extra data.                                                       | `xTaskNotifyGive()` / `ulTaskNotifyTake()` |
| **Value-based notifications**  | Used to send custom values or bitmasks. Each task has one 32-bit “notification value” you can read/write atomically. | `xTaskNotify()` / `xTaskNotifyWait()`      |

So — when you use `xTaskNotifyGive()`, the receiving task **just knows “I’ve been notified”**, not *why* or *by whom*.
That’s perfect for *single-event* synchronization (like “new sample ready” or “buffer full”).

But if you want to distinguish between **multiple events**, you should switch to **value-based notifications**, using `xTaskNotify()` and `xTaskNotifyWait()`.

---

## Example: Multiple Event Sources → One Task

Let’s imagine a **Parser Task** that may get data from:

* UART1 interrupt
* UART2 interrupt
* Timer overflow event

You can represent each as a **bit flag**:

```c
#define EVENT_UART1_RX  (1UL << 0)
#define EVENT_UART2_RX  (1UL << 1)
#define EVENT_TIMER     (1UL << 2)
```

Now, from each ISR or task, you “set” the appropriate bit using `xTaskNotify()`.

---

### Multi-Source Notifications Example

```c
#include "FreeRTOS.h"
#include "task.h"
#include <stdio.h>

TaskHandle_t xParserTaskHandle = NULL;

#define EVENT_UART1_RX  (1UL << 0)
#define EVENT_UART2_RX  (1UL << 1)
#define EVENT_TIMER     (1UL << 2)

// Simulated ISRs or event sources:
void vUART1_ISR_Sim(void)
{
    // Notify parser task: UART1 RX event
    xTaskNotify(xParserTaskHandle, EVENT_UART1_RX, eSetBits);
}

void vUART2_ISR_Sim(void)
{
    // Notify parser task: UART2 RX event
    xTaskNotify(xParserTaskHandle, EVENT_UART2_RX, eSetBits);
}

void vTimer_ISR_Sim(void)
{
    // Notify parser task: timer event
    xTaskNotify(xParserTaskHandle, EVENT_TIMER, eSetBits);
}

void vParserTask(void *pvParameters)
{
    uint32_t ulNotifiedValue;

    while (1)
    {
        // Wait for any notification bits to be set
        xTaskNotifyWait(0x00, ULONG_MAX, &ulNotifiedValue, portMAX_DELAY);

        if (ulNotifiedValue & EVENT_UART1_RX)
            printf("[Parser] Handling UART1 RX\n");

        if (ulNotifiedValue & EVENT_UART2_RX)
            printf("[Parser] Handling UART2 RX\n");

        if (ulNotifiedValue & EVENT_TIMER)
            printf("[Parser] Handling Timer Event\n");
    }
}

int main(void)
{
    xTaskCreate(vParserTask, "Parser", 1000, NULL, 2, &xParserTaskHandle);
    vTaskStartScheduler();
    return 0;
}
```

### Example Output (simulated sequence)

```
[Parser] Handling UART1 RX
[Parser] Handling UART2 RX
[Parser] Handling Timer Event
```

---

## Explanation

* Each event sets one bit in the **task’s 32-bit notification value**.
* `xTaskNotifyWait(clearOnEntryMask, clearOnExitMask, &value, timeout)` waits and retrieves these bits.
* Here, we clear all bits (`ULONG_MAX`) after reading.
* This allows multiple simultaneous events — all detected at once.

You can think of this like a **lightweight event flags group** (similar to `EventGroup` in FreeRTOS, but faster and task-specific).

---

## Choosing the Right Notification Mode

| Purpose                            | Use                                           | Example                     |
| ---------------------------------- | --------------------------------------------- | --------------------------- |
| **Single event (count/semaphore)** | `xTaskNotifyGive()` / `ulTaskNotifyTake()`    | “New sample ready”          |
| **Multiple event types (flags)**   | `xTaskNotify()` / `xTaskNotifyWait()`         | UART1, UART2, Timer         |
| **Send integer data**              | `xTaskNotify()` with `eSetValueWithOverwrite` | Send ADC value, index, etc. |

---
