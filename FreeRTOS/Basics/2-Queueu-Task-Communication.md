### 🧩 Task Communication Using FreeRTOS Queues

#### Concept Overview

* **Queues** in FreeRTOS allow tasks (or ISRs) to send and receive messages safely.
* Each item in a queue is a fixed-size structure (can be integer, struct, etc.).
* Tasks can **block** on a queue (wait until data arrives or space frees up).
* Queues are essential for event passing, message buffering, and control commands.

#### Example Scenario

We’ll create:

* **Producer Task** — periodically sends integer messages to a queue.
* **Consumer Task** — receives and prints messages from the queue.

---

### Example Code (for your WIN32 FreeRTOS project)

Create a new source file, e.g. `QueueDemo.c`:

```c
#include "FreeRTOS.h"
#include "task.h"
#include "queue.h"
#include <stdio.h>

QueueHandle_t xQueue;

void vProducerTask(void *pvParameters)
{
    int counter = 0;
    while (1)
    {
        counter++;
        if (xQueueSend(xQueue, &counter, portMAX_DELAY) == pdPASS)
        {
            printf("Producer sent: %d\n", counter);
        }
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

void vConsumerTask(void *pvParameters)
{
    int receivedValue = 0;
    while (1)
    {
        if (xQueueReceive(xQueue, &receivedValue, portMAX_DELAY) == pdPASS)
        {
            printf("Consumer received: %d\n", receivedValue);
        }
    }
}

int main(void)
{
    xQueue = xQueueCreate(5, sizeof(int));
    if (xQueue == NULL)
    {
        printf("Failed to create queue!\n");
        return 1;
    }

    xTaskCreate(vProducerTask, "Producer", 1000, NULL, 1, NULL);
    xTaskCreate(vConsumerTask, "Consumer", 1000, NULL, 1, NULL);

    vTaskStartScheduler();
    return 0;
}
```

---

### Expected Console Output:

```
Producer sent: 1
Consumer received: 1
Producer sent: 2
Consumer received: 2
Producer sent: 3
Consumer received: 3
...
```

---

### 🔍 Key Points:

* `xQueueCreate(5, sizeof(int))` → Queue can hold 5 integers.
* `xQueueSend` → Adds an item; blocks if full.
* `xQueueReceive` → Waits until an item is available.
* Both tasks are **synchronized by the queue** — no need for locks.

---

## 🧭 Where Queues Are Used in Real Embedded Projects

Here are some very typical examples — you’ll see this pattern across industrial, automotive, and appliance control firmware:

### 1. **Event Messaging Between Control Tasks**

Each control task (Fan Control, Compressor Control, etc.) may send *events* to a central manager task.

```c
typedef enum { EVENT_COMPRESSOR_ON, EVENT_DOOR_OPEN, EVENT_DEFROST_START } SystemEventType;

typedef struct
{
    SystemEventType eventType;
    uint32_t timestamp;
    uint8_t sourceTaskId;
} SystemEvent_t;
```

The queue transfers these events to a *System Manager Task*, which then updates the system state machine.

---

### 2. **Command and Data Passing**

When a UI task or network task sends a *command* to another subsystem:

```c
typedef struct
{
    uint8_t commandId;
    uint8_t targetComponent;
    uint16_t value;
} ControlCommand_t;
```

A queue between the **UI Task → Control Task** can deliver such messages safely.

---

### 3. **Sensor Data Pipeline**

Sensors running in interrupt-driven or periodic tasks post their measurements into a queue:

```c
typedef struct
{
    float temperature;
    float humidity;
    uint32_t timestamp;
} SensorData_t;
```

Another task (e.g. Logging or Display) reads these structs from the queue for further processing.

---

### 4. **Interfacing Between ISR and Tasks**

Interrupt Service Routines (ISRs) use queues to *defer heavy work* to background tasks:

```c
void EXTI_IRQHandler(void)
{
    SensorData_t data;
    data.temperature = ReadTemp();
    data.timestamp = xTaskGetTickCountFromISR();
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    xQueueSendFromISR(sensorQueue, &data, &xHigherPriorityTaskWoken);
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}
```

→ This lets the ISR remain lightweight while the task processes the data.

---

## ⚙️ Practical Example — Passing Structs via Queue

Let’s create a realistic scenario — imagine you’re designing a **FreeRTOS-based refrigerator control system**, where a **Temperature Sensor Task** periodically reads data and sends it to a **Cooling Controller Task**.

```c
#include "FreeRTOS.h"
#include "task.h"
#include "queue.h"
#include <stdio.h>

typedef struct
{
    float freezerTemp;
    float coolerTemp;
    uint32_t timestamp;
} TemperatureData_t;

QueueHandle_t xTempQueue;

void vSensorTask(void *pvParameters)
{
    TemperatureData_t data;
    uint32_t counter = 0;
    while (1)
    {
        // Fake temperature values
        data.freezerTemp = -18.0f + (rand() % 4 - 2);
        data.coolerTemp = 4.0f + (rand() % 3 - 1);
        data.timestamp = counter++;

        if (xQueueSend(xTempQueue, &data, portMAX_DELAY) == pdPASS)
        {
            printf("[Sensor] Sent: Freezer=%.1f°C, Cooler=%.1f°C\n",
                   data.freezerTemp, data.coolerTemp);
        }
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

void vControllerTask(void *pvParameters)
{
    TemperatureData_t recvData;
    while (1)
    {
        if (xQueueReceive(xTempQueue, &recvData, portMAX_DELAY) == pdPASS)
        {
            printf("→ [Controller] Got: F=%.1f°C, C=%.1f°C, t=%lu\n",
                   recvData.freezerTemp, recvData.coolerTemp, recvData.timestamp);

            // Decision logic example
            if (recvData.freezerTemp > -16.0f)
                printf("   [Action] Increase compressor speed\n");
            else if (recvData.freezerTemp < -19.0f)
                printf("   [Action] Decrease compressor speed\n");
        }
    }
}

int main(void)
{
    xTempQueue = xQueueCreate(10, sizeof(TemperatureData_t));
    if (xTempQueue == NULL)
    {
        printf("Failed to create queue!\n");
        return 1;
    }

    xTaskCreate(vSensorTask, "Sensor", 1000, NULL, 2, NULL);
    xTaskCreate(vControllerTask, "Controller", 1000, NULL, 1, NULL);

    vTaskStartScheduler();
    return 0;
}
```

---

### 🧾 Output Example:

```
[Sensor] Sent: Freezer=-18.0°C, Cooler=4.0°C
→ [Controller] Got: F=-18.0°C, C=4.0°C, t=0
[Action] OK (no change)

[Sensor] Sent: Freezer=-15.0°C, Cooler=5.0°C
→ [Controller] Got: F=-15.0°C, C=5.0°C, t=1
[Action] Increase compressor speed
```

---

### 🧠 Key Takeaways

* Struct-based queues let you *encapsulate context-rich messages*.
* Each task focuses only on its domain (separation of concerns).
* It makes debugging, testing, and scalability much easier.
* Queues guarantee thread-safe delivery — no shared-variable hazards.

