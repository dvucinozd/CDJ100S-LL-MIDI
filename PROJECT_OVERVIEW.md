# CDJ-100S MIDI Adapter - Project Overview

## 📁 Location
`D:\AI\CDJ100\CDJ100-MIDI`

## 🔗 Original Repository
[spectran/CDJ-100S-MIDI-Adapter](https://github.com/spectran/CDJ-100S-MIDI-Adapter)

---

## Ključni Nalazi - Analiza Koda

### Hardware Pinout (STM32F103C8T6)

#### SPI1 - Komunikacija sa Display Boardom
```
PA4 - SPI1_NSS  (Chip select)
PA5 - SPI1_SCK  (Clock) → STM32F746 PI1
PA6 - SPI1_MISO (← Display board) → STM32F746 PB14
PA7 - SPI1_MOSI (→ Display board) → STM32F746 PB15
```

#### CDJ-100S Input Signals
```
# Keyboard Matrix (3x5)
PB3  - KD0 (Keyboard data 0)
PB4  - KD1 (Keyboard data 1)
PB5  - KD2 (Keyboard data 2)
PB12 - S3  (Scan line 3)
PB13 - S4  (Scan line 4)
PB14 - S5  (Scan line 5)
PB15 - S1  (Scan line 1)
PA8  - S2  (Scan line 2)

# Jog Wheel Encoder (TIM4)
PB6  - JOG1 (TIM4_CH1 - Encoder A)
PB7  - JOG2 (TIM4_CH2 - Encoder B)

# Analog Inputs (ADC1)
PA0  - ADIN (Pitch slider 0-3.3V)
PA1  - CT   (Control timing signal)

# LED Outputs (read-only na CDJ-100S)
PB2  - CUE  LED
PB10 - PLAY LED
PB11 - DISC LED
```

#### USB MIDI
```
PA11 - USB_DM (USB Data Minus)
PA12 - USB_DP (USB Data Plus)
PA10 - USB_VBUS (USB detection)
PB9  - USB_DISCONNECT (soft disconnect)
```

---

## SPI Protokol - Detaljna Analiza

### Main Loop ([`main.c:212-218`](file:///D:/AI/CDJ100/CDJ100-MIDI/Src/main.c#L212-L218))

```c
while (1) {
  Keyboard_Read();      // Čita sve tipke (keyboard matrix scan)
  HAL_Delay(25);
  Display_DataRx();     // Prima LED status od Display boarda
  HAL_Delay(25);
}
```

**Polling Rate**: 20 Hz (50ms cycle time)

### SPI Transmit - Ka Display Boardu

**Funkcija**: `Display_DataTx()` ([main.c:539-543](file:///D:/AI/CDJ100/CDJ100-MIDI/Src/main.c#L539-L543))

```c
void Display_DataTx(uint8_t *data, uint16_t length) {
  HAL_GPIO_WritePin(SPI1_NSS_GPIO_Port, SPI1_NSS_Pin, GPIO_PIN_RESET);  // CS low
  HAL_SPI_Transmit(&hspi1, data, length, 0x1000);                       // Send
  HAL_GPIO_WritePin(SPI1_NSS_GPIO_Port, SPI1_NSS_Pin, GPIO_PIN_SET);    // CS high
}
```

**Packet Examples** (iz koda):
```c
// Button press events
uint8_t Hold_On[4]  = {0x08, 0x90, 0x40, 0x47};  // HOLD button pressed
uint8_t Play_On[4]  = {0x08, 0x90, 0x49, 0x47};  // PLAY button pressed

// Button release events  
uint8_t Hold_Off[4] = {0x08, 0x80, 0x40, 0x47};  // HOLD button released
uint8_t Play_Off[4] = {0x08, 0x80, 0x49, 0x47};  // PLAY button released

// Pitch bend
uint8_t Pitch[4]    = {0x08, 0xE0, 0x00, 0x00};  // Bytes 2-3 for pitch value
```

### SPI Receive - Od Display Boarda

**Funkcija**: `Display_DataRx()` ([main.c:546-574](file:///D:/AI/CDJ100/CDJ100-MIDI/Src/main.c#L546-L574))

```c
void Display_DataRx() {
  uint8_t dummy_data[4] = {0};
  HAL_SPI_TransmitReceive(&hspi1, dummy_data, spi_rx_buffer, 4, 0x1000);
  
  // Update LEDs based on display board status
  if ((spi_rx_buffer[1] & 0xF0) == 0x90) {
    // Bit 0: PLAY LED
    if (spi_rx_buffer[2] & (1 << 0)) {
      HAL_GPIO_WritePin(PLAY_GPIO_Port, PLAY_Pin, GPIO_PIN_SET);
    }
    // Bit 1: CUE LED
    if (spi_rx_buffer[2] & (1 << 1)) {
      HAL_GPIO_WritePin(CUE_GPIO_Port, CUE_Pin, GPIO_PIN_SET);
    }
    // Bit 2: DISC LED
    if (spi_rx_buffer[2] & (1 << 2)) {
      HAL_GPIO_WritePin(DISC_GPIO_Port, DISC_Pin, GPIO_PIN_SET);
    }
  }
}
```

**LED Feedback**: Display board vraća status koji upravlja LED diodama na CDJ-100S hardveru.

---

## Keyboard Matrix Scanning

### Scan Pattern ([main.c:273-536](file:///D:/AI/CDJ100/CDJ100-MIDI/Src/main.c#L273-L536))

MIDI adapter koristi **keyboard matrix** tehniku:

1. **Aktivira S1** (set high) → čita KD0-2
2. **Aktivira S2** (set high) → čita KD0-2
3. **Aktivira S3** (set high) → čita KD0-2
4. **Aktivira S4** (set high) → čita KD0-2
5. **Aktivira S5** (set high) → čita KD0-2

**Ukupno**: 3 x 5 = **15 buttons** skeniranih

### Button Mapping

| Scan Line | KD0 | KD1 | KD2 |
|-----------|-----|-----|-----|
| S1 | HOLD | TRACKBACK | PLAY |
| S2 | TIME | TRACKFORWARD | CUE |
| S3 | EJECT | JET | SCANBACK |
| S4 | MASTERTEMPO | ZIP | SCANFORWARD |
| S5 | JOG | WAH | - |

---

## USB MIDI Funkcionalnost

### MIDI Output

Kod šalje **duale pakete** na svaki button event:
1. **Display board** (via SPI) - za audio player kontrolu
2. **PC** (via USB MIDI) - za DAW/VirtualDJ

**Primjer** ([main.c:278-281](file:///D:/AI/CDJ100/CDJ100-MIDI/Src/main.c#L278-L281)):
```c
if (usb_status == CONNECTED) {
  MIDI_DataTx(Hold_On, 4);      // USB MIDI → PC
}
Display_DataTx(Hold_On, 4);     // SPI → Display board
```

### USB Detection ([main.c:194-206](file:///D:/AI/CDJ100/CDJ100-MIDI/Src/main.c#L194-L206))

```c
if (HAL_GPIO_ReadPin(USB_VBUS_GPIO_Port, USB_VBUS_Pin) == GPIO_PIN_RESET) {
  usb_status = DISCONNECTED;
} else {
  usb_status = CONNECTED;
  MX_USB_DEVICE_Init();
}
```

---

## Jog Wheel (Encoder)

### Hardware: TIM4 Enkoder Mode

**Config**: TIM4 u quadrature encoder mode
- **Channel 1** (PB6): Encoder A signal
- **Channel 2** (PB7): Encoder B signal

**Interrupt**: `TIM4_IRQHandler` (u [`stm32f1xx_it.c`](file:///D:/AI/CDJ100/CDJ100-MIDI/Src/stm32f7xx_it.c))

**Data**: Encoder count se šalje kao **pitch bend** MIDI message ili custom SPI paket.

---

## ADC - Pitch Slider

### Konfiguracija

**Periferija**: ADC1, Channel 0 (PA0)  
**Resolution**: 12-bit (0-4095)  
**Trigger**: TIM3 (periodični trigger)  
**Mode**: DMA Continuous

**Kod** ([main.c:187-188](file:///D:/AI/CDJ100/CDJ100-MIDI/Src/main.c#L187-L188)):
```c
HAL_ADCEx_Calibration_Start(&hadc1);
HAL_ADC_Start_DMA(&hadc1, (uint32_t*)&adc_data, 200);
```

**Buffer**: 200 samples za averaging

---

## Build & Flash

### STM32CubeIDE

1. **Open Project**
   ```
   File → Open Projects from File System
   → Select: D:\AI\CDJ100\CDJ100-MIDI
   ```

2. **Build**
   ```
   Project → Build Project (Ctrl+B)
   ```

3. **Flash**
   ```
   Run → Debug (F11)
   ```

### ST-Link Connection
```
ST-Link → Blue Pill
SWDIO  → PA13 (SWDIO)
SWCLK  → PA14 (SWCLK)
GND    → GND
3.3V   → 3.3V
```

---

## Kompatibilnost sa Display Boardom (STM32F746)

### ✅ Pin Matching

| MIDI Adapter (STM32F103) | Display Board (STM32F746) | Signal |
|--------------------------|----------------------------|--------|
| PA5 (SPI1_SCK) | PI1 (SPI2_SCK) | Clock |
| PA7 (SPI1_MOSI) | PB15 (SPI2_MOSI) | Data → |
| PA6 (SPI1_MISO) | PB14 (SPI2_MISO) | Data ← |

### ✅ Protocol Matching

**Packet Structure**: Obje strane koriste 4-byte pakete  
**Command Codes**: Definirani u [`stm32f7xx_it.h`](file:///d:/AI/CDJ100/Inc/stm32f7xx_it.h) na Display boardu

### ⚠️ SPI Mode

**MIDI Adapter**: SPI1 **Master**  
**Display Board**: SPI2 **Slave**

**Clock Speed**: Podesiti na 2-4 Mbps za stabilnu komunikaciju

---

## Sljedeći Koraci

1. ✅ **Repozitorij kloniran** - Gotovo
2. ✅ **Kod analiziran** - Gotovo
3. 🔲 **Hardware testiranje** - Potrebno:
   - Blue Pill board
   - ST-Link programmer
   - CDJ-100S Display board za testiranje
4. 🔲 **Flash firmware** na Blue Pill
5. 🔲 **Testirati SPI komunikaciju** sa Display boardom
6. 🔲 **Verification** na hardveru

---

## Reference Files

- **Main Application**: [`main.c`](file:///D:/AI/CDJ100/CDJ100-MIDI/Src/main.c)
- **SPI Config**: [`spi.c`](file:///D:/AI/CDJ100/CDJ100-MIDI/Src/spi.c)
- **ADC Config**: [`adc.c`](file:///D:/AI/CDJ100/CDJ100-MIDI/Src/adc.c)
- **TIM Config**: [`tim.c`](file:///D:/AI/CDJ100/CDJ100-MIDI/Src/tim.c)
- **USB MIDI**: [`usbd_midi_if.c`](file:///D:/AI/CDJ100/CDJ100-MIDI/Src/usbd_midi_if.c)
- **Schematics**: [`Schematics/Connection_scheme.pdf`](file:///D:/AI/CDJ100/CDJ100-MIDI/Schematics/Connection_scheme.pdf)

---

**Status**: ✅ Projekt pripremljen za hardware implementaciju!
