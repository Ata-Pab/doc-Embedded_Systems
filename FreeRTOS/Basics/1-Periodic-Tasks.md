# Periodic Tasks Demo

We’ll use:

* Two tasks: `FastTask` and `SlowTask`
* `vTaskDelayUntil()` → the correct way to run a periodic task
* Different execution periods (100 ms and 500 ms)

---

### Example Code (`main.c`)

```c
#include "FreeRTOS.h"
#include "task.h"
#include <stdio.h>

/*-----------------------------------------------------------*/
/* Task function prototypes */
void vFastTask(void *pvParameters);
void vSlowTask(void *pvParameters);

/*-----------------------------------------------------------*/
int main(void)
{
    printf("FreeRTOS Periodic Tasks Demo starting...\n");

    /* Create two tasks with different periods */
    xTaskCreate(vFastTask, "Fast Task", configMINIMAL_STACK_SIZE, NULL, 2, NULL);
    xTaskCreate(vSlowTask, "Slow Task", configMINIMAL_STACK_SIZE, NULL, 1, NULL);

    /* Start the scheduler */
    vTaskStartScheduler();

    /* We should never reach here */
    for(;;);
}

/*-----------------------------------------------------------*/
/* Fast Task: runs every 100ms */
void vFastTask(void *pvParameters)
{
    TickType_t xLastWakeTime;
    const TickType_t xPeriod = pdMS_TO_TICKS(100);

    /* Initialize the xLastWakeTime variable with the current tick count. */
    xLastWakeTime = xTaskGetTickCount();

    for( ;; )
    {
        printf("[FAST] Running every 100 ms\n");

        /* Wait for the next cycle */
        vTaskDelayUntil(&xLastWakeTime, xPeriod);
    }
}

/*-----------------------------------------------------------*/
/* Slow Task: runs every 500ms */
void vSlowTask(void *pvParameters)
{
    TickType_t xLastWakeTime;
    const TickType_t xPeriod = pdMS_TO_TICKS(500);

    xLastWakeTime = xTaskGetTickCount();

    for( ;; )
    {
        printf("      [SLOW] Running every 500 ms\n");

        vTaskDelayUntil(&xLastWakeTime, xPeriod);
    }
}
/*-----------------------------------------------------------*/

/* Hook stubs for Win32 port (you already added these earlier) */
void vConfigureTimerForRunTimeStats(void) {}
unsigned long ulGetRunTimeCounterValue(void) { return 0; }
void vAssertCalled(const char *file, int line)
{
    printf("ASSERT FAILED! File: %s, Line: %d\n", file, line);
    taskDISABLE_INTERRUPTS();
    for( ;; );
}
void vApplicationMallocFailedHook(void)
{
    printf("Malloc failed!\n");
    taskDISABLE_INTERRUPTS();
    for( ;; );
}
void vApplicationGetIdleTaskMemory( StaticTask_t **ppxIdleTaskTCBBuffer,
                                   StackType_t **ppxIdleTaskStackBuffer,
                                   uint32_t *pulIdleTaskStackSize )
{
    static StaticTask_t xIdleTaskTCB;
    static StackType_t uxIdleTaskStack[ configMINIMAL_STACK_SIZE ];
    *ppxIdleTaskTCBBuffer = &xIdleTaskTCB;
    *ppxIdleTaskStackBuffer = uxIdleTaskStack;
    *pulIdleTaskStackSize = configMINIMAL_STACK_SIZE;
}
void vApplicationGetTimerTaskMemory( StaticTask_t **ppxTimerTaskTCBBuffer,
                                    StackType_t **ppxTimerTaskStackBuffer,
                                    uint32_t *pulTimerTaskStackSize )
{
    static StaticTask_t xTimerTaskTCB;
    static StackType_t uxTimerTaskStack[ configTIMER_TASK_STACK_DEPTH ];
    *ppxTimerTaskTCBBuffer = &xTimerTaskTCB;
    *ppxTimerTaskStackBuffer = uxTimerTaskStack;
    *pulTimerTaskStackSize = configTIMER_TASK_STACK_DEPTH;
}
```

---

### Key Concept — `vTaskDelay()` vs `vTaskDelayUntil()`

| Function                                  | Use Case                                      | Timing Accuracy          |
| ----------------------------------------- | --------------------------------------------- | ------------------------ |
| `vTaskDelay(xTicks)`                      | Simple delay (relative to *now*)              | May drift over time      |
| `vTaskDelayUntil(&xLastWakeTime, xTicks)` | Periodic tasks (based on last scheduled time) | Constant, precise period |

> Example:
> If your task should run every 100 ms, and it takes 10 ms to execute, `vTaskDelayUntil()` ensures it *still waits exactly 90 ms* before the next cycle — no drift.

---

### Console Output

```
FreeRTOS Periodic Tasks Demo starting...
[FAST] Running every 100 ms
      [SLOW] Running every 500 ms
[FAST] Running every 100 ms
[FAST] Running every 100 ms
[FAST] Running every 100 ms
      [SLOW] Running every 500 ms
...
```

Notice how the **fast task prints 5 times for every slow task print** — this shows that the FreeRTOS tick scheduler is controlling timing precisely.

---

### Try Experiments

Once it runs:

1. Change the tick rate in `FreeRTOSConfig.h`

   ```
   #define configTICK_RATE_HZ 1000
   ```

   (That means 1 tick = 1 ms)
2. Add an artificial delay in one task (e.g., a `Sleep(50)` inside the fast task)
   → observe scheduling impact.

---

### Concept Takeaway

* Every RTOS task is *just a function with a loop*.
* The scheduler runs each task based on **priority + timing**.
* Periodic timing in real systems (e.g. fan control, temperature regulation) is done this way — using `vTaskDelayUntil()`.

---

## Both are “delay” functions — but they work **very differently**

| Function                | Description                                                                                          | Use Case                              |
| ----------------------- | ---------------------------------------------------------------------------------------------------- | ------------------------------------- |
| **`vTaskDelay()`**      | Delays a task for **a relative time** from **now**                                                   | “Sleep for X milliseconds”            |
| **`vTaskDelayUntil()`** | Delays a task **until a specific absolute tick count**, keeping a **fixed period** between task runs | “Run every X milliseconds (periodic)” |

---

## `vTaskDelay()`: **Relative delay**

```c
void vTaskDelay(const TickType_t xTicksToDelay);
```

* It blocks the calling task for *xTicksToDelay* ticks.
* The **next wake-up time depends on when you call it**.
* If your task takes variable time to execute before the delay, the period will drift.

**Example:**

```c
void vTaskFanControl(void *pvParams)
{
    for (;;)
    {
        ControlFreezerFan();       // Takes ~20ms
        vTaskDelay(pdMS_TO_TICKS(100)); // Sleep for 100ms
    }
}
```

Here, each loop actually takes **120ms total** (≈20ms work + 100ms delay).
If the work time changes, the total period changes → **non-deterministic** loop timing.

**Pros**:

* Simple
* Good for one-shot or event-driven tasks

**Cons**:

* Not good for **precise periodic** tasks

---

## `vTaskDelayUntil()`: **Absolute periodic delay**

```c
void vTaskDelayUntil(TickType_t *pxPreviousWakeTime, const TickType_t xTimeIncrement);
```

* You maintain a **reference tick count** (`xLastWakeTime`).
* The scheduler wakes up the task at a **fixed period** (xTimeIncrement) from the previous wake time, not from now.
* Keeps timing **consistent**, even if execution time varies.

**Example (correct periodic control loop):**

```c
void vTaskCompressorControl(void *pvParams)
{
    TickType_t xLastWakeTime = xTaskGetTickCount();

    for (;;)
    {
        ControlCompressor();  // maybe takes 10-30ms
        vTaskDelayUntil(&xLastWakeTime, pdMS_TO_TICKS(100)); // run every 100ms exactly
    }
}
```

Now, even if `ControlCompressor()` takes a variable amount of time (say 10ms or 25ms),
the task **still runs every 100ms**, as measured in system ticks — **no drift**.

**Pros**:

* Precise periodic timing
* Ideal for control loops, sensor sampling, and PID updates

**Cons**:

* Slightly more code
* If task overruns (takes longer than period), it will wake up immediately next time (no catch-up mechanism)

---

## Visual Comparison

**Using `vTaskDelay(100)`**

```
|---work (20ms)---|---delay(100ms)---|---work(25ms)---|---delay(100ms)---|
                    ↑ drift grows if work time changes
```

**Using `vTaskDelayUntil(&lastWake, 100)`**

```
|---work (20ms)---|--delay(80ms)--|---work (25ms)---|--delay(75ms)--|
Fixed period = 100ms
```

---

## Rule of thumb

| Situation                                    | Recommended Function                                         |
| -------------------------------------------- | ------------------------------------------------------------ |
| Periodic control loop                        | `vTaskDelayUntil()`                                        |
| Non-periodic background or event-driven task | `vTaskDelay()`                                             |
| Wait for next message/event indefinitely     | `xQueueReceive()` or `ulTaskNotifyTake()` (instead of delay) |

---

## Summary

| Feature         | `vTaskDelay()`              | `vTaskDelayUntil()`                     |
| --------------- | --------------------------- | --------------------------------------- |
| Delay Type      | Relative                    | Absolute / periodic                     |
| Timing accuracy | Variable                    | Fixed-period                            |
| Good for        | Background, event, one-shot | Control loops, sampling, periodic tasks |
| Risk            | Timing drift                | Possible overrun if work > period       |

---
