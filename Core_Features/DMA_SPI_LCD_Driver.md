# 🧠 Step 1: Concept — Why SPI + DMA for LCD?

Most **TFT, OLED, and segment LCD displays** (especially SPI-based ones like ST7735, ILI9341, SSD1306, etc.) are driven using **SPI + DMA** in embedded systems.

Why? Because sending a full frame buffer (e.g. 128×160 pixels × 2 bytes = 40,960 bytes) **would kill the CPU** if done byte-by-byte — but **DMA can blast it out automatically** with almost zero CPU load.

When updating a display:

* You typically prepare a **frame buffer** in memory (e.g., `uint16_t frame[128][160]`)
* Then you send it to the display controller over **SPI**
* Without DMA → CPU manually pushes each byte through SPI DR register
* With DMA → SPI + DMA handle everything while CPU is free to compute next frame, update UI, etc.

**Result:**
✅ Smoother animations
✅ More CPU time
✅ Stable frame rate

---

# ⚙️ Step 2: SPI + DMA LCD Example (Realistic STM32 Code)

Let’s use **STM32 + ST7735 TFT LCD** (SPI-based) as a concrete example.

---

## 🔧 Hardware Setup (Common Pinout)

| LCD Pin | STM32 Pin | Description           |
| ------- | --------- | --------------------- |
| VCC     | 3.3V      | Power                 |
| GND     | GND       | Ground                |
| CS      | GPIO      | Chip Select           |
| DC      | GPIO      | Data/Command select   |
| RESET   | GPIO      | Reset pin             |
| SCK     | SPIx_SCK  | SPI clock             |
| MOSI    | SPIx_MOSI | SPI data (Master Out) |

---

# 🧩 Step 3: Code Example — DMA-Based SPI LCD Driver

```c
#include "main.h"
#include <string.h>

SPI_HandleTypeDef hspi1;
DMA_HandleTypeDef hdma_spi1_tx;

/* LCD Control Pins */
#define LCD_CS_LOW()     HAL_GPIO_WritePin(GPIOB, GPIO_PIN_6, GPIO_PIN_RESET)
#define LCD_CS_HIGH()    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_6, GPIO_PIN_SET)
#define LCD_DC_LOW()     HAL_GPIO_WritePin(GPIOB, GPIO_PIN_7, GPIO_PIN_RESET)
#define LCD_DC_HIGH()    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_7, GPIO_PIN_SET)

/* Frame buffer */
#define LCD_WIDTH   128
#define LCD_HEIGHT  160
uint16_t frameBuffer[LCD_WIDTH * LCD_HEIGHT];

/* DMA completion flag */
volatile uint8_t spiDmaDone = 0;

/* --- DMA complete callback --- */
void HAL_SPI_TxCpltCallback(SPI_HandleTypeDef *hspi)
{
    if (hspi->Instance == SPI1)
    {
        spiDmaDone = 1;
        LCD_CS_HIGH(); // Deselect LCD
    }
}

/* --- Send command (1 byte, blocking) --- */
static void LCD_SendCommand(uint8_t cmd)
{
    LCD_CS_LOW();
    LCD_DC_LOW();
    HAL_SPI_Transmit(&hspi1, &cmd, 1, HAL_MAX_DELAY);
    LCD_DC_HIGH();
}

/* --- Send data using DMA --- */
static void LCD_SendDataDMA(uint8_t *data, uint32_t size)
{
    spiDmaDone = 0;
    LCD_CS_LOW();
    LCD_DC_HIGH();
    HAL_SPI_Transmit_DMA(&hspi1, data, size);
}

/* --- Wait for DMA transfer to finish --- */
static void LCD_WaitForDMA(void)
{
    while (!spiDmaDone)
        ; // Optionally replace with RTOS semaphore
}

/* --- LCD Initialization (Simplified ST7735 example) --- */
void LCD_Init(void)
{
    HAL_Delay(10);
    LCD_SendCommand(0x01); // Software Reset
    HAL_Delay(120);
    LCD_SendCommand(0x11); // Sleep Out
    HAL_Delay(120);
    LCD_SendCommand(0x29); // Display ON
}

/* --- Draw a full frame using DMA --- */
void LCD_DrawFrameDMA(uint16_t *buffer)
{
    uint8_t cmd = 0x2C; // Memory Write
    LCD_SendCommand(cmd);

    // Transmit pixel data (2 bytes per pixel)
    LCD_SendDataDMA((uint8_t *)buffer, LCD_WIDTH * LCD_HEIGHT * 2);
    LCD_WaitForDMA();
}

/* --- Example usage --- */
int main(void)
{
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();
    MX_DMA_Init();
    MX_SPI1_Init();

    LCD_Init();

    // Fill framebuffer with red pixels
    for (int i = 0; i < LCD_WIDTH * LCD_HEIGHT; i++)
        frameBuffer[i] = 0xF800; // RGB565 Red

    // Send frame using DMA (non-blocking SPI transfer)
    LCD_DrawFrameDMA(frameBuffer);

    while (1)
    {
        // CPU is free to do other things
        HAL_Delay(1000);

        // Change frame color
        static uint16_t color = 0x07E0; // Green
        for (int i = 0; i < LCD_WIDTH * LCD_HEIGHT; i++)
            frameBuffer[i] = color;

        LCD_DrawFrameDMA(frameBuffer);

        color = (color == 0x07E0) ? 0x001F : 0x07E0; // Toggle color
    }
}
```

---

# 🔍 Step 4: How This Works

| Step | Operation                                                                         |
| ---- | --------------------------------------------------------------------------------- |
| 1️⃣  | You call `LCD_DrawFrameDMA()`                                                     |
| 2️⃣  | SPI + DMA begin transferring your entire frame buffer                             |
| 3️⃣  | CPU immediately returns to the main loop — can compute next frame or update logic |
| 4️⃣  | When DMA finishes, interrupt fires → `HAL_SPI_TxCpltCallback()` runs              |
| 5️⃣  | LCD chip select released, frame complete                                          |

This is how **DMA enables full-frame refresh without blocking CPU**.

---

# ⚡ Step 5: Performance Benefit

| Method       | Approx. 128×160 Frame Update Time @ 8 MHz SPI             |     |
| ------------ | --------------------------------------------------------- | --- |
| Blocking SPI | ~80 ms (CPU busy entire time)                             |     |
| SPI + DMA    | ~80 ms (same transfer time, but CPU free during transfer) |     |
| CPU Load     | ~100%                                                     | ~5% |

The **transfer duration** doesn’t change (it’s hardware-limited by SPI clock),
but **CPU availability** does — that’s the entire benefit.

---

# 💡 Step 6: Integration Tips

✅ You can double-buffer the frame (one shown, one updated).
✅ You can synchronize DMA completion with an RTOS task using a semaphore.
✅ You can use DMA for both **TX and RX** (for SPI sensors).
✅ Same logic applies to **I2C-based displays** (slower, but same pattern).

---

# 🧠 Summary

| Feature    | SPI (blocking)        | SPI + DMA                    |
| ---------- | --------------------- | ---------------------------- |
| CPU Load   | High                  | Very low                     |
| Speed      | Limited by SPI        | Same SPI speed               |
| Efficiency | Poor for large frames | Excellent                    |
| Use Case   | Simple commands       | LCD / sensors / memory chips |

---

If you’d like, next we can go one layer deeper and discuss **double-buffering with DMA** (to update one frame while DMA transmits another), which is used in **smooth LCD animation** or **real-time UI** systems.

Would you like to continue to that next (DMA double-buffered LCD updates)?
