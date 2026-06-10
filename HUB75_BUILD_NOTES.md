# WLED + HUB75 — návod na buildy a flash (osobní poznámky)

Jak postavit a nahrát WLED s HUB75 maticí. Prostředí: **WSL2** (build) + **Windows** (flash přes USB).

---

## 0. Jednorázová příprava

PlatformIO není v systému. Nainstaluj ho bez sudo do uživatelského prostoru:

```bash
python3 -m pip install --user --upgrade platformio
```

Pak se PlatformIO spouští jako Python modul (není na PATH):

```bash
python3 -m platformio --version
```

> ⚠️ **WSL + sandbox:** první build stahuje toolchain a espressif32 platformu z GitHubu.
> Pokud build spadne na `No route to host` (sandbox blokuje síť), spusť ho s vypnutým
> sandboxem. Toolchain se uloží do `~/.platformio` a další buildy už síť nepotřebují
> (dokud se nepřidají nové knihovny).

---

## 1. Které prostředí (env) pro kterou desku

| Deska | env | Pozn. |
|-------|-----|-------|
| Klasický ESP32-WROOM (esp32dev) | `esp32dev_hub75` | hotové v `platformio.ini` |
| Klasický ESP32, SmartMatrix/forum zapojení | `esp32dev_hub75_forum_pinout` | hotové |
| **ESP32-S3-WROOM-1 N8R8** (8MB flash / 8MB octal PSRAM) | `esp32s3_n8r8_hub75` | **vlastní env v `platformio_override.ini`** |

Přepínač `-D WLED_ENABLE_HUB75MATRIX` je ve všech těchto envech zapnutý automaticky
(přes blok `[hub75]` v `platformio.ini`, řádek `build_flags`).

### Vlastní env pro S3 N8R8

Je v `platformio_override.ini` (přežije update WLED). Dědí ze stock envu
`esp32s3dev_8MB_opi` (octal PSRAM, `qio_opi`) a přidá HUB75 flagy:

```ini
[env:esp32s3_n8r8_hub75]
extends = env:esp32s3dev_8MB_opi
build_flags = ${common.build_flags} ${esp32s3.build_flags} ${hub75.build_flags} ${hub75.s3_build_flags} ${hub75.i2s_disable_flags}
              -D WLED_RELEASE_NAME=\"ESP32-S3_N8R8_HUB75\"
lib_deps = ${esp32s3.lib_deps}
           ${hub75.lib_deps}
```

Pro jinou variantu S3 změň base env (`extends`):
- N16R8 (16MB/8MB octal) → `env:esp32s3dev_16MB_opi`
- N8R8 (8MB/8MB octal) → `env:esp32s3dev_8MB_opi`  ← *aktuální*
- N8R2 (8MB/2MB quad)  → `env:esp32s3dev_8MB_qspi`

---

## 2. Build

```bash
# S3 N8R8 (aktuální deska)
python3 -m platformio run -e esp32s3_n8r8_hub75

# klasický ESP32-WROOM
python3 -m platformio run -e esp32dev_hub75
```

(Když síť blokuje sandbox, viz pozn. v sekci 0.)

Výstup:
- `.pio/build/<env>/firmware.bin` — app image (pro OTA update)
- `build_output/release/WLED_<verze>_<RELEASE_NAME>.bin` — totéž, pojmenované
- `.pio/build/<env>/{bootloader,partitions}.bin` — pro factory image

---

## 3. Factory bin (jeden soubor na offset 0x0)

Sloučí bootloader + partitions + boot_app0 + app do jednoho souboru.
**Offsety a `--chip` se liší podle čipu!**

`boot_app0.bin` najdeš tady:
```
~/.platformio/packages/framework-arduinoespressif32/tools/partitions/boot_app0.bin
```

### ESP32-S3 (bootloader na 0x0)

> ⚠️ **Použij `dio` / `40m`, ne `qio` / `80m`!** S `qio`/`80MHz` se deska zacyklila
> v boot loopu (ROM nezvládl číst flash — viz Troubleshooting). `dio`/`40m` je
> univerzálně funkční kombinace.

```bash
B=.pio/build/esp32s3_n8r8_hub75
BOOT_APP0=$(ls ~/.platformio/packages/framework-arduinoespressif32/tools/partitions/boot_app0.bin)
python3 -m platformio pkg exec -p tool-esptoolpy -- esptool.py --chip esp32s3 merge_bin \
  -o build_output/release/WLED_S3_HUB75_factory.bin \
  --flash_mode dio --flash_freq 40m --flash_size 8MB \
  0x0 $B/bootloader.bin 0x8000 $B/partitions.bin 0xe000 "$BOOT_APP0" 0x10000 $B/firmware.bin
```

### Klasický ESP32 (bootloader na 0x1000!)

```bash
B=.pio/build/esp32dev_hub75
BOOT_APP0=$(ls ~/.platformio/packages/framework-arduinoespressif32/tools/partitions/boot_app0.bin)
python3 -m platformio pkg exec -p tool-esptoolpy -- esptool.py --chip esp32 merge_bin \
  -o build_output/release/WLED_ESP32_HUB75_factory.bin \
  --flash_mode dio --flash_freq 40m --flash_size 4MB \
  0x1000 $B/bootloader.bin 0x8000 $B/partitions.bin 0xe000 "$BOOT_APP0" 0x10000 $B/firmware.bin
```

Soubor pak zkopíruj do Windows:
```bash
cp build_output/release/WLED_S3_HUB75_factory.bin /mnt/c/Users/chalu/Downloads/
```

---

## 4. Flash z Windows

> WSL2 nevidí USB sériové porty → flashuje se z Windows strany (port je tam nativně).
> Alternativa: probrosit USB do WSL přes `usbipd-win`, ale je to otrava (chybí driver
> v kernelu). Jednodušší je flashovat z Windows.

esptool na Windows je v PATH potíž — spouštěj ho přes modul. **Před flashem je dobré
udělat plný erase** (vyčistí staré zbytky / boot loop):

```
python -m esptool --chip esp32s3 --port COM5 erase_flash
python -m esptool --chip esp32s3 --port COM5 write-flash 0x0 WLED_S3_HUB75_factory.bin
```

> esptool **v5** přejmenoval `write_flash` → `write-flash` (se spojovníkem); `erase_flash`
> stále funguje. (`COM5` = port z Tovární zařízení → Porty (COM a LPT). Pro klasický
> ESP32 dej `--chip esp32`.) Před flashem ukonči případný běžící miniterm (`Ctrl+]`),
> jinak drží COM port.

**Nebo bez instalace, v prohlížeči:** https://espressif.github.io/esptool-js/
→ Connect → COM port → přidej factory bin, offset **`0x0`** → Program.

**Download mód, když port nenaváže:** podrž **BOOT**, cvakni **RESET**, pusť BOOT.
U S3 s nativním USB se po flashi může změnit COM port (objeví se nové USB zařízení) — normální.

**Nejjednodušší update, když už WLED běží:** OTA přes web UI →
Config → Security & Updates → Manual OTA Update → nahraj `firmware.bin` (app image, ne factory).

---

## 5. Piny HUB75

> ⚠️ Piny jsou **napevno ve firmwaru** — nejde je měnit v UI ani build flagem.
> Definice: `wled00/bus_manager.cpp` (hledej `mxconfig.gpio`).
> Panel se MUSÍ zapojit přesně podle tabulky pro daný build.

### ESP32-S3 s PSRAM, výchozí pinout (aktuální deska)

Větev `CONFIG_IDF_TARGET_ESP32S3 && BOARD_HAS_PSRAM`, bez pinout makra:

| Signál | GPIO | | Signál | GPIO |
|--------|------|---|--------|------|
| R1 | 1  | | A | 45 |
| G1 | 2  | | B | 48 |
| B1 | 42 | | C | 47 |
| R2 | 41 | | D | 21 |
| G2 | 40 | | E | 38 |
| B2 | 39 | | LAT/STB | 8 |
| CLK | 18 | | OE | 3 |

Plus **GND** panelu na GND desky. **E (GPIO38)** se zapojuje jen u 1/32-scan panelů
(64px na výšku); u 1/16-scan (32px) se nechá nezapojené.

#### Náš konkrétní panel (1/8-scan)

Tenhle panel má 16pinový konektor **HUB75 (ne HUB75E)** — adresní linky jen **A, B, C**;
pozice **D je označená `NC`** a **E na konektoru není**. Je to **1/8-scan** panel.
Takže:

- **D = GPIO21** → nezapojovat (panel D = NC)
- **E = GPIO38** → nezapojovat (panel E neexistuje)
- ostatní piny podle tabulky výše

Firmwaru nevadí, že D/E vedou nikam — pořád je obsluhuje, jen nejsou připojené.

**Sestava: 3× modul 64×32 v řadě zleva doprava = celkem 192×32.** Každý modul je
1/8-scan, výška 32 → ve WLED **HUB75 (Quarter Scan)** (32/4 = 8). Pravidlo: 1/8-scan
panel vysoký 32 px = Quarter Scan; vysoký 16 px = Half Scan. (Pozor: Quarter Scan
podporuje jen výšky 16/32/64 — jiná výška = „Unsupported height" a driver se zastaví.)

#### Nastavení v UI (Config → LED Preferences → LED outputs)

| Pole | Hodnota |
|------|---------|
| LED output typ | **HUB75 (Quarter Scan)** |
| Panel (width × height) | **64 × 32** (rozměr jednoho modulu!) |
| No. of Panels | **3** |
| rows × cols | **1 × 3** |
| Reversed | nezaškrtnuto |

Pak **Config → 2D Configuration → Strip or panel → 2D Matrix** (rozměry 192×32), Save.
Bez toho WLED renderuje jen 1D.

#### Zapojení řetězu (3 moduly)

ESP32 → jen **vstup prvního modulu** (13 vodičů + GND). Dál se moduly řetězí plochým
kabelem: OUT modulu 1 → IN modulu 2 → OUT modulu 2 → IN modulu 3.

> Když obraz vyjde rozsekaný/zdvojený, zkus přepnout na **Half Scan** (empirický test).
> Špatné pořadí/zrcadlení modulů → **Reversed** nebo doladit v 2D mapě. Prohozené barvy
> (R↔B) → color order RGB↔BGR. Half Scan = běžné panely (scan = výška/2), Quarter Scan
> = FOUR_SCAN panely s remappingem (scan = výška/4).

**Pozn. k volbě pinů (proč to nezlobí):**
- Vyhýbá se flash pinům (26–32), **octal-PSRAM pinům (33–37)** i USB pinům (19, 20) — proto OK pro N8R8.
- GPIO 45 (A) a GPIO 3 (OE) jsou strapping piny, ale jako výstupy to v praxi nevadí (panel je přijímač). OE může při bootu chvíli bliknout šum.
- GPIO 48 (B) je vývod onboard RGB LED na DevKitC-1 → vestavěná ledka bude blikat. Neškodné.

### Klasický ESP32, výchozí pinout

Větev „Default pins":

| R1=25 | G1=26 | B1=27 | R2=14 | G2=12 | B2=13 |
|-------|-------|-------|-------|-------|-------|
| A=23 | B=19 | C=5 | D=17 | E=18 | |
| LAT=4 | OE=15 | CLK=16 | | | |

### Další hotové pinouty (přes makro v build_flags)

| Makro | Deska |
|-------|-------|
| `ESP32_FORUM_PINOUT` | klasický ESP32, SmartMatrix zapojení |
| `HD_WF2_PINOUT` | Huidu HD-WF2 (S3, bez PSRAM) |
| `MOONHUB_S3_PINOUT` | MOONHUB adaptér (lilygo T7-S3) |
| `ARDUINO_ADAFRUIT_MATRIXPORTAL_ESP32S3` | Adafruit MatrixPortal S3 |

---

## 6. Nastavení ve WLED UI po naběhnutí

1. **Config → LED Preferences** → přidej LED output typu **HUB75**, nastav šířku, výšku
   a počet zřetězených panelů (chain length).
2. **Config → 2D Configuration** → nastav 2D mapování (matrix).
3. Proti ghostingu (panel svítí i kde nemá): build flag `-D WLED_HUB75_MAX_BRIGHTNESS=239`
   (nebo níž) — přidat do `build_flags` envu a přebuildit.

---

## 7. Napájení (důležité)

Velký panel táhne hodně proudu — klidně **4–8 A na 5 V** (a víc u velkých/jasných).
**Napájej panel z externího 5V zdroje, ne z USB.** GND zdroje, panelu a ESP32 musí
být spojené. ESP32-S3 s 8MB PSRAM zvládá plnou 8bit barevnou hloubku i pro velké panely
(omezení bitdepth v kódu se týká jen klasického ESP32/S2, ne S3).

---

## 8. Troubleshooting

### Boot loop po flashi (dokola se opakuje ROM bootloader)

Příznak v sériovém logu (115200 baud) — opakuje se pořád dokola, nikdy nenaběhne WLED:

```
ESP-ROM:esp32s3-20210327
rst:0x7 (TG0WDT_SYS_RST) / rst:0x10 (RTCWDT_RTC_RST)
load:0x3fce3808,len:0x128
ets_loader.c 78
```

Padá ještě **před** druhým stupněm bootloaderu = ROM nedokáže číst flash tak, jak mu
říká hlavička image. Řešení (od nejpravděpodobnějšího):

1. **Špatná flash mode/freq ve factory binu** — vyrob ho znovu s `--flash_mode dio
   --flash_freq 40m` (ne `qio`/`80m`), pak `erase_flash` + `write-flash`. ← tohle nám pomohlo.
2. Panel připojený na **strapping piny** (GPIO45=A, GPIO3=OE) může táhnout boot — odpoj
   panel a zkus znovu. (U nás panel připojený nebyl, takže to byla příčina č. 1.)
3. Špatný COM port pro log — u S3 s nativním USB jdou ROM logy přes USB-Serial-JTAG;
   po bootu se může číslo COM portu změnit (zkontroluj Správce zařízení).

### Quarter Scan: rozměry se po uložení samy mění (32 → 16 → 8 …)

**Příznak:** zadáš Panel `64×32`, dáš Save, a ono se to přepne na `128×8` (a panel
přestane fungovat / „Unsupported height").

**Příčina (bug ve stock WLED):** pro Quarter Scan konstruktor přepočítá rozměry na
fyzické (`mx_width = šířka×2`, `mx_height = výška÷2`), ale `BusHub75Matrix::getPins()`
vrací do configu právě tyhle **přepočtené** hodnoty. Při každém uložení/bootu se
přepočítají znovu → výška se půlí (32 → 16 → 8 → 4). Navíc se z `panelHeight` vybírá
scan režim (case 16/32/64), takže po pár cyklech spadne do „Unsupported height".

**Fix (aplikovaný v našem buildu):** `getPins()` v [bus_manager.cpp:1145](wled00/bus_manager.cpp#L1145)
upraven tak, aby pro Quarter Scan vracel **logické** rozměry (obrácení přepočtu:
`mx_width/2`, `mx_height*2`). Pak je `64×32` stabilní napříč rebooty. Po aktualizaci
firmwaru zadej `64×32` znovu (přepíše starou špatnou hodnotu v configu).

> Pozn.: tohle je úprava zdrojáku WLED (ne jen `platformio_override.ini`) — při update
> WLED z upstreamu ji bude potřeba znovu aplikovat (nebo poslat jako PR).

### WLED-AP se neobjeví
- Dej **fyzický RESET** (tlačítko) — „Hard resetting via RTS pin" u nativního USB často nezabere.
- SSID `WLED-AP`, heslo `wled1234`; chvíli to po prvním bootu trvá. Zkus jiné zařízení (mobil).
- Když pořád nic → sériový log (sekce 4), hledej boot loop nebo „Guru Meditation".

---

## 9. Git / verzování

`build_output/` je v `.gitignore`, ale upravili jsme to tak, aby **release biny šly
commitnout** (ignorují se jen velké `.map` a `firmware/`):

```gitignore
/build_output/map/
/build_output/firmware/
```

> `platformio_override.ini` zůstává v `.gitignore` (WLED ho ignoruje záměrně jako
> user-local). Obsah envu `esp32s3_n8r8_hub75` je ale zdokumentovaný v sekci 1 tohoto
> souboru, takže je kdykoli k obnovení.
