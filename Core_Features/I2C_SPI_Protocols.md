# 1) I²C — theory (concise, practical)

* **What it is:** Two-wire, multi-master, multi-slave synchronous serial bus. Lines: **SDA** (data) and **SCL** (clock). Both are open-drain (or open-collector) and require pull-ups.
* **Bit-level:** Data valid on SDA while SCL is high. Devices pull the line low to transmit a 0; release it for 1.
* **Transactions (master initiates):**

  * **START**: SDA goes low while SCL high.
  * **Address + R/W bit**: 7-bit (or 10-bit) address then R/W (0 write, 1 read).
  * **ACK/NACK**: Receiver pulls SDA low to ACK after each byte; otherwise NACK.
  * **Data bytes**: Sent MSB first.
  * **STOP**: SDA goes high while SCL high.
* **Clock stretching:** Slave may hold SCL low if it needs extra time — master must respect this.
* **7-bit vs 10-bit addressing:** 7-bit most common. 10-bit used for more devices.
* **Speeds:** Standard (100 kHz), Fast (400 kHz), Fast+ (1 MHz), High Speed (3.4 MHz) — hardware dependent.
* **Use cases:** Sensors, EEPROMs, ADCs/DACs, small peripherals when only 2 wires desired.
* **Pros/Cons:**

  * Pros: simple wiring, many devices on 2 pins, addressing, multi-master capable.
  * Cons: slower than SPI, more complex bus arbitration, pull-ups required, possible bus hung if devices hold lines.

# 2) I²C — STM32 practical (HAL examples)

Assume: STM32Cube HAL, `I2C_HandleTypeDef hi2c1` configured (I2C1 pins, pullups, timing set in CubeMX or manually).

## 2.1 Initialize (CubeMX / HAL)

CubeMX will generate `MX_I2C1_Init()` that sets timing, addressing mode, own address, etc. Example snippet (auto-generated style):

```c
I2C_HandleTypeDef hi2c1;

void MX_I2C1_Init(void)
{
  hi2c1.Instance = I2C1;
  hi2c1.Init.Timing = 0x00707CBB; // example timing value — use CubeMX for exact
  hi2c1.Init.OwnAddress1 = 0;
  hi2c1.Init.AddressingMode = I2C_ADDRESSINGMODE_7BIT;
  hi2c1.Init.DualAddressMode = I2C_DUALADDRESS_DISABLE;
  hi2c1.Init.OwnAddress2 = 0;
  hi2c1.Init.GeneralCallMode = I2C_GENERALCALL_DISABLE;
  hi2c1.Init.NoStretchMode = I2C_NOSTRETCH_DISABLE;
  HAL_I2C_Init(&hi2c1);
  // Configure analog/digital filter if needed
  HAL_I2CEx_ConfigAnalogFilter(&hi2c1, I2C_ANALOGFILTER_ENABLE);
}
```

**Important:** timing register must be calculated for your clock — use CubeMX.

## 2.2 Master write (blocking)

Write `N` bytes to a 7-bit slave address:

```c
uint8_t data[] = {0x10, 0x20, 0x30};
uint16_t devAddr = 0x50 << 1; // HAL uses 8-bit address (7-bit << 1)
if(HAL_I2C_Master_Transmit(&hi2c1, devAddr, data, sizeof(data), HAL_MAX_DELAY) != HAL_OK) {
    // error handling
}
```

Note: HAL expects the 8-bit address (7-bit << 1). Use appropriate timeout.

## 2.3 Master read (blocking)

Read N bytes:

```c
uint8_t buf[16];
uint16_t devAddr = 0x50 << 1;
if(HAL_I2C_Master_Receive(&hi2c1, devAddr, buf, 16, HAL_MAX_DELAY) != HAL_OK) {
    // error
}
```

## 2.4 Common pattern: write register then read

Many sensors use reg pointer then read:

```c
uint8_t reg = 0x0A;
uint8_t val;
HAL_I2C_Master_Transmit(&hi2c1, devAddr, &reg, 1, HAL_MAX_DELAY);
HAL_I2C_Master_Receive(&hi2c1, devAddr, &val, 1, HAL_MAX_DELAY);
// Or use HAL_I2C_Mem_Read shorthand:
HAL_I2C_Mem_Read(&hi2c1, devAddr, reg, I2C_MEMADD_SIZE_8BIT, &val, 1, HAL_MAX_DELAY);
```

## 2.5 I2C bus scan (useful debug)

```c
for(uint8_t addr = 1; addr < 0x7F; addr++) {
  if(HAL_I2C_IsDeviceReady(&hi2c1, addr<<1, 3, 50) == HAL_OK) {
    // device present at addr
  }
}
```

## 2.6 Notes on interrupts / DMA

* Use `HAL_I2C_Master_Transmit_IT` and callbacks `HAL_I2C_MasterTxCpltCallback` for non-blocking.
* Use DMA for large transfers: `HAL_I2C_Master_Transmit_DMA`.
* Handle NACKs and BUS errors; implement recovery (e.g., re-init or toggling SCL pulses) if bus stuck.

---

# 3) SPI — theory (concise)

* **What it is:** Synchronous full-/half-duplex bus. Typically 4 signals:

  * **SCLK** (clock)
  * **MOSI** (master out / slave in)
  * **MISO** (master in / slave out)
  * **NSS/CS** (chip select, active low) — one per slave (unless daisy/chaining).
* **Modes (CPOL/CPHA):**

  * CPOL = clock polarity (idle low or high).
  * CPHA = clock phase (sample on first or second clock edge).
  * Modes 0..3 map to combinations:

    * Mode0: CPOL=0, CPHA=0
    * Mode1: CPOL=0, CPHA=1
    * Mode2: CPOL=1, CPHA=0
    * Mode3: CPOL=1, CPHA=1
* **Clock mastership:** Master generates clock. No addressing — chip selected by CS.
* **Data order:** MSB-first or LSB-first configurable.
* **Speeds:** Much faster than I2C — depends on MCU peripheral and wiring.
* **Pros/Cons:**

  * Pros: fast, simple protocol, full duplex.
  * Cons: needs separate CS per slave (wire cost), no built-in flow control or ACK, not multi-master.

# 4) SPI — STM32 practical (HAL examples)

Assume `SPI_HandleTypeDef hspi1` configured.

## 4.1 Init (CubeMX/HAL)

```c
SPI_HandleTypeDef hspi1;

void MX_SPI1_Init(void)
{
  hspi1.Instance = SPI1;
  hspi1.Init.Mode = SPI_MODE_MASTER;
  hspi1.Init.Direction = SPI_DIRECTION_2LINES; // full-duplex
  hspi1.Init.DataSize = SPI_DATASIZE_8BIT;
  hspi1.Init.CLKPolarity = SPI_POLARITY_LOW;   // CPOL = 0
  hspi1.Init.CLKPhase = SPI_PHASE_1EDGE;       // CPHA = 0 -> Mode 0
  hspi1.Init.NSS = SPI_NSS_SOFT;               // Software-managed CS
  hspi1.Init.BaudRatePrescaler = SPI_BAUDRATEPRESCALER_16;
  hspi1.Init.FirstBit = SPI_FIRSTBIT_MSB;
  hspi1.Init.CRCCalculation = SPI_CRCCALCULATION_DISABLE;
  HAL_SPI_Init(&hspi1);
}
```

Pick `BaudRatePrescaler` to set SPI freq relative to APB clock.

## 4.2 Master transmit / receive (blocking)

Full-duplex transfer (send and get simultaneously):

```c
uint8_t tx[] = {0x9F}; // example: JEDEC ID command for SPI flash
uint8_t rx[3];
HAL_GPIO_WritePin(CS_GPIO_Port, CS_Pin, GPIO_PIN_RESET); // assert CS
HAL_SPI_TransmitReceive(&hspi1, tx, rx, 1, HAL_MAX_DELAY);
HAL_GPIO_WritePin(CS_GPIO_Port, CS_Pin, GPIO_PIN_SET); // deassert CS
```

For longer transfers use length parameter accordingly.

## 4.3 Simple write-only

```c
uint8_t buf[] = {0x01, 0x02, 0x03};
HAL_GPIO_WritePin(CS_GPIO_Port, CS_Pin, GPIO_PIN_RESET);
HAL_SPI_Transmit(&hspi1, buf, sizeof(buf), HAL_MAX_DELAY);
HAL_GPIO_WritePin(CS_GPIO_Port, CS_Pin, GPIO_PIN_SET);
```

## 4.4 Read from a device (command then read)

For SPI flash JEDEC ID:

```c
uint8_t cmd = 0x9F;
uint8_t id[3];
HAL_GPIO_WritePin(CS_GPIO_Port, CS_Pin, GPIO_PIN_RESET);
HAL_SPI_Transmit(&hspi1, &cmd, 1, HAL_MAX_DELAY);
HAL_SPI_Receive(&hspi1, id, 3, HAL_MAX_DELAY);
HAL_GPIO_WritePin(CS_GPIO_Port, CS_Pin, GPIO_PIN_SET);
```

Or combine with TransmitReceive for simultaneous clocking.

## 4.5 Half-duplex and simplex

* Some devices or MCUs support 1-line (half duplex) SPI — check CubeMX setting `SPI_DIRECTION_1LINE`.

## 4.6 Interrupts / DMA

* Use `HAL_SPI_Transmit_IT` / `HAL_SPI_Receive_IT` with callback `HAL_SPI_TxCpltCallback`.
* DMA via `HAL_SPI_Transmit_DMA` for high throughput.

---

# 5) Concrete real-device example sketches (useful patterns)

## 5.1 I²C EEPROM ([24LC256](https://www.microchip.com/en-us/product/24lc256)) — write then read

```c
#define EEPROM_ADDR (0x50<<1)

uint16_t memAddress = 0x0123;
uint8_t dataToWrite[2] = {0xAA, 0x55};
// For 24LC256, we send 2-byte memory address MSB first
uint8_t tx[2 + sizeof(dataToWrite)];
tx[0] = (memAddress >> 8) & 0xFF;
tx[1] = memAddress & 0xFF;
memcpy(&tx[2], dataToWrite, sizeof(dataToWrite));

HAL_I2C_Master_Transmit(&hi2c1, EEPROM_ADDR, tx, sizeof(tx), HAL_MAX_DELAY);
// Wait write cycle ~5ms (polling is better)
HAL_Delay(10);

// Read back
uint8_t addrBytes[2] = { (memAddress>>8)&0xFF, memAddress&0xFF };
uint8_t buf[2];
HAL_I2C_Master_Transmit(&hi2c1, EEPROM_ADDR, addrBytes, 2, HAL_MAX_DELAY);
HAL_I2C_Master_Receive(&hi2c1, EEPROM_ADDR, buf, 2, HAL_MAX_DELAY);
```

## 5.2 SPI flash (W25Qxx) — read JEDEC ID

```c
uint8_t cmd = 0x9F;
uint8_t id[3];
HAL_GPIO_WritePin(CS_GPIO_Port, CS_Pin, GPIO_PIN_RESET);
HAL_SPI_TransmitReceive(&hspi1, &cmd, id, 1, HAL_MAX_DELAY); // first byte clocks one response byte in id[0]
HAL_SPI_Receive(&hspi1, &id[1], 2, HAL_MAX_DELAY); // read remaining 2 bytes
HAL_GPIO_WritePin(CS_GPIO_Port, CS_Pin, GPIO_PIN_SET);
```

(Or simpler: transmit 0x9F then read 3 bytes by sending 0x00s.)

---

# 6) Practical tips & pitfalls

* **I²C:**

  * Always use pull-ups sized for bus capacitance. Typical 2.2k–10k. Faster speeds need stronger pull-ups.
  * Beware of mixing devices at different voltages — use level shifters if needed.
  * If bus stuck (SDA or SCL low), try toggling SCL manually to release a hung slave.
  * Check address confusion: some datasheets print 7-bit address; HAL often expects 8-bit (<<1).
* **SPI:**

  * Match CPOL/CPHA and bit order with slave datasheet.
  * Drive CS low before command and release after required clocks — many bugs from wrong CS timing.
  * Minimize CS toggles where device expects multi-byte command.
* **Timing:** Use CubeMX to compute I²C timing values for your MCU clock. For SPI desired frequency pick appropriate prescaler (APB clock / prescaler).

---
