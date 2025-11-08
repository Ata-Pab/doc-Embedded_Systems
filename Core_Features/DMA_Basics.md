## 🧠 1. What DMA Is and Why We Use It

**DMA (Direct Memory Access)** allows peripherals (like I²C, SPI, ADC, UART, etc.) to move data directly to/from **memory (RAM)** without CPU intervention.

### ✳️ The Problem Without DMA

Normally, when you transmit or receive data:

* The CPU writes each byte to a peripheral register (TX buffer).
* Or reads each byte from RX register.

For large transfers (say, 512 bytes over SPI), the CPU wastes time in loops or interrupts, doing nothing but moving bytes.

### ✳️ The Solution With DMA

With DMA:

* You tell the DMA controller:

  * where to read data from (source address)
  * where to write data (destination address)
  * how many elements to move
* Then you **start the peripheral + DMA** and the DMA engine handles each transfer **autonomously**.
* The CPU is free until DMA signals **Transfer Complete** or **Error interrupt**.

**Result:**
✅ Drastically lower CPU load
✅ Better throughput
✅ Deterministic timing (since CPU not involved per byte)

---

## ⚙️ 2. How DMA Works in STM32

### 🔸 Basic Path

```
[Peripheral Data Register] <--> [DMA Controller] <--> [Memory (RAM)]
```

### 🔸 Controllers & Streams (varies by STM32 family)

For example, in STM32F4:

* There are DMA1 and DMA2 controllers.
* Each has several **streams** (Stream0…Stream7).
* Each stream has multiple **channels** (Channel 0…7) used to map to specific peripherals.

Example:

| Peripheral | DMA Controller | Stream  | Channel  |
| ---------- | -------------- | ------- | -------- |
| SPI1_TX    | DMA2           | Stream3 | Channel3 |
| SPI1_RX    | DMA2           | Stream0 | Channel3 |
| I2C1_TX    | DMA1           | Stream6 | Channel1 |
| I2C1_RX    | DMA1           | Stream0 | Channel1 |

*(exact mapping from Reference Manual, section “DMA request mapping”)*

You must pick a **stream/channel combination** that supports your peripheral.

---

## 🧩 3. DMA Configuration Parameters (CubeMX / HAL)

| Setting                 | Meaning                                                                      |
| ----------------------- | ---------------------------------------------------------------------------- |
| **Direction**           | Memory-to-Peripheral (TX), Peripheral-to-Memory (RX), or Memory-to-Memory    |
| **PeriphInc**           | Usually DISABLE — peripheral register fixed                                  |
| **MemInc**              | Usually ENABLE — move through memory array                                   |
| **PeriphDataAlignment** | Byte / Halfword / Word — must match peripheral                               |
| **MemDataAlignment**    | Same                                                                         |
| **Mode**                | NORMAL (one-shot) or CIRCULAR (auto-restart — e.g., ADC continuous sampling) |
| **Priority**            | LOW / MEDIUM / HIGH / VERY_HIGH                                              |
| **FIFO**                | Some families (like F4) have FIFO buffer to optimize bursts                  |

---

## ⚡ 4. DMA Transfer Lifecycle

### 1️⃣ Configure DMA:

Tell it:

* Source (memory or peripheral)
* Destination
* Number of items

### 2️⃣ Enable DMA in peripheral:

The peripheral must know to trigger DMA requests (e.g., SPI TX, UART RX).

### 3️⃣ Start DMA:

Set stream enable bit → DMA starts moving data as peripheral requests.

### 4️⃣ Wait for completion:

* CPU can **poll** (not ideal)
* or **interrupt on completion** (best way)

### 5️⃣ On completion:

* DMA interrupt fires → callback like `HAL_SPI_TxCpltCallback()` executes.

---

## 🧠 5. STM32 HAL DMA Example (Generic Concept)

We’ll use **memory-to-memory** copy example first, because it doesn’t involve other peripherals yet — it’s a pure DMA transfer.

```c
#include "main.h"

DMA_HandleTypeDef hdma_memtomem_dma2_stream0;

uint32_t srcData[100];
uint32_t destData[100];

void MX_DMA_Init(void)
{
  __HAL_RCC_DMA2_CLK_ENABLE();

  hdma_memtomem_dma2_stream0.Instance = DMA2_Stream0;
  hdma_memtomem_dma2_stream0.Init.Channel = DMA_CHANNEL_0;
  hdma_memtomem_dma2_stream0.Init.Direction = DMA_MEMORY_TO_MEMORY;
  hdma_memtomem_dma2_stream0.Init.PeriphInc = DMA_PINC_ENABLE;
  hdma_memtomem_dma2_stream0.Init.MemInc = DMA_MINC_ENABLE;
  hdma_memtomem_dma2_stream0.Init.PeriphDataAlignment = DMA_PDATAALIGN_WORD;
  hdma_memtomem_dma2_stream0.Init.MemDataAlignment = DMA_MDATAALIGN_WORD;
  hdma_memtomem_dma2_stream0.Init.Mode = DMA_NORMAL;
  hdma_memtomem_dma2_stream0.Init.Priority = DMA_PRIORITY_HIGH;
  hdma_memtomem_dma2_stream0.Init.FIFOMode = DMA_FIFOMODE_DISABLE;
  HAL_DMA_Init(&hdma_memtomem_dma2_stream0);
}

int main(void)
{
  HAL_Init();
  MX_DMA_Init();

  for (int i = 0; i < 100; i++) srcData[i] = i;

  // Start DMA transfer (source, destination, number of items)
  HAL_DMA_Start(&hdma_memtomem_dma2_stream0,
                (uint32_t)srcData,
                (uint32_t)destData,
                100);

  // Wait for it to finish (polling)
  HAL_DMA_PollForTransfer(&hdma_memtomem_dma2_stream0, HAL_DMA_FULL_TRANSFER, HAL_MAX_DELAY);

  // Now destData[] contains copied values
  while (1);
}
```

### 💡 Notes

* The HAL automatically configures direction, source, and destination registers.
* For real peripherals, the “Peripheral address” is the peripheral’s **data register**, e.g. `&SPI1->DR` or `&I2C1->RXDR`.

---

## ⚙️ 6. DMA Interrupt Callbacks (when using `_IT` or with peripheral DMA)

HAL provides **callbacks** that fire when transfer completes or errors occur:

| Callback Function                | Trigger                                                  |
| -------------------------------- | -------------------------------------------------------- |
| `HAL_DMA_XferCpltCallback()`     | Full transfer complete                                   |
| `HAL_DMA_XferHalfCpltCallback()` | Half of buffer transferred (useful for double-buffering) |
| `HAL_DMA_ErrorCallback()`        | Error occurred                                           |

You link these callbacks to your peripheral handle when using `HAL_xxx_Transmit_DMA()`.

---

## 🔄 7. CIRCULAR Mode (example concept)

Circular DMA restarts automatically after completion — perfect for continuous data streams (like ADC or UART logging):

```c
hdma.Init.Mode = DMA_CIRCULAR;
```

* When circular, you often use the **Half Complete** and **Complete** callbacks to process data in chunks without stopping DMA.

---

## 🔍 8. DMA and Interrupt Priorities

Each DMA stream interrupt (e.g., `DMA2_Stream0_IRQn`) must be enabled and prioritized in NVIC.

CubeMX does it automatically, or you can manually:

```c
HAL_NVIC_SetPriority(DMA2_Stream0_IRQn, 5, 0);
HAL_NVIC_EnableIRQ(DMA2_Stream0_IRQn);
```

---

## ✅ Summary

| Concept                                      | Meaning                                                                         |
| -------------------------------------------- | ------------------------------------------------------------------------------- |
| DMA moves data                               | Without CPU                                                                     |
| Config: Direction, Increment, Mode, Priority | Essential parameters                                                            |
| Triggers                                     | Peripheral signals DMA when ready                                               |
| Modes                                        | Normal (one-shot) or Circular                                                   |
| Notifications                                | Interrupts or Polling                                                           |
| HAL integration                              | e.g., `HAL_SPI_Transmit_DMA()`, `HAL_UART_Receive_DMA()`, `HAL_ADC_Start_DMA()` |

---

# 🧠 1. I²C + DMA — Conceptual Overview

In standard blocking mode:

* CPU waits while each byte is sent/received.
* For each byte, the CPU writes to or reads from I²C data register (`TXDR` or `RXDR`).

In **DMA mode:**

* You configure DMA to move bytes **between memory and the I²C peripheral data register**.
* DMA automatically feeds the I²C peripheral as it transmits or receives bytes.

### 🔸 Data Flow (Write Example)

```
Memory buffer (TxData[]) ──► DMA ──► I2C1->TXDR ──► SDA/SCL lines
```

### 🔸 Data Flow (Read Example)

```
SDA/SCL lines ──► I2C1->RXDR ──► DMA ──► Memory buffer (RxData[])
```

### ⚙️  Hardware events

Each time the I²C peripheral signals “TX ready” or “RX ready,” DMA transfers a byte automatically, without CPU help.

### 🧩 Result

* The CPU just starts the DMA transfer and waits for completion.
* When all bytes are moved, DMA triggers a **Transfer Complete interrupt**, and HAL calls a **callback** like `HAL_I2C_MasterTxCpltCallback()`.

---

# ⚙️ 2. Typical DMA Flow with I²C (Step-by-Step)

| Step | Description                                                                |
| ---- | -------------------------------------------------------------------------- |
| 1️⃣  | Configure and enable DMA (done by CubeMX automatically)                    |
| 2️⃣  | Configure I²C peripheral (normal HAL init)                                 |
| 3️⃣  | Call a DMA-enabled HAL function like `HAL_I2C_Master_Transmit_DMA()`       |
| 4️⃣  | DMA transfers bytes automatically                                          |
| 5️⃣  | On completion, HAL calls your callback function                            |
| 6️⃣  | In callback, you can signal another task, set flag, or start next transfer |

---

# 🧩 3. STM32 HAL I²C + DMA Example (Master Write)

Assume:

* STM32Cube HAL
* I²C1 configured
* DMA mapping for I²C1_TX and I²C1_RX set by CubeMX

### Example Code

```c
#include "main.h"

extern I2C_HandleTypeDef hi2c1;

uint8_t txData[3] = {0x10, 0x20, 0x30};
uint8_t rxData[3];
uint8_t slaveAddr = 0x50 << 1; // 7-bit address shifted

volatile uint8_t transferDone = 0;

void HAL_I2C_MasterTxCpltCallback(I2C_HandleTypeDef *hi2c)
{
    if (hi2c->Instance == I2C1)
        transferDone = 1; // signal completion
}

int main(void)
{
    HAL_Init();
    SystemClock_Config();
    MX_I2C1_Init(); // Generated by CubeMX
    MX_DMA_Init();  // Also generated automatically

    // Start DMA-based write transfer
    if (HAL_I2C_Master_Transmit_DMA(&hi2c1, slaveAddr, txData, sizeof(txData)) != HAL_OK)
    {
        Error_Handler();
    }

    // Wait for completion (polling or event)
    while (!transferDone);

    // Now maybe start a read transfer
    transferDone = 0;
    if (HAL_I2C_Master_Receive_DMA(&hi2c1, slaveAddr, rxData, sizeof(rxData)) != HAL_OK)
    {
        Error_Handler();
    }

    while (!transferDone);

    while (1)
    {
        // main loop
    }
}
```

### 🔸 Explanation

* `HAL_I2C_Master_Transmit_DMA()`:

  * Configures DMA with `Memory -> Peripheral` direction.
  * DMA moves bytes from `txData[]` to `I2C1->TXDR`.
  * HAL automatically enables the necessary I²C DMA requests.
* Callback `HAL_I2C_MasterTxCpltCallback()` is executed when all data is transmitted.
* Same pattern applies for receive (`HAL_I2C_Master_Receive_DMA()`).

---

# ⚙️ 4. How the HAL Manages DMA for You

Internally:

* HAL configures the DMA stream’s:

  * source = buffer address (`txData`)
  * destination = peripheral register address (`I2C1->TXDR`)
  * transfer size = buffer length
  * direction = memory-to-peripheral
* HAL enables the I²C’s TXDMAEN bit (in CR1).
* DMA hardware takes over.

---

# 🔍 5. DMA + I²C Memory Read Example (e.g., sensor register)

This is a very common case:
You need to **write a register address**, then **read data**.

### Using DMA:

```c
uint8_t regAddr = 0x0A;
uint8_t sensorData[2];
uint8_t devAddr = 0x68 << 1;

HAL_I2C_Master_Transmit_DMA(&hi2c1, devAddr, &regAddr, 1);
while (!transferDone);  // Wait TX done

transferDone = 0;
HAL_I2C_Master_Receive_DMA(&hi2c1, devAddr, sensorData, 2);
while (!transferDone);
```

### Or shorter:

```c
HAL_I2C_Mem_Read_DMA(&hi2c1, devAddr, regAddr, I2C_MEMADD_SIZE_8BIT, sensorData, 2);
```

This is the HAL helper that automatically performs write-then-read using DMA.

---

# ⚠️ 6. Important Practical Details

### 1️⃣ Address shifting

HAL expects an **8-bit address** → always shift the 7-bit address left by 1.

### 2️⃣ I²C bus busy / NACK handling

DMA can’t recover from bus errors by itself.
HAL will abort the transfer and call `HAL_I2C_ErrorCallback()` — always implement this.

```c
void HAL_I2C_ErrorCallback(I2C_HandleTypeDef *hi2c)
{
    // e.g., reset state or reinit
}
```

### 3️⃣ Never modify DMA buffers while transfer is ongoing.

DMA and CPU could access same memory — leads to undefined data.

### 4️⃣ Priority & NVIC

Make sure DMA IRQ priorities are lower than critical RTOS interrupts if using FreeRTOS.

---

# 🔄 7. Comparing I²C DMA vs I²C Interrupt Mode

| Aspect            | DMA Mode                 | Interrupt Mode       |
| ----------------- | ------------------------ | -------------------- |
| CPU Load          | Very Low                 | Medium               |
| Overhead per byte | None                     | 1 interrupt per byte |
| Best For          | Large blocks             | Short quick commands |
| Latency           | Slightly higher start-up | Low for small data   |
| Complexity        | Slightly more setup      | Simpler              |

So, for **sensor readings** or **LCD writes** (big buffers), DMA is ideal.
For **quick commands** (single register writes), interrupts or blocking mode is fine.

---

# 🧩 8. Debug Tips

* **Bus Analyzer / Logic Analyzer**: Always verify START–ADDR–DATA–STOP sequences.
* **HAL_I2C_GetError()**: Check error flags after each transfer.
* **Stuck bus**: If SDA stays low, reinitialize I²C or pulse SCL lines to recover.
* **DMA Interrupt fires but no data:** Check stream mapping (CubeMX sets this automatically, but manual config might mismatch).

---

# ✅ Summary

| Concept                                                       | Description                                      |
| ------------------------------------------------------------- | ------------------------------------------------ |
| DMA moves data between I²C registers and memory automatically | CPU-free transfers                               |
| Use HAL functions with `_DMA` suffix                          | e.g., `HAL_I2C_Master_Transmit_DMA()`            |
| Completion handled by callbacks                               | `HAL_I2C_MasterTxCpltCallback()` etc.            |
| Error callback important                                      | to handle bus faults                             |
| Perfect for                                                   | Sensor data streaming, EEPROM reads, LCD refresh |

---

```Why do we wait transfer_end (while (!transferDone);) in an infinite loop? To prevent buffer changes? If we have big transfer data this wait loop causes CPU halt, where is the advantage of using DMA in this case?```

Excellent and very sharp question — this shows you’ve really understood how DMA works in practice. 👏
Let’s unpack this carefully, because this is exactly where many people misunderstand DMA’s purpose.

---

## 🔍 1. What the `while (!transferDone);` Loop Really Means

In our DMA example, that line:

```c
while (!transferDone);
```

is **only used for demonstration** — it’s there to **block the main loop** until the DMA transfer completes, so we can safely continue the next step (like printing, validating, or starting another transfer).

But in *real embedded applications*, we usually **do not** wait in a blocking loop like this.
Because you’re absolutely right — that would **stall the CPU** and **defeat the benefit** of DMA.

---

## ⚙️ 2. What DMA Is Actually For

DMA = **Direct Memory Access**, meaning it transfers data between peripherals and memory **without CPU intervention**.

So, while DMA is transferring, your **CPU is completely free** to do other useful work:

* Execute other tasks (e.g. PID controller, data processing, etc.)
* Enter low-power (sleep) mode to save energy
* Handle other interrupts or background jobs

---

## 💡 3. Why We Wait in Some Examples (Like `while (!transferDone);`)

Because in small test/demo programs (like STM32Cube examples), the main loop doesn’t have other meaningful tasks.
So instead of creating a complex state machine, they just wait for completion to keep it simple.

In real firmware, you’d structure it like this:

```c
HAL_DMA_Start_IT(&hdma_usart2_tx, (uint32_t)txBuffer, (uint32_t)&USART2->TDR, sizeof(txBuffer));
// No blocking wait!

// CPU continues with other jobs
DoSensorSampling();
UpdateDisplay();
RunStateMachine();

// Later, when transfer completes, DMA interrupt triggers:
void HAL_UART_TxCpltCallback(UART_HandleTypeDef *huart)
{
    transferDone = 1;
    // Maybe schedule next task or notify FreeRTOS queue/semaphore
}
```

So the **DMA interrupt handler** notifies you when the job finishes.

---

## ⚖️ 4. The Real Advantages of DMA

| Feature                  | DMA                         | Interrupt-only | Polling |
| ------------------------ | --------------------------- | -------------- | ------- |
| CPU usage                | Minimal (non-blocking)      | Medium         | High    |
| Transfer size            | Excellent for large buffers | Limited        | Limited |
| Power efficiency         | High (CPU can sleep)        | Medium         | Low     |
| Code complexity          | Moderate                    | Moderate       | Low     |
| Latency (start→transfer) | Slightly higher setup       | Lower          | Lowest  |

DMA shines especially for **large or continuous data streams**:

* ADC → Memory (sampling)
* Memory → SPI (LCD, display buffer)
* UART → Memory (data logging)
* I2C → Memory (sensor readout)

---

## 🧠 5. How to Avoid CPU Halt While Waiting

If you really need to wait until a transfer is done **without halting**, you can:

### Option 1 — Use Interrupt Callbacks

Let the DMA interrupt notify you via a flag, semaphore, or event.

### Option 2 — Use FreeRTOS Synchronization

Wait on a queue, semaphore, or event group rather than looping.

Example:

```c
// Task waits for DMA to finish
xSemaphoreTake(xDMADone, portMAX_DELAY);

// DMA completion ISR
void HAL_DMA_TxCpltCallback(DMA_HandleTypeDef *hdma)
{
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    xSemaphoreGiveFromISR(xDMADone, &xHigherPriorityTaskWoken);
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}
```

Here, CPU is **sleeping or doing other work** until the DMA completes.

---

## ✅ Summary

So your observation is correct:

> Waiting in a loop wastes CPU time — it’s only for simple examples.

**In real systems:**

* DMA runs independently
* CPU does other work or sleeps
* Interrupt callback signals transfer completion

That’s the **core advantage** of DMA: offloading repetitive transfer tasks to hardware.

---

# Real World DMA Example

## 🧠 1. Concept Overview

We’ll configure DMA to send a buffer over UART.
The CPU will **not** wait in a loop — instead, it will be **notified via interrupt** when the transfer is done.

Here’s the high-level flow:

```
+-------------+
| CPU starts  |----> HAL_UART_Transmit_DMA() ----+
+-------------+                                  |
        |                                        |
        |                                +-------------------+
        |                                | DMA controller    |
        |                                | sends data byte by|
        |                                | byte automatically|
        |                                +-------------------+
        |                                        |
        |<---------------------------------------+
   Callback interrupt (HAL_UART_TxCpltCallback)
```

---

## ⚙️ 2. Example: UART TX with DMA (Non-blocking)

Let’s assume:

* `USART2` is used
* `DMA1_Channel7` (common for STM32F4)
* Using CubeIDE or STM32 HAL drivers

### **Code Example:**

```c
#include "main.h"
#include <string.h>

UART_HandleTypeDef huart2;
DMA_HandleTypeDef hdma_usart2_tx;

/* Global flag for transfer completion */
volatile uint8_t txDone = 0;

/* Message buffer */
char txMessage[] = "Hello from STM32 using DMA (non-blocking)!\r\n";

/* --- DMA interrupt callback --- */
void HAL_UART_TxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART2)
    {
        txDone = 1; // Set completion flag
    }
}

/* --- UART + DMA Initialization --- */
void MX_USART2_UART_Init(void)
{
    huart2.Instance = USART2;
    huart2.Init.BaudRate = 115200;
    huart2.Init.WordLength = UART_WORDLENGTH_8B;
    huart2.Init.StopBits = UART_STOPBITS_1;
    huart2.Init.Parity = UART_PARITY_NONE;
    huart2.Init.Mode = UART_MODE_TX_RX;
    huart2.Init.HwFlowCtl = UART_HWCONTROL_NONE;
    huart2.Init.OverSampling = UART_OVERSAMPLING_16;

    if (HAL_UART_Init(&huart2) != HAL_OK)
    {
        Error_Handler();
    }
}

/* --- DMA Initialization (Generated by CubeMX normally) --- */
void MX_DMA_Init(void)
{
    __HAL_RCC_DMA1_CLK_ENABLE();

    /* Configure NVIC for DMA interrupt */
    HAL_NVIC_SetPriority(DMA1_Channel7_IRQn, 0, 0);
    HAL_NVIC_EnableIRQ(DMA1_Channel7_IRQn);
}

/* --- DMA Interrupt Handler --- */
void DMA1_Channel7_IRQHandler(void)
{
    HAL_DMA_IRQHandler(&hdma_usart2_tx);
}

/* --- Main Function --- */
int main(void)
{
    HAL_Init();
    SystemClock_Config();
    MX_DMA_Init();
    MX_USART2_UART_Init();

    /* Start DMA transmission */
    HAL_UART_Transmit_DMA(&huart2, (uint8_t *)txMessage, strlen(txMessage));

    /* No blocking wait! */
    while (1)
    {
        if (txDone)
        {
            txDone = 0;
            HAL_UART_Transmit_DMA(&huart2, (uint8_t *)txMessage, strlen(txMessage));
            HAL_Delay(1000); // Repeat every second
        }

        // CPU can do other tasks here
        DoSensorRead();
        UpdateDisplay();
        HandleButtons();
    }
}
```

---

## 🧩 3. What’s Happening

| Step | Description                                                                          |
| ---- | ------------------------------------------------------------------------------------ |
| ①    | `HAL_UART_Transmit_DMA()` starts DMA engine — it immediately returns (non-blocking). |
| ②    | DMA sends bytes from memory to UART peripheral automatically.                        |
| ③    | When DMA finishes, it triggers an **interrupt**.                                     |
| ④    | HAL calls `HAL_UART_TxCpltCallback()`.                                               |
| ⑤    | Inside callback, you set a flag or send a signal to RTOS task.                       |
| ⑥    | Main loop checks the flag and continues next work.                                   |

---

## ⚡ 4. CPU Efficiency Gain

✅ CPU is free during transfer (can handle other tasks).
✅ No need for polling or busy-wait loops.
✅ Ideal for **large data blocks** or **continuous streaming**.
✅ Integrates seamlessly with **FreeRTOS** (via semaphores or event groups).

---

## 🧩 5. Optional: RTOS Integration Example

If using **FreeRTOS**, you’d replace the flag with a semaphore:

```c
SemaphoreHandle_t xDMATxDone;

void HAL_UART_TxCpltCallback(UART_HandleTypeDef *huart)
{
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    xSemaphoreGiveFromISR(xDMATxDone, &xHigherPriorityTaskWoken);
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}

void MyTask(void *pvParameters)
{
    for (;;)
    {
        HAL_UART_Transmit_DMA(&huart2, (uint8_t *)msg, strlen(msg));
        xSemaphoreTake(xDMATxDone, portMAX_DELAY);
        // Continue after transmission complete
    }
}
```

---

## 🧠 Summary

| Without DMA           | With DMA + Interrupt              |
| --------------------- | --------------------------------- |
| CPU copies every byte | Hardware DMA copies automatically |
| CPU blocked           | CPU free to do other work         |
| High overhead         | Minimal CPU usage                 |
| Simpler but wasteful  | Slightly more setup but efficient |

---
