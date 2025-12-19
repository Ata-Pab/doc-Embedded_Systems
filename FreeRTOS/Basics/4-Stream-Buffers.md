## What Is a Stream Buffer?

A **Stream Buffer** is a unidirectional byte stream between:

* **One writer (sender)** and
* **One reader (receiver)**

It’s typically used when you need to send a variable number of bytes, not discrete messages.

> Unlike a Queue, it doesn’t store item “slots” — it’s a circular buffer holding raw bytes.

## Common Real Use Case

**UART receive handling** is a *perfect* match:

* The **UART RX ISR** writes incoming bytes into the stream buffer.
* A **Parser Task** reads from the stream buffer, processes complete commands or frames.


## System Setup (Concept)

| Component                | Description                                                               |
| ------------------------ | ------------------------------------------------------------------------- |
| **UART RX ISR**          | Fills received bytes into StreamBuffer using `xStreamBufferSendFromISR()` |
| **Parser Task**          | Reads bytes using `xStreamBufferReceive()` and parses                     |
| **Stream Buffer Handle** | Shared object created before scheduler start                              |

## UART RX Stream Buffer Demo

### Code Skeleton

```c
#include "FreeRTOS.h"
#include "task.h"
#include "stream_buffer.h"

/* UART driver function prototypes (example) */
void UART_Init(void);
void UART_EnableRxInterrupt(void);
uint8_t UART_ReadByte(void);

/* Stream buffer handle */
StreamBufferHandle_t xUartRxStreamBuffer;

/* ISR prototype (defined in vector table normally) */
void USARTx_IRQHandler(void);

/* Parser Task */
void vParserTask(void *pvParameters);

int main(void)
{
    UART_Init();

    /* Create a stream buffer of 128 bytes total, trigger level = 1 byte */
    xUartRxStreamBuffer = xStreamBufferCreate(128, 1);

    if (xUartRxStreamBuffer == NULL)
    {
        /* Error handling */
        for (;;);
    }

    xTaskCreate(vParserTask, "Parser", 256, NULL, 2, NULL);

    UART_EnableRxInterrupt();

    vTaskStartScheduler();
    for (;;);
}

/* ---------------- UART RX ISR ---------------- */
void USARTx_IRQHandler(void)
{
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    uint8_t ucReceived;

    /* Read the received byte from hardware register */
    ucReceived = UART_ReadByte();

    /* Send byte into stream buffer (from ISR context) */
    xStreamBufferSendFromISR(xUartRxStreamBuffer, &ucReceived, 1, &xHigherPriorityTaskWoken);

    /* If parser task was blocked waiting for data, unblock it */
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}

/* ---------------- Parser Task ---------------- */
void vParserTask(void *pvParameters)
{
    uint8_t rxBuffer[32];
    size_t xReceivedBytes;

    for (;;)
    {
        /* Block until at least one byte is available */
        xReceivedBytes = xStreamBufferReceive(xUartRxStreamBuffer,
                                              rxBuffer,
                                              sizeof(rxBuffer),
                                              portMAX_DELAY);

        /* Process received bytes */
        for (size_t i = 0; i < xReceivedBytes; i++)
        {
            /* Example: simple protocol parsing */
            if (rxBuffer[i] == '\n')
            {
                // End of command line detected
                // Call CommandParser()
            }
        }
    }
}
```

## Behavior Summary

| Scenario                | Behavior                                                        |
| ----------------------- | --------------------------------------------------------------- |
| UART ISR receives bytes | Adds them into Stream Buffer                                    |
| Parser task blocked     | Unblocked automatically when data arrives                       |
| Stream buffer full      | Oldest data overwritten (if implemented manually) or send fails |
| No ISR data             | Task remains blocked (no CPU waste)                             |

## Why Stream Buffer (vs Queue)?

| Feature                       | Queue                       | Stream Buffer  |
| ----------------------------- | --------------------------- | -------------- |
| Stores discrete messages      | ✓                           | ❌              |
| Stores continuous byte stream | ❌                           | ✓              |
| Suitable for UART RX/TX       | ⚠️ possible but not optimal | ✓ Ideal        |
| ISR safe                      | ✓                           | ✓              |
| Memory efficient              | Medium                      | Very efficient |

## Real-World Example Context

A fridge controller or IoT board that communicates over UART with a diagnostic tool or Wi-Fi module:

* ISR keeps buffering UART data into the stream.
* A parser task interprets frames (e.g., JSON, AT commands).
* Decouples real-time RX from CPU-heavy parsing.

