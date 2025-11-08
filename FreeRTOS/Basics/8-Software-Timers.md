## 🧠 Software Timers and Deferred Execution

This topic builds on what you already know — but takes a different angle.

While tasks can use `vTaskDelay()` or `vTaskDelayUntil()` for periodic behavior, **Software Timers** are:

* Lightweight kernel-managed timers that **don’t require a dedicated task**.
* Ideal for **deferred execution**, **timeouts**, or **non-critical periodic functions**.
* Often used instead of spawning a new task (saving stack memory and context-switch overhead).

---

### When to Use Software Timers

| Use Case                      | Why Timers Are Ideal                            |
| ----------------------------- | ----------------------------------------------- |
| Sensor polling every 1 second | Avoids a task stuck in delay loop               |
| Watchdog refresh              | Centralized periodic function                   |
| Communication timeout         | Callback runs only if event not completed       |
| LED blinking                  | No need for full task; just toggle periodically |

---

### 🔧 How They Work

FreeRTOS has a **Timer Service Task** (created automatically when you use any timer).
Each timer has:

* A **period** (time interval)
* A **callback function** (executed when the timer expires)
* A **mode**: one-shot or auto-reload

Timers are created using `xTimerCreate()`, started with `xTimerStart()`, and run in the **context of the Timer Service Task** (not your application task).

---

### 💡 Example — One-Shot and Auto-Reload Timers

```c
#include "FreeRTOS.h"
#include "task.h"
#include "timers.h"
#include <stdio.h>

/* Timer handles */
TimerHandle_t xOneShotTimer;
TimerHandle_t xAutoReloadTimer;

/* Callback for One-Shot Timer */
void vOneShotCallback(TimerHandle_t xTimer)
{
    printf("⏱ One-shot timer expired. Executing action.\n");
}

/* Callback for Auto-Reload Timer */
void vAutoReloadCallback(TimerHandle_t xTimer)
{
    static int count = 0;
    printf("🔁 Auto-reload timer tick #%d\n", ++count);

    if (count >= 5)
    {
        printf("Stopping auto-reload timer.\n");
        xTimerStop(xAutoReloadTimer, 0);
    }
}

/* Task to demonstrate deferred action */
void vMainTask(void *pv)
{
    printf("Starting timers...\n");

    /* Start both timers */
    xTimerStart(xOneShotTimer, 0);
    xTimerStart(xAutoReloadTimer, 0);

    for (;;)
    {
        printf("Main task is running background operations...\n");
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

/* --- Main Function --- */
int main(void)
{
    /* Create timers */
    xOneShotTimer = xTimerCreate("OneShot", pdMS_TO_TICKS(3000), pdFALSE, 0, vOneShotCallback);
    xAutoReloadTimer = xTimerCreate("AutoReload", pdMS_TO_TICKS(1000), pdTRUE, 0, vAutoReloadCallback);

    /* Create main task */
    xTaskCreate(vMainTask, "MainTask", 1000, NULL, 2, NULL);

    vTaskStartScheduler();

    for (;;);
}
```

---

### 🧾 Output Example

```
Starting timers...
Main task is running background operations...
🔁 Auto-reload timer tick #1
Main task is running background operations...
🔁 Auto-reload timer tick #2
⏱ One-shot timer expired. Executing action.
Main task is running background operations...
🔁 Auto-reload timer tick #3
...
Stopping auto-reload timer.
```

---

### ⚙️ Notes

* **Timer callbacks** execute in **Timer Service Task context**, not your main task.
* Don’t block or call long operations inside a callback — it affects *all* timers.
* Timers are often used for **deferred ISR work** (instead of waking a separate task).

---

## 🧠 1. FreeRTOS Software Timers vs STM32 Hardware Timers

FreeRTOS **software timers** are completely different from **STM32 hardware timers**.

| Feature          | FreeRTOS Software Timer                                                   | STM32 Hardware Timer (TIMx)                                |
| ---------------- | ------------------------------------------------------------------------- | ---------------------------------------------------------- |
| Source           | Derived from **RTOS tick interrupt** (SysTick or alternative tick source) | Uses **dedicated peripheral counter** (TIM1, TIM2, etc.)   |
| Timing accuracy  | Limited to RTOS tick period (e.g. 1 ms typical)                           | Very accurate, independent of CPU load                     |
| Used for         | Scheduling tasks, timeout handling, software events                       | PWM, encoder, frequency measurement, hardware-based events |
| Setup            | Needs no hardware timer setup; uses system tick                           | Requires HAL or LL configuration                           |
| Callback context | Runs in RTOS timer service task                                           | Runs in interrupt context                                  |

So — **you don’t need to configure any STM32 timer peripheral** for FreeRTOS timers to work.

---

## ⚙️ 2. What *is* Required to Make FreeRTOS Timers Work

There are **two** key requirements:

### ✅ a) The RTOS tick must be configured and running

This is handled automatically when:

* You enable FreeRTOS in STM32CubeMX.
* SysTick (or another timer) generates the **tick interrupt** at `configTICK_RATE_HZ` frequency (e.g. 1000 Hz = 1 ms).

> The tick interrupt drives both task scheduling and software timers.

You can check this in your `FreeRTOSConfig.h`:

```c
#define configUSE_TIMERS           1
#define configTIMER_TASK_PRIORITY  (tskIDLE_PRIORITY + 2)
#define configTIMER_QUEUE_LENGTH   10
#define configTIMER_TASK_STACK_DEPTH  256
```

---

### ✅ b) You must start the timer service by calling:

```c
vTaskStartScheduler();
```

Once the scheduler starts:

* The **timer service task** is created internally.
* It processes all pending software timer events.

---

## 🧩 3. STM32CubeMX Setup Example

If you’re using STM32CubeMX:

1. Go to **Middleware → FreeRTOS**.
2. Enable `Timers` (`configUSE_TIMERS = 1`).
3. CubeMX will automatically:

   * Configure SysTick for the RTOS tick.
   * Create the timer task with given stack and priority.

You **don’t** need to configure any hardware timer unless:

* You want a **high-precision hardware-based timer**, or
* You want to **use TIMx for PWM, input capture, etc.**

---

## 💡 4. STM32 Hardware Timer + FreeRTOS Example (Optional Integration)

If you *do* want to use an STM32 timer alongside FreeRTOS, it works independently.

Example: using TIM3 for 10kHz PWM output while FreeRTOS runs software timers.

```c
/* Timer3 Initialization (HAL) */
void MX_TIM3_Init(void)
{
    htim3.Instance = TIM3;
    htim3.Init.Prescaler = 84 - 1; // 1 MHz clock
    htim3.Init.Period = 100 - 1;   // 10 kHz PWM
    HAL_TIM_PWM_Init(&htim3);

    TIM_OC_InitTypeDef sConfigOC = {0};
    sConfigOC.OCMode = TIM_OCMODE_PWM1;
    sConfigOC.Pulse = 50; // 50% duty
    HAL_TIM_PWM_ConfigChannel(&htim3, &sConfigOC, TIM_CHANNEL_1);
}

/* Start PWM */
HAL_TIM_PWM_Start(&htim3, TIM_CHANNEL_1);
```

This **does not** interact with FreeRTOS timers — both systems run concurrently.

---

## 🧭 Summary

| Requirement                          | Needed for FreeRTOS timers? |
| ------------------------------------ | --------------------------- |
| SysTick configured                   | ✅ Yes                       |
| `configUSE_TIMERS` enabled           | ✅ Yes                       |
| Timer service task created           | ✅ Automatically             |
| STM32 TIMx hardware timer configured | ❌ Not needed                |
| `vTaskStartScheduler()` called       | ✅ Must be called            |


