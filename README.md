# CDJ-100S MIDI Adapter - Blue Pill (STM32F103C8T6)

## Pregled

MIDI adapter za Pioneer CDJ-100S DJ konzolu baziran na **Blue Pill (STM32F103C8T6)**. Čita hardverske kontrole (jog wheel, pitch slider, tipke) i šalje komande **STM32F746 Display boardu** preko SPI2 komunikacije.

**Fork od**: [spectran/CDJ-100S-MIDI-Adapter](https://github.com/spectran/CDJ-100S-MIDI-Adapter)

---

## Ključne Značajke

- ✅ **SPI Master Mode** - komunikacija sa Display boardom
- 🎛️ **Keyboard Matrix** - 3x5 = 15 buttons
- 🎚️ **Jog Wheel** - TIM4 Encoder mode (quadrature)
- 📐 **Pitch Slider** - ADC1 (0-3.3V analog input)
- 🎵 **USB MIDI** - paralelni output za PC/DAW (opciono)
- 💡 **LED Feedback** - prima status od Display boarda

---

## Hardware

### Komponente

| Dio | Specifikacija |
|-----|---------------|
| **Mikrokontroler** | STM32F103C8T6 (Blue Pill) |
| **Flash** | 64KB |
| **RAM** | 20KB |
| **Clock** | 72MHz (HSE 8MHz crystal) |
| **Programmer** | ST-Link v2 (SWD) |

### Pinout - Ključni Signali

#### SPI1 → Display Board (STM32F746)
```
PA4 - SPI1_NSS  (Chip select)
PA5 - SPI1_SCK  → PI1 (Display board)
PA6 - SPI1_MISO ← PB14 (Display board)
PA7 - SPI1_MOSI → PB15 (Display board)
```

#### CDJ-100S Input
```
# Keyboard Matrix
PB3-5   - KD0-2 (Keyboard data)
PB12-15 - S3-5, S1 (Scan lines)
PA8     - S2 (Scan line)

# Jog Wheel
PB6 - JOG1 (Encoder A, TIM4_CH1)
PB7 - JOG2 (Encoder B, TIM4_CH2)

# Pitch Slider
PA0 - ADIN (ADC1_IN0, 0-3.3V)

# LEDs (output)
PB2  - CUE LED
PB10 - PLAY LED
PB11 - DISC LED
```

#### USB MIDI (opciono)
```
PA11 - USB_DM
PA12 - USB_DP
PA10 - USB_VBUS (detection)
```

**⚠️ VAŽNO**: Pitch slider mora imati **3.3V max** (ne 5V)!

---

## SPI Protokol

### Packet Format (4 bajta)

```
┌────────┬─────────┬──────────┬─────────┐
│ Byte 0 │ Byte 1  │ Byte 2   │ Byte 3  │
│ 0x08   │ Command │ Data1    │ Data2   │
└────────┴─────────┴──────────┴─────────┘
```

### Command Types (Byte 1)

| Command | Značenje |
|---------|----------|
| `0x90` | Button Press |
| `0x80` | Button Release |
| `0xE0` | Pitch Bend |

### Button IDs (Byte 2)

| ID | Button | Funkcija |
|----|--------|----------|
| `0x40` | HOLD | Hold mode |
| `0x41` | TIME | Time display |
| `0x43` | MASTERTEMPO | Master tempo |
| `0x44` | TRACKBACK | Previous track |
| `0x45` | TRACKFORWARD | Next track |
| `0x46` | JET | Loop start |
| `0x47` | ZIP | Loop end |
| `0x48` | WAH | Effect/hot cue |
| `0x49` | PLAY | Play/Pause |
| `0x4A` | CUE | Cue point |
| `0x4B` | SCANBACK | Fast rewind |
| `0x4C` | SCANFORWARD | Fast forward |
| `0x4D` | JOG | Jog wheel press |

---

## Build Instrukcije

### Zahtjevi

- **STM32CubeIDE** (verzija 1.8+)
- **ST-Link v2** programmer
- **Blue Pill** board
- **Git**

### Kloniranje

```bash
git clone https://github.com/dvucinozd/CDJ100S-LL-MIDI.git
cd CDJ100S-LL-MIDI
```

### Build

#### PlatformIO (Preporučeno)

Projekt je konfiguriran za **PlatformIO** koji omogućava jednostavan build bez instalacije STM32CubeIDE-a.

1. **Otvori Folder** u VS Code (sa PlatformIO ekstenzijom)
2. **Build**:
   - Klikni na `Build` ikonu (kvačica) u donjem lijevom kutu
   - Ili u terminalu: `pio run`
3. **Upload**:
   - Spoji Blue Pill preko ST-Linka
   - Klikni na `Upload` ikonu (strelica)
   - Ili u terminalu: `pio run -t upload`

#### STM32CubeIDE

1. **Import Projekt**
   ```
   File → Open Projects from File System
   → Select CDJ100S-LL-MIDI folder
   ```

2. **Build**
   ```
   Project → Build Project (Ctrl+B)
   ```

3. **Flash**
   ```
   Run → Debug (F11)
   ```

#### Command Line (alternative)

```bash
# Build release version
arm-none-eabi-gcc -mcpu=cortex-m3 -std=gnu11 -DUSE_HAL_DRIVER \
  -DSTM32F103xB -c -O2 -ffunction-sections -o firmware.elf ...

# Flash sa ST-Link
st-flash write firmware.bin 0x08000000
```

---

## Hardware Setup

### 1. ST-Link Connection

```
ST-Link → Blue Pill
┌────────┬─────────┐
│ SWDIO  │ PA13    │
│ SWCLK  │ PA14    │
│ GND    │ GND     │
│ 3.3V   │ 3.3V    │
└────────┴─────────┘
```

### 2. CDJ-100S Display Board Connection

Prema `Schematics/Connection_scheme.pdf`:

- Spoji **V+5V, GNDD** sa Trans boarda
- Povezi **JOG1-2, S1-S5, KD0-2** signale
- **ADIN** pitch slider (⚠️ 3.3V max!)
- **CUE, PLAY, DISC** LED signali

### 3. Display Board (STM32F746) Connection

```
Blue Pill → Display Board
┌─────────┬──────────┐
│ PA5     │ PI1      │ SPI Clock
│ PA7     │ PB15     │ MOSI
│ PA6     │ PB14     │ MISO
│ GND     │ GND      │ Ground
└─────────┴──────────┘
```

---

## Testing

### 1. Power Test
```c
// LED blink test
HAL_GPIO_TogglePin(LED_GPIO_Port, LED_Pin);
HAL_Delay(500);
```

### 2. SPI Loopback Test
```c
// Short MOSI to MISO (PA7 ↔ PA6)
uint8_t tx[4] = {0x08, 0x90, 0x49, 0x47};
uint8_t rx[4];
HAL_SPI_TransmitReceive(&hspi1, tx, rx, 4, 1000);
// Check: rx == tx
```

### 3. Button Test
- Spoj S1 sa KD0 → trebao bi slati HOLD event
- Provjeri serial output ili USB MIDI stream

### 4. Integration Test
- Spoji sa Display boardom
- Pritisni PLAY tipku na CDJ-100S
- Display board bi trebao primiti `0x90 0x49` paket

---

## Troubleshooting

### Problem: Blue Pill se ne flasha

**Rješenje:**
- Provjeri BOOT0 jumper (mora biti na 0)
- Reset board prije flash-a
- Provjeri ST-Link konekciju (SWDIO/SWCLK)

### Problem: SPI ne šalje pakete

**Rješenje:**
- Provjeri NSS (CS) pin - mora toggle-ati
- Smanjiti SPI brzinu (probaj 2 Mbps)
- Provjeri GND konekciju

### Problem: Display board ne prima podatke

**Rješenje:**
- Provjeri da Display board SPI2 radi kao Slave
- Provjeri clock polarity/phase (CPOL/CPHA)
- Koristi logic analyzer za debug

---

## Dokumentacija

- 📄 [PROJECT_OVERVIEW.md](PROJECT_OVERVIEW.md) - Detaljna analiza koda
- 📁 [Schematics/](Schematics/) - Električne sheme i PCB layout
- 🎥 [YouTube Demo](https://www.youtube.com/watch?v=wtE2o-FcW-4)

---

## Roadmap

- [ ] **PCB Design** - Custom board umjesto Blue Pill
- [ ] **USB MIDI Testing** - Verifikacija sa DAW softverom
- [ ] **LED Brightness** - PWM kontrola LED dioda
- [ ] **Pitch Quantization** - Software debouncing

---

## Licenca i Autori

**Original**: [spectran/CDJ-100S-MIDI-Adapter](https://github.com/spectran/CDJ-100S-MIDI-Adapter)  
**Autor**: Ruslan Terentiev (spectran)

**Fork**: [dvucinozd/CDJ100S-LL-MIDI](https://github.com/dvucinozd/CDJ100S-LL-MIDI)

---

## Reference

- [STM32F103C8T6 Datasheet](https://www.st.com/resource/en/datasheet/stm32f103c8.pdf)
- [Blue Pill Pinout](https://stm32-base.org/boards/STM32F103C8T6-Blue-Pill.html)
- [Display Board Repozitorij](https://github.com/dvucinozd/CDJ100S-LL)
- [Connection Scheme PDF](Schematics/Connection_scheme.pdf)
