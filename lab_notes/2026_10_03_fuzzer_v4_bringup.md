# Gateway PHY-fuzzer (Heltec V4) - hardveres bring-up (2026-10-03)

## Előfeltételek (a 09-28-i README feltételezéseivel szemben)
Minden megvolt már telepítve: Heltec-esp32 core 3.3.8, RadioLib 7.6.0,
„ESP8266 and ESP32 OLED driver for SSD1306" 4.6.2. A V4-hez nincs külön
FQBN, de a sketch explicit pineket használ, így a **V3 FQBN**
(`Heltec-esp32:esp32:heltec_wifi_lora_32_V3`) jó - a V4 SX1262/OLED pinjeit
a sketch adja.

## Fordítási hiba - javítva
A kód **sosem volt lefordítva** (09-28: „written, untested"). Első build
elhasalt: a `logResult()` a definíciója előtt hívódik (`doFuzzCycle`-ben),
és az Arduino auto-prototípus rosszul vezette le a típust (`const int&`).
**Javítás:** explicit forward-deklaráció a `ResultEntry` struct után.
Ezután tisztán fordul (1 029 547 byte, 30% flash).

## Flash + futás (igazolva)
- Flash a `/dev/ttyACM0`-n (ESP32-S3 natív USB-JTAG), hash verify OK.
- Boot ROM üzenetek rendben (`SPI_FAST_FLASH_BOOT`, `entry …`) - **nincs
  boot-loop** (a V3-as GigaDevice-bootloader gond itt nem jött elő).
- **WiFi AP `LoRaWAN-Fuzzer-5688` fent** (dashboard él).
- **OLED: „868.1MHz SF7 TX:114 Sleeping"** -> a `radio.begin()` sikerült
  (különben „Radio init FAILED"-be ragadna a `while(true)` ág), és a rádió
  **114 TX-ciklust** adott. A fuzzer működik, ahogy tervezve: 868.1 MHz,
  SF7, BW125, sync 0x34 (LoRaWAN public), 10s-enként, BASELINE variánssal.

## Tooling-korlátok, amiket megtanultam
- **Soros kimenet nem jön a `ttyACM0`-n:** a Heltec board-def a `Serial`-t
  az UART0-ra teszi (nem a natív USB CDC-re), így a `ttyACM0`-n csak a ROM
  boot-üzenetei látszanak, a sketch `Serial.print`-je nem. (A státuszt az
  OLED + a WiFi-dashboard adja.)
- **A ChirpStack `StreamGatewayFrames` (Internal API) nem jeleníti meg a
  fuzzer kereteit.** A base_node érvényes LoRaWAN-keretei látszanak, a
  fuzzer BASELINE kerete (érvénytelen LoRaWAN-MHDR: MType/major a „FUZZ…"
  payloadból) viszont nem jött át a 75s-es capture-ben. Vagyis **a
  malformed/nem-LoRaWAN keretek vizsgálatára a ChirpStack API nem alkalmas**
  - a gateway (RAK7268) saját packet-forwarder logja kell hozzá (ez a
  README „Testing TODO" 3./5. lépése).
