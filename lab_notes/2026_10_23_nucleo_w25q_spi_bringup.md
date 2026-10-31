# NUCLEO külső SPI flash (W25Q) bringup - 1. lépés HARDVERESEN KÉSZ, 2026-10-23

A NUCLEO OTA külső-flash áttervezés 1. lépése (ld.
`Documentation/brainstorms/2026_10_02_nucleo_ota_external_flash/brainstorm.md`):
a megérkezett SPI NOR flash bekötése + driver bringup. **Valódi hardveren
igazolva**: a JEDEC ID + státuszregiszter helyesen beolvasva.

## Chip és bekötés
- Chip: **Winbond W25Q64** (JEDEC ID `EF 40 17` -> mfr EF, type 40, cap 0x17 =
  64 Mbit / **8 MiB**). A rendelt chip tényleges típusa ezzel megerősítve.
- NUCLEO-WL55JC1 Arduino-fejléc (CN5/CN6), a pinout a board fotóiról + 3
  független BSP-forrásból (Zephyr/NuttX/Meshtastic) megerősítve, nem tippelve:
  - PA5 = SPI1_SCK (D13), PA6 = SPI1_MISO (D12), PA7 = SPI1_MOSI (D11), AF5
  - PA4 = szoftveres CS (D10), 3V3 + GND a CN6-ról.

## Firmware
- Új, **csupasz-regiszteres SPI1 driver** (`Core/Src/w25q_bringup.c` +
  `Core/Inc/w25q_bringup.h`): a vendorolt fa nem engedélyezi/építi a HAL SPI
  modult, és az ST-források letöltése tiltott, ezért közvetlen SPI1/GPIOA/RCC
  regiszter-hozzáférés (a CMSIS device header a `main.h`-n át jön). Lassú
  bringup-órajel (fPCLK2/256), mode 0, 8-bit.
- `W25Q_Bringup()`: SPI1-init (egyszer), JEDEC ID (0x9F) + SR1 (0x05) olvasás,
  `APP_LOG` kiírás a VCP-re. A `base_node` main.c-be hookolva: egyszeri hívás a
  `MX_LoRaWAN_Init()` után + **periodikus** (~3 s) a main-loopban (lásd lentebb,
  miért kell a periodikus).
- **Formázási bug menet közben:** az ST `stm32_tiny_vsnprintf` nem tudja a
  `%lu`-t (literálisan kiírta) - `%u`-ra javítva.

## Build/flash buktatók (megoldva)
1. **A buildelhető fa nem a repóban van:** a repó csak a user-fájlokat (Core/,
   LoRaWAN/App…) tartja; a teljes fa (Drivers/CMSIS/HAL + LBM middleware) a
   `~/source/repos/STM32CubeWL` klónban van (2026-10-02 lab note). A build a
   klón `Projects/NUCLEO-WL55JC/.../LoRaWAN_End_Node_LBM` mappájában megy; a
   repó Core-fájljait oda másolva, `gen_makefile.py` + `make`
   (`arm-none-eabi-gcc`), flash st-link-kel.
2. **A klón `gen_makefile.py`-ja régebbi volt**, hiányzott belőle az új custom
   `Core/Src/*.c` auto-glob - ezért a `w25q_bringup.c` nem linkelődött
   (`undefined reference`). A repóbeli (újabb) `gen_makefile.py`-t a klónba
   másolva megoldódott.
3. **OpenOCD nem tud csatlakozni** a futó firmware-hez: a base_node STOP módba
   megy, ami lekapcsolja a debug portot, az ST-Link hla transport pedig nem tud
   SRST-t assertálni (`BUG: can't assert SRST`). Megoldás: **`st-flash
   --connect-under-reset --reset write <bin> 0x08000000`** (a ST-Link natív
   under-reset-je kiveszi STOP-ból).
4. **ST-Link beragadt** egy félbemaradt flash után (`GET_CURRENT_MODE /
   ENTER_SWD LIBUSB_ERROR_TIMEOUT`). Fizikai újradugás helyett **USBDEVFS_RESET
   ioctl**-lel újraenumeráltam (`/dev/bus/usb/.../`), utána a flash ment.
5. **Az egyszeri boot-kiírás nem elkapható:** az ST-Link V3 ugyanazon az USB-n
   adja az SWD-t és a VCP-t (ttyACM0), így flashelés és soros-capture nem fed
   át. Ezért a JEDEC-olvasást **periodikusan** is kiíratom (~3 s), és a
   kiolvasás: `st-flash reset` **párhuzamosan futó** soros-capture mellett.

## Eredmény (valódi hardver)
```
[W25Q] JEDEC ID: EF 40 17   SR1=00
[W25Q] Winbond, type=40, ~8192 KiB
```
SR1=00 -> a chip nem busy, nincs write-protect; a teljes SPI oda-vissza út
(parancs ki + adat be) működik.

## 2. lépés - erase/program/read KÉSZ, hardveresen igazolva (ugyanaznap)
A driver kibővítve: write-enable (0x06), sector erase 4 KiB (0x20), page program
(0x02), read (0x03), BUSY-poll (SR1 bit0, korlátos guarddal). Egyszeri self-teszt
egy scratch-szektoron (`0x001000`): erase -> blank-check (mind 0xFF) -> 16 bájtos
minta (A0..AF) programozás -> read-back verify. Eredmény a VCP-n:
```
[W25Q] erase/program/read self-test @0x001000: PASS (rb[0]=A0, expect A0)
```
A teljes flash írás/olvasás út tehát működik a valódi W25Q64-en.

## 3. lépés - staging átirányítva a W25Q-ra, FORDUL (2026-10-24)
A bringup driver tiszta, újrafelhasználható modullá refaktorálva (`w25q.c`/`.h`,
`otaa_basic` Core-jában), API: `W25Q_Init/ReadJedec/Read/SectorErase/PageProgram`.
Az `ota_update.c` staging-rétege átirányítva:
- **write:** `ota_flush_pending` a belső `FLASH_IF_Write` helyett `W25Q_PageProgram`-ot
  hív (8 bájtos blokkok sosem lépnek át 256-bájtos page-határt, így egy page-program
  mindig érvényes).
- **erase:** `ota_handle_manifest` a `FLASH_IF_Erase` helyett annyi 4 KiB-os W25Q
  sectort töröl (`W25Q_SectorErase`), amennyi az image-hez kell, + `W25Q_Init()`.
- **címek** (`ota_update.h`): `OTA_STAGING_FLASH_ADDR`=0 (W25Q offset),
  `OTA_STAGING_FLASH_SIZE`=256 KiB, `OTA_STATE_FLASH_ADDR`=0x0803D800 (belső, csak
  secure módban). A `#error` guard a külső-flash szemantikához igazítva.
- **Kulcsfelismerés:** a SHA-256 inkrementálisan számolódik a beérkező chunkokból
  (`sha256_update`), **nincs staging-visszaolvasás** a verify-hoz -> a nem
  memory-mapped W25Q nem gond, csak a write+erase kellett átirányítani.

Build: az `otaa_basic` fát a STM32CubeWL klónba overlay-elve (`gen_makefile.py`
auto-glob felveszi a `w25q.c`-t és `ota_update.c`-t) **tisztán fordul** (ELF text
123488). **Ez fordítás, nem hardveres igazolás.**

## 4. lépés indítása - join blokkoló megtalálva és megoldva (2026-10-24)
Az `otaa_basic` felflashelve, de **JOINFAIL**. Diagnózis:
- ChirpStack **gateway** stream (`StreamGatewayFrames`): 12 élő JoinRequest -> **0
  downlink** -> ChirpStack át sem adja a JoinAccept-et a gateway-nek. Tehát **nem RF,
  nem a node vétele** a gond; az RTL-SDR itt nem segít.
- **ChirpStack szerver-log (SSH):** `join: Handle join-request error error=Invalid MIC`
  minden join requestre -> a JoinRequest MIC-je nem validál a szerver AppKey-jével.
- **gRPC GetKeys:** ChirpStack nwk_key/app_key = `86b1deca…98ba`, join_eui =
  `2670…1183`; a firmware `se-identity.h`-ja viszont az ST stock kulcsot
  (`2b7e1516…`) + JoinEUI `0101…` használja. **Ez az eltérés** okozta az Invalid MIC-et.

**Döntés:** ChirpStack marad érintetlen (alap), a **firmware** kulcsát
igazítom. A `se-identity.h` `LORAWAN_JOIN_EUI`/`GEN_APP_KEY`/`APP_KEY` a ChirpStack-beli
értékekre állítva - **csak a build-másolatban (STM32CubeWL klón), a repóba NEM**
(a valós értékek a gitignore-olt `Information.md`-ben). Újrabuild + flash
után: **nincs több Invalid MIC**, ChirpStack `data_up`-okat logol, `GetActivation`
DevAddr `00a8650c`, f_cnt_up 13 -> **a node joinol és küld**. A step-4 OTA-push
blokkolója feloldva; jöhet a tényleges image-push.

## 4. lépés - OTA-push: a W25Q staging-út OTA-ból IGAZOLVA, teljes transzfer flaky (2026-10-24)
Setup: `Tools/OTA_Update` - `config.json` device.dev_eui = `0080e115061bebe9`,
`--fport 10` (= firmware `OTA_PORT`). A tool aláírt firmware-t vár; Ed25519 kulcs
`keygen.py`-val (gitignore-olt `keys/ota_signing_key.pem`), image `sign_firmware.py`-val
(manifest = verzió + SHA256 + aláírás). `otaa_basic` nem ellenőrzi az aláírást
(OTA_SECURE_MODE=0), de a tool akkor is kéri.

**Eredmények (valódi hardver, LoRaWAN-on):**
- **256 bájtos push:** a node UART-ján `OTA: manifest accepted, staging region
  erased, ready for data frames` -> **a 3. lépésben megírt W25Q-erase staging-út
  valódi OTA-downlinkből, hardveren működik.** A manifest + chunk 1 kézbesítve, de
  **chunk 2-nél a downlink-queue nem ürült** (120s, 3 retry -> abort, 2/6 chunk).
- **40 bájtos (1-chunk) push:** egyszer **tool-szinten SUCCESS** (21s, queue kiürült);
  egy másik próbán viszont már a **manifest** is beragadt (queue depth=1, retry).

**Diagnózis:** a downlink-kézbesítés **intermittens** - néha pár mp alatt megy,
néha percekig beragad. A node SF7-en van (nem SF12), tehát nem a lassú uplink a gond;
inkább LoRaWAN downlink-oldali (duty-cycle / RX-timing / gateway downlink flakiness,
a `Tools/OTA_Update/README.md` "Unconfirmed downlinks only" + queue-depth
false-positive korlátja). **Nem a firmware-kód hibája** - a staging-logika igazoltan
indul. Ráadásul a **ST-Link VCP időnként néma** (reset nélkül nem ad trace-t, reset
viszont megtörné az OTA-állapotot), ami a node-oldali monitorozást nehezíti.

**Mellékes bug javítva:** az `ota_update.c` `APP_LOG`-jai `%lu`-t használtak, amit az
ST `stm32_tiny_vsnprintf` nem renderel (literálisan `%lu`-t írt a manifest-logban) ->
`%u`-ra javítva (32-bites ARM-on a `(unsigned long)` cast és `%u` ugyanaz a méret).

**Állapot:** a W25Q staging **redirect (3. lépés) OTA-ból, hardveren igazolt** a
manifest+erase úton; a teljes, megbízható multi-chunk transzfer a LoRaWAN
downlink-reliability tuningtól függ (confirmed downlink / RX-timing / kisebb image /
hosszabb poll-timeout) - ez lassú, iteratív, külön menet. **Aktiváció** (mini-bootloader)
továbbra is a 11-15 tétel.

## 4. lépés - confirmed downlink: manifest megbízható, chunk-stall marad (2026-10-25)
A downlink-root-cause megtalálva: a tool **unconfirmed** downlinkeket küldött, ezért
a manifest némán elveszett (a node `data frame ... no active manifest, ignored`-ot
logolt). Átállítva **confirmed downlinkekre** (`grpc_client.enqueue(confirmed=True)`,
commit `67ada88`). Hatás, hardveren igazolva (NUCLEO fizikai replug után, tiszta
állapotból): a **manifest most ~14s alatt megbízhatóan kézbesül**, `version=3
total_bytes=40` (a `%lu`->`%u` fix is látszik), a **W25Q staging-erase lefut**.

**De a data chunk konzisztensen beragad:** manifest OK (14s), chunk SOHA (180s × 3
retry). A node közben végig ad (~5s-enként TX MNG REQ) és kap is downlinket (második
DOWNDATA esemény), tehát **nem node/VCP/erase-blokk gond** - ChirpStack a chunkot a
queue-ban tartja, de nem ütemezi ki a RX-ablakba. INFO-log szinten nem látszik az ok.

**DEBUG-diagnosztika előkészítve (és visszaállítva):** a szerveren
`configuration/chirpstack/chirpstack.toml` `[logging] level` átállítható `debug`-ra
(`docker restart lorawan-chirpstack-1`), ekkor a downlink-scheduler döntése látszik.
Ezt beállítottam, DE a node közben ismét rossz állapotba került (UART néma, nem ad),
így a chunk-stallt most nem tudtam előállítani a log-rögzítéshez - a log-szintet
**visszaállítottam `info`-ra** (szerver = baseline). A diagnosztika kész, a következő
menetben (tiszta, power-cycle-ölt node) gyors.

**Halmozódó akadály:** a NUCLEO + ST-Link VCP ismételten rossz állapotba kerül (node
elnémul, VCP néma) a sok szoftveres reset/push után; fizikai ki-be dugás átmenetileg
helyreállítja. A megbízható OTA-iterációhoz emiatt tiszta kiindulás kell minden
futtatáshoz, és kerülendő a sok `st-flash reset`.

## 4. lépés - GYÖKÉROK ÁTKERETEZVE: a NODE LEÁLL a manifest után (2026-10-25)
A DEBUG-diagnosztika közben (chirpstack.toml -> debug, friss replug-olt node) kiderült,
hogy a chunk nem azért nem megy, mert ChirpStack nem ütemezi - hanem mert **a node
maga leáll a manifest feldolgozása után.** Három FÜGGETLEN mérés egybehangzik:
- node UART: néma (nincs trace),
- ChirpStack DEBUG-log: a device `0080e115…` **15 percig egyáltalán nem jelenik meg**
  (se uplink, se semmi), miközben a gateway-stats folyik,
- **RTL-SDR on-air (a bedugott rádióval): 0 burst 30 másodperc alatt** (csúcs csak
  +2.8 dB a zaj felett) -> a node **fizikailag nem ad a levegőbe.**

Tehát a chunk azért "ragad be", mert **mire a chunk jönne, a node már halott** (nincs
RX-ablak az ACK-hoz). Ez megmagyarázza az összes korábbi flakyságot (manifest néha
átment + erase lefutott, aztán elnémulás; a "második DOWNDATA no OTA log").

**Fő gyanúsított (firmware):** a manifest-út **blokkoló W25Q-műveletei az LBM
real-time rádió-callbackjében**. A `w25q_wait_busy` bounded loopja (`g < 4_000_000`)
a lassú /256-os SPI-n akár **~344 s blokkolás** is lehet, ha a BUSY nem tisztul -> a
node teljesen megáll. A 4KB sector-erase ~400ms blokkolása is megtörheti az LBM
RX/TX időzítését a callbackben.

A szerver (ChirpStack) ekkor ismét `info` log-szinten, baseline. A tool-oldali
confirmed-downlink fix (`67ada88`) megmarad - az a manifest-megbízhatóságot valóban
javította, a hang ettől független firmware-ügy.

## 5. lépés - A node-leállás VALÓDI gyökéroka + javítás, hardveren igazolva (2026-10-25 .. 10-28)

### Két fix-kísérlet, ami NEM volt elég
1. **Deferálás a main-loopba** (a downlink-callback csak pufferel, az erase/program a
   `main.c` while-ciklusából hívott `OTA_Update_Process()`-ben fut): a node a
   manifest után ugyanúgy elnémult (RTL-SDR: nincs on-air burst, push FAILED).
2. **Nem-blokkoló állapotgép** (`W25Q_SectorErase_Issue` / `W25Q_PageProgram_Issue` /
   `W25Q_IsBusy`, main-loop iterációnként egy lépés, BUSY-t pollozva, sosem pörögve):
   a node a manifest + erase-parancs UTÁN még egyszer adott (`TXDONE` a manifest
   log után - ez új volt), majd elnémult. A push-oldalon a ChirpStack-queue a
   manifestnél **3 retry alatt sem ürült** (depth=1) -> független bizonyíték, hogy
   a node tényleg leállt, nem csak a VCP.

A 2. kísérletből derült ki a lényeg: a manifest handler már egyáltalán nem nyúlt a
W25Q-hoz, a leállás az **első alvás utáni SPI-hozzáférésnél** jött.

### Gyökérok: SPI1 nem marad meg STOP2-ben
A `Core/Src/stm32_lpm_if.c` `PWR_ExitStopMode()` kommentje szerint STOP2 után a
**SPIx perifériák NEM őrződnek meg** (csak a USART/DMA-t állítja vissza a
`vcom_Resume()`). A `W25Q_Init()` viszont egy `static inited` flaggel csak egyszer
konfigurált. Lefolyás:
1. manifest -> első `W25Q_Init()` + erase-parancs: rendben;
2. az LBM sequencer STOP2-be alszik -> SPI1 regiszterei resetelődnek (SPE=0);
3. ébredés -> `spi_xfer()` a `RXNE`-re vár, ami kikapcsolt SPI-n **soha nem jön** ->
   örök pörgés a main loopban -> a stack nem fut tovább, a node elnémul.

Ez MINDEN korábbi tünetet megmagyaráz: a base_node bringup azért ment, mert a
self-test bootkor, STOP2 előtt futott; a stub-teszt azért, mert ott nem volt SPI;
a "blokkolás a callbackben" hipotézis tévút volt (a blokkolás nem segített, de nem
is ez ölt). A `st-info --probe` "Failed to enter SWD mode" is stimmel: `DEBUGGER_ENABLED=0`
mellett az ST init letiltja az SWD-lábakat (csak `--connect-under-reset` megy).

### Javítás
- `w25q.c` `W25Q_Init()`: a `static inited` helyett `if (SPI1->CR1 & SPI_CR1_SPE) return;`
  -> ha az SPI1 ki van kapcsolva (STOP2 után), újrakonfigurál. GPIO/RCC beállítás
  STOP2-ben megmarad, az újraírás ártalmatlan.
- `ota_update.c` `OTA_Update_Process()`: ha van függő munka, minden lépés előtt
  `W25Q_Init()`; ha nincs, korán visszatér (nem nyúl az SPI1-hez).
- A nem-blokkoló állapotgép megmarad: a javítás nélkül is a helyes design (egy
  egész-image erase blokkolva több mp-ig tartaná fel a stacket).

### Hardveres igazolás (2026-10-28, NUCLEO + W25Q64 + ChirpStack, confirmed downlink)
| Teszt | Chunk | Node UART | Push |
|---|---|---|---|
| 250 B, v5 (1 lap) | 5×50 B | `seq 0..4 ok` -> `whole-image SHA-256 verified OK` | SUCCESS, 5/5, 0 retry, 68 s |
| 600 B, v6 (2 laphatár, lap-split ág) | 12×50 B | `seq 0..11 ok` -> `whole-image SHA-256 verified OK` | SUCCESS, 12/12, 0 retry, 137 s |

A node mindkét transzfer után tovább ad (`TX DONE` / `Next TX` a verify után).

**Korlát:** a node-oldali SHA-256 a *beérkezett* bájtokon számolódik, nem a
W25Q-ból visszaolvasott tartalmon - ez a fogadási láncot igazolja, a flash-tartalmat
nem. Többszektoros (>4 KiB) image-et még nem teszteltem (~13.6 s/chunk -> egy 5 KiB
image ~23 perc).

**Környezeti megfigyelések:** flash után a VCP gyakran néma; az ST-Link
USBDEVFS_RESET + `st-flash --connect-under-reset reset` helyreállítja, de a NUCLEO
VCP ekkor más `ttyACM` számot kaphat (most `ttyACM0`, korábban `ttyACM1`).
