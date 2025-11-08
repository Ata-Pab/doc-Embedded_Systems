Perfect — now we step into one of the most *essential* FreeRTOS concepts for writing robust multitasking code:
👉 **Mutexes (Mutual Exclusion Objects)**

They look similar to semaphores at first glance — but their purpose and behavior are quite different once you understand the details.

---

## 🧩 1. What is a Mutex?

A **Mutex** (short for *mutual exclusion*) is used to **protect shared resources** — such as:

* Shared global variables
* I²C / SPI peripheral access
* Shared communication buffers
* Logging / printing functions

It ensures **only one task** at a time can access that resource.

---

## ⚙️ 2. How Mutex Works

Internally, a Mutex is like a **binary semaphore**, but with a few key differences:

| Feature              | Binary Semaphore        | Mutex                               |
| -------------------- | ----------------------- | ----------------------------------- |
| Ownership            | No concept of ownership | The task that takes it *owns* it    |
| Priority Inheritance | ❌ No                    | ✅ Yes — prevents priority inversion |
| Typical Use          | Event signaling         | Protecting critical sections        |
| Give/Take From ISR   | ✅ Yes                   | ❌ No (must use in tasks only)       |

---

## 🧠 3. Real Example: Protecting UART (or printf)

Let’s say two tasks both print messages via UART or `printf`.
Without a mutex, their messages can overlap or interleave (data corruption).

### ✅ Example Code

```c
#include "FreeRTOS.h"
#include "task.h"
#include "semphr.h"
#include <stdio.h>

SemaphoreHandle_t xMutex;

void vTask1(void *pvParameters)
{
    for(;;)
    {
        /* Wait until mutex is available */
        xSemaphoreTake(xMutex, portMAX_DELAY);

        printf("Task 1 entering critical section...\n");
        vTaskDelay(pdMS_TO_TICKS(500));  // Simulate work
        printf("Task 1 leaving critical section...\n");

        xSemaphoreGive(xMutex);  // Release mutex
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

void vTask2(void *pvParameters)
{
    for(;;)
    {
        xSemaphoreTake(xMutex, portMAX_DELAY);

        printf("Task 2 entering critical section...\n");
        vTaskDelay(pdMS_TO_TICKS(300));  // Simulate work
        printf("Task 2 leaving critical section...\n");

        xSemaphoreGive(xMutex);
        vTaskDelay(pdMS_TO_TICKS(700));
    }
}

int main(void)
{
    /* Create mutex */
    xMutex = xSemaphoreCreateMutex();
    if (xMutex == NULL)
    {
        printf("Failed to create mutex!\n");
        for(;;);
    }

    xTaskCreate(vTask1, "Task1", 1000, NULL, 2, NULL);
    xTaskCreate(vTask2, "Task2", 1000, NULL, 2, NULL);

    vTaskStartScheduler();
    for(;;);
}
```

---

### 🧾 Expected Console Output:

```
Task 1 entering critical section...
Task 1 leaving critical section...
Task 2 entering critical section...
Task 2 leaving critical section...
Task 1 entering critical section...
Task 1 leaving critical section...
...
```

Each task *waits* for the mutex before entering the critical section → no interleaved output.
Only one task can access the shared resource (UART/console) at a time.

---

## ⚖️ 4. Priority Inheritance (Why Mutex ≠ Binary Semaphore)

**Priority inversion** can occur when:

* A low-priority task holds a mutex.
* A high-priority task tries to take the same mutex and blocks.
* A medium-priority task keeps preempting the low one → high-priority task starves.

Mutexes solve this with **priority inheritance**:

> The low-priority task **temporarily inherits** the higher priority
> so it can finish quickly and release the mutex.

Binary semaphores do **not** provide this — they’re only counters.

---

## 🔁 5. Real Embedded Example

| Scenario                      | Protected Resource | Why Mutex is used            |
| ----------------------------- | ------------------ | ---------------------------- |
| Two tasks writing to UART     | UART TX buffer     | Prevent garbled output       |
| Logging to SD card            | File system driver | Avoid corrupted files        |
| Shared I²C peripheral         | I²C bus            | Prevent overlapping commands |
| Updating shared global struct | Config/state data  | Prevent inconsistent reads   |

---

## 🧩 6. Recursive Mutex (Special Case)

Sometimes, a task might need to **lock the same mutex multiple times** (e.g., nested function calls using the same resource).

For that, use:

```c
xSemaphoreCreateRecursiveMutex();
xSemaphoreTakeRecursive(xMutex, portMAX_DELAY);
xSemaphoreGiveRecursive(xMutex);
```

Normal mutexes will *deadlock* if taken twice by the same task; recursive mutexes allow re-entrancy safely.

---

✅ **Summary:**

| Mechanism          | Use for                        | ISR-safe | Priority Inheritance |
| ------------------ | ------------------------------ | -------- | -------------------- |
| Binary Semaphore   | Event signaling                | ✅        | ❌                    |
| Counting Semaphore | Multiple identical resources   | ✅        | ❌                    |
| Mutex              | Protecting shared resources    | ❌        | ✅                    |
| Recursive Mutex    | Re-entrant resource protection | ❌        | ✅                    |

---

## 🧩 What Is a Recursive Mutex?

A **Recursive Mutex** is a special type of mutex that allows **the same task** to lock the mutex **multiple times** without causing a deadlock.
Each time the mutex is taken by the same task, an internal counter increments. The task must release (give) it **the same number of times** to fully unlock it.

---

## 🔍 Why Do We Need Recursive Mutexes?

In a large embedded codebase, it’s common for multiple functions to **call each other**, and each may try to acquire the same mutex to protect a shared resource (like I²C bus, SPI interface, or UART buffer).

Without a recursive mutex:

* The **second call** (from within the same task) will block forever, since the mutex is already taken by the *same* task → **deadlock**.

With a recursive mutex:

* The same task can safely re-enter sections protected by the same mutex.

---

## ⚙️ Example — When to Use It

Imagine this:

```c
void WriteToEEPROM(uint8_t *data);
void SaveDeviceConfig();
```

`SaveDeviceConfig()` calls `WriteToEEPROM()` internally,
and both functions lock the same mutex (`xEEPROM_Mutex`) to protect I2C access.

Without recursive mutex → Deadlock!
With recursive mutex → Works safely.

---

## ✅ Pros

* Prevents **deadlocks** in nested calls by the same task.
* Simplifies API layering (each function can protect itself independently).

## ⚠️ Cons

* Slightly **more overhead** (counter management).
* Can hide poor design — overuse may make code logic messy.
* Should **not be shared across unrelated resources** (use regular mutexes then).

---

## 💻 Real Embedded-Style Example

Let’s simulate a **shared I2C bus** where multiple layers of functions try to write to EEPROM.

### Code Example (Recursive Mutex)

```c
#include "FreeRTOS.h"
#include "task.h"
#include "semphr.h"
#include <stdio.h>

/* Recursive mutex handle */
SemaphoreHandle_t xI2C_Mutex;

/* --- Low-level driver --- */
void I2C_Write(uint8_t address, uint8_t data)
{
    /* Lock before accessing I2C hardware */
    xSemaphoreTakeRecursive(xI2C_Mutex, portMAX_DELAY);
    printf("Writing 0x%02X to address 0x%02X\n", data, address);
    vTaskDelay(pdMS_TO_TICKS(50));  // Simulate delay
    xSemaphoreGiveRecursive(xI2C_Mutex);
}

/* --- High-level API --- */
void WriteToEEPROM(uint8_t address, uint8_t value)
{
    /* This function also protects access */
    xSemaphoreTakeRecursive(xI2C_Mutex, portMAX_DELAY);

    printf("EEPROM write started\n");
    I2C_Write(address, value);   // Nested call!
    printf("EEPROM write completed\n");

    xSemaphoreGiveRecursive(xI2C_Mutex);
}

/* --- Task that uses EEPROM --- */
void vTask_ConfigWriter(void *pv)
{
    for (;;)
    {
        WriteToEEPROM(0x10, 0xAB);
        vTaskDelay(pdMS_TO_TICKS(500));
    }
}

/* --- Main Setup --- */
int main(void)
{
    xI2C_Mutex = xSemaphoreCreateRecursiveMutex();

    xTaskCreate(vTask_ConfigWriter, "ConfigWriter", 1000, NULL, 1, NULL);

    vTaskStartScheduler();
    for(;;);
}
```

### Output (simulated)

```
EEPROM write started
Writing 0xAB to address 0x10
EEPROM write completed
```

---

### 🔧 Explanation

* `WriteToEEPROM()` locks the mutex.
* Then it calls `I2C_Write()` — which **tries to lock the same mutex again**.
* Since it’s the **same task**, and the mutex is **recursive**, this is allowed.
* The counter increases internally, and both functions must “give” it back.

If it were a **normal mutex**, the task would **deadlock** in `I2C_Write()`.
