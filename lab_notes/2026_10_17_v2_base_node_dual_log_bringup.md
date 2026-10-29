# V2 base_node bringup a jelenlegi ChirpStackre (kettős napló node-oldal), 2026-10-17

**Cél:** a kettős napló módszertan (node-oldali soros log + ChirpStack-oldali
kép ugyanarról az eseményről, ld. `ROADMAP.md` 1. sáv) node-oldalának
beindítása egy valódi Heltec V2-vel. Menet közben két, egymástól független
akadály derült ki; mindkettő megoldva, a lánc végül hardveresen igazolva.

## Node és firmware
- Heltec **WiFi LoRa 32 V2**, `/dev/ttyUSB0`, firmware: `base_node`
  (`Firmwares/Nodes/Heltec_WiFi_LoRa_32_V2/base_node/base_node.ino`, MCCI LMIC,
  LoRaWAN 1.0.x, EU868, TX_INTERVAL 15 s).
- Kulcsok flash-időben injektálva (`load.py`) egy `/tmp` másolatba, **nem a
  tracked forrásba**. FQBN `Heltec-esp32:esp32:heltec_wifi_lora_32_V2`.

## 1. akadály - a `base_node` nem fordult az aktuális toolchainnel
A sima `arduino-cli compile` két hibával elhasalt, **egyik sem a saját kódom**:
1. `SpiInOut` implicit deklaráció a Heltec "ESP32 Dev-Boards" library saját
   `driver/sx1276.c`-jében - a GCC 14 (esp-14.2.0) alapból hibának veszi
   (`-Wimplicit-function-declaration`), régebbi Arduino IDE toolchainnel csak
   warning volt.
2. `multiple definition of 'display'` - a Heltec library `LoRaWan_APP.cpp`-je is
   definiál egy globális `display` objektumot, ami ütközik a sketch sajátjával
   (`-fno-common` a modern linkeren).

Gyökér: a `base_node` a Heltec **bundle-olt** `HT_SSD1306Wire.h`-ját használta,
ami behúzta az egész Heltec LoRaWAN stacket. A `firmware_writer/builder.py` is
sima `arduino-cli compile`-t hív, tehát az sem segített volna - ezek a sketchek
legutóbb régebbi library/core verzióval fordultak.

**Megoldás (a repó saját, újabb konvenciója szerint, ahogy a V4 `base_node` is):**
a bundle-olt Heltec OLED helyett a különálló **ThingPulse `SSD1306Wire`** (telepítve:
"ESP8266 and ESP32 OLED driver for SSD1306 displays" 4.6.2). Változás a
`base_node.ino`-ban:
- `#include "HT_SSD1306Wire.h"` -> `#include <SSD1306Wire.h>`
- konstruktor `SSD1306Wire display(0x3c, 500000, 4, 15, GEOMETRY_128_64, 16)`
  -> `SSD1306Wire display(0x3c, 4, 15)` (a ThingPulse konstruktor nem kezel RST lábat)
- a V2 OLED RST-je (GPIO16) kézzel pulse-olva a `display.init()` előtt.

Ezután **flag nélkül tisztán fordult** (a Heltec Dev-Boards library be sem töltődik),
felflashelve + hash-verified, boot + soros kimenet + OLED init rendben.

## 2. akadály - a node nem joinolt (`no JoinAccept`)
Flash után a node kiküldte a Join Requestet, de több kísérletre sem jött
JoinAccept. **Nem RF/firmware/antenna probléma** (az antenna végig rajta volt).

Szerver-oldali diagnózis (gRPC, `Tools/OTA_Update` TLS-csatorna + tenant-token):
- A DevEUI `05ec…8fe5`-re `Device.Get` **UNAUTHENTICATED**-et adott - ez a
  már ismert ChirpStack tenant-kulcs quirk (NOT_FOUND helyett, ld.
  `2026_09_19_api_token_minimal_scope.md`), azaz az eszköz **nincs** ebben a
  tenantban/appban.
- A token maga érvényes: a tenantban 1 app van (`Application_01`,
  `5f34788a-9c7e-4d28-8b44-e6ccb0628a99`), benne **mindössze 3 eszköz**:
  `0080e115061bebe9` (NUCLEO base_node), `09deb432b597ef3a` (V4-attack-node),
  `feedface00000001` (weak-key-victim-POC).

**Fontos tanulság:** az `Information.md` V2/V3 node-jai (és a benne szereplő
`1c953322…` app-id) **egy korábbi/másik ChirpStack példányhoz tartoznak** (valószínűleg
a régi Pi-s legacy szerver) - a jelenlegi szerveren (192.168.0.166) **nincsenek
provisionálva**. Ezért nem joinolt a V2: nem volt kinek.

**Megoldás:** a V2 Node 1 provisionálva az `Application_01`-ben a `OTAA` device
profillal (`c3ee3bd1-c443-4544-a382-b706e4a5ecc7`, EU868, LoRaWAN 1.0.0), az
`Information.md`-beli értékekkel (DevEUI `05ec…8fe5`, JoinEUI
`d6e3…5502`). 1.0.x miatt az AppKey a `nwk_key` mezőbe írva.

## Igazolás (hardveres, mindkét oldal)
Node-oldali soros log provisioning után:
```
EV_JOINED - Successfully joined!
DevAddr: 14315C2
Packet queued: 1 -> EV_TXCOMPLETE
Packet queued: 2 -> EV_TXCOMPLETE   (+ további ciklusok 15 s-enként)
```
ChirpStack-oldal (`Device.GetActivation`) ugyanarra az eszközre:
```
DevAddr: 014315c2   (bit-pontosan == node 014315C2)
f_cnt_up: 6
last_seen: 2026-10-17 12:22
```
A DevAddr bit-pontos egyezése node és szerver között = a legerősebb korreláció,
ugyanaz a logika, mint az `Application_Key_Request` (2026-09-19) tesztben.

## Megjegyzések / nyitott
- **Soros capture resetel:** a port megnyitása (pyserial) minden alkalommal
  rebootolja a node-ot a DTR/RTS auto-reset miatt. A sima `cat` ezen felül
  **RTS-t asszertálva reset-ben tartja** a chipet (néma port!) - ezért pyserial
  kell `rts=False, dtr=False`-szal. A tartós kettős-napló-rögzítéshez a
  reset-on-open-t kezelni kell (hupcl off / tartós kapcsolat), hogy ne induljon
  újra a számláló minden csatlakozáskor.
- A kettős napló **node-oldala ezzel kész és bizonyított**. Az önálló,
  újrafelhasználható **ChirpStack frame-logger** eszköz (folyamatos, időbélyeges
  keret-mentés fájlba) és a soros-log-fájlba-időbélyegzés még megírandó - ezt a
  session csak kézzel/egyszeri lekérdezéssel demonstrálta.
- Diagnosztikai scriptek `/tmp`-ben (nem repó): `cs_list.py`, `cs_profiles.py`,
  `cs_provision.py`, `cs_verify.py`, soros capture `cap.py`.

## Kettős napló - teljes futtatás, igazolva (2026-10-17, ugyanaznap)
Kiderült, hogy a ChirpStack-oldali eszköz **már létezett**: `Tools/frame_logger/frame_logger.py`
(2026-10-02, NUCLEO-val igazolva) - a ROADMAP "még nincs megírva" sora elavult volt.
Ami hiányzott, a **node-oldali időbélyeges soros logger**: megírva ma,
`Tools/frame_logger/serial_logger.py` (pyserial, fali-óra időbélyeg, RTS/DTR
deasszertálva a néma-port elkerülésére, JSONL kimenet).

Éles futtatás: `frame_logger.py --gateway ac1f09fffe1f4131 --device 05ec…8fe5`
(hálózat) + `serial_logger.py --port /dev/ttyUSB0` (node), párhuzamosan. A soros
port megnyitása rebootolta a node-ot, így egy teljes friss join+uplink ciklust
kaptam mindkét oldalról. Korreláció (gateway = rak7268v2, SF7/868.5 MHz, RSSI -52,
SNR 9):

| esemény | node-oldal (soros) | hálózat-oldal (ChirpStack) |
|---|---|---|
| join | `EV_JOINED` 14:33:31 | `JoinRequest` 14:33:27 (devNonce 49171) + `JoinAccept` 14:33:28 |
| DevAddr | `80BD9B` | `0080bd9b` (bit-pontos) |
| uplink 1 | Packet queued 1 @14:33:32 | `UnconfirmedDataUp` @14:33:33 |
| uplink 2-4 | @14:34:04 / :21 / :38 | `UnconfirmedDataUp` @14:34:05 / :22 / :39 |

A DevAddr bit-pontosan egyezik, a devNonce (49171) egyezik, minden uplink ~1 mp-en
belül mindkét naplóban megjelenik. A hálózati oldal ezen felül `UnconfirmedDataDown`
kereteket is mutat (ChirpStack MAC/ADR), amit a node-oldal nem lát - ez a kettős
nézőpont értéke. **A kettős napló módszertan ezzel end-to-end bizonyított**; mindkét
eszköz a repóban (`Tools/frame_logger/`). Hátralévő finomítás: a reset-on-open miatt
a sustained (join nélküli, folyamatos) rögzítéshez a node-ot érdemes külön
tápról/already-joined állapotban hagyni, vagy a logger megnyitását reset nélkül
megoldani.
