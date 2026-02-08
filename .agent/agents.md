# CDJ-100S MIDI Adapter - AI Agent Development Guide

## Pregled Projekta

**Tip**: MIDI Adapter / Bridge za Pioneer CDJ-100S
**MCU**: STM32F103C8T6 (Blue Pill) - ARM Cortex-M3, 72MHz
**Uloga**: Čita hardverske kontrole i šalje ih Display boardu (SPI) i PC-u (USB MIDI).
**OS**: Bare Metal (HAL Library)

---

## Arhitektura

```
┌───────────────────────────┐
│  Main Loop (main.c)       │
│  - Keyboard_Read()        │
│  - Display_DataRx()       │
├───────────────────────────┤
│  Peripherals (HAL)        │
│  - SPI1 (Master)          │
│  - ADC1 (Pitch Slider)    │
│  - TIM4 (Jog Encoder)     │
│  - USB Device (MIDI Class)│
└───────────────────────────┘
```

---

## Struktura Direktorija

```
CDJ100S-LL-MIDI/
├── .agent/                   # 👈 Agenti dokumentacija
├── Drivers/                  # STM32F1xx HAL Drivers
├── Inc/                      # Headers
├── Middlewares/              # USB Device Library
├── Src/                      # Source Code
│   ├── main.c                # Glavna logika
│   ├── spi.c                 # SPI konfiguracija
│   ├── usbd_midi_if.c        # USB MIDI interface
│   └── ...
├── platformio.ini            # PlatformIO Config (PREPORUČENO)
└── CDJ_Control_C8.ioc        # STM32CubeMX Config
```

---

## Build Workflow

### PlatformIO (Primarni)

Koristimo `platformio.ini` koji je konfiguriran da koristi lokalne HAL drivere (ne framework).

```bash
# Build
pio run

# Upload (ST-Link)
pio run -t upload
```

### STM32CubeIDE (Sekundarni)

Projekt je originalno generiran u CubeIDE. Može se otvoriti preko `.cproject`.

---

## Ključni Moduli

1.  **SPI Komunikacija (`spi.c`, `main.c`)**
    *   **Master Mode**: Generira clock za Display board.
    *   **Protokol**: 4-byte paketi `[0x08, Command, Data1, Data2]`.
    *   **TX**: `Display_DataTx()` šalje komande.
    *   **RX**: `Display_DataRx()` prima LED status.

2.  **Keyboard Matrix (`main.c`)**
    *   Skenira 3x5 matricu (S1-S5 scan lines, KD0-2 data lines).
    *   Detektira pritiske i šalje MIDI/SPI evente.

3.  **Jog Wheel (`tim.c`)**
    *   Koristi TIM4 u Encoder modu (PB6, PB7).
    *   Interrupti nisu nužni za basic counting, ali se koriste za preciznost.

4.  **Pitch Slider (`adc.c`)**
    *   ADC1 Channel 0 (PA0).
    *   DMA transfer u kružni buffer.

5.  **USB MIDI (`usbd_midi_if.c`)**
    *   Standardna USB Audio Class (MIDI subclass).
    *   Radi paralelno sa SPI komunikacijom.

---

## Development Pravila

*   **Napajanje**: Pitch slider na CDJ-100S radi na 5V, ali STM32F103 ADC je 3.3V! **OBAVEZNO** provjeri voltage divider ili modifikaciju napajanja prije spajanja.
*   **Pinout**: Definiran u `main.h` i CubeMX `.ioc` fajlu. Ne mijenjaj pinove bez provjere `Connection_scheme.pdf`.
*   **Kod**: `main.c` je prilično velik (monolitan). Kod refaktoringa, pazi da ne slomiš timing SPI komunikacije.

---

## Debugging

*   **SPI Problemi**: Provjeri GND vezu između Blue Pill i Display boarda. Provjeri CPOL/CPHA settings.
*   **USB Ne Radi**: Provjeri `USB_DISCONNECT` pin (PB9) i pull-up na PA12 (DP). Blue Pill nekad ima pogrešan otpornik na USB brzinama.
