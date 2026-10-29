# STM32WL - a négy támadási forgatókönyv alkalmazhatósága (2026-10-03)

**Kérdés:** a négy támadási
forgatókönyv (SDR sniff, SPI-tap, AppKey brute-force, firmware-dump) közül
melyik vihető át az integrált rádiós STM32WL-osztályú csomópontra
(NUCLEO-WL55JC1), szemben a külső rádiós Heltec/ESP32 node-dal?

**Rövid válasz:** 3/4 átvihető; az SPI-tap architekturálisan nem
értelmezhető. A firmware-dump egy provisioning-lépéssel (RDP1) kivédhető.

## Mátrix

| Támadás | STM32WL-en | Bizonyíték / indok |
|---|---|---|
| **Debug-port / konfiguráció-kiolvasás** | **átvihető**, RDP1 kivédi | Lásd lentebb: a lényeg NEM a dump ténye (triviális egy védtelen chipen), hanem a **biztonsági konfiguráció jellemzése** (RDP/WRP/PCROP) és hogy minden védelem kikapcsolt. RDP1-en a kiolvasás tiltott (`2026_10_02_stm32wl_rdp_hardening_demo.md`). |
| **SPI-busz tap** | **nem alkalmazható** | A STM32WL **integrált** MCU+rádió: a rádiómaghoz a dedikált, chipen belüli SUBGHZSPI-n keresztül fér hozzá a CPU - **nincs külső MCU↔rádió SPI-busz**, amire logikai analizátort lehetne kötni. A Heltec/ESP32 (külső SX127x/SX126x) node-on ez pont a tappelhető felület; a STM32WL-en ez a fizikai csatorna megszűnik. |
| **AppKey brute-force** | **átvihető** (node-független) | A támadás a Join~Request MIC-jén dolgozik, nem a node hardverén. A MIC rádión/gateway-en elkapható (a gateway-frame log rögzítette a STM32WL Join Requestjeit `dev_nonce`+`mic` mezőkkel). Ugyanaz a brute-force pipeline fut, mint a Heltec node ellen. **Védelem:** erős, véletlenszerű AppKey - a tesztkulcsom (random 128 bit) ellenáll; csak gyenge/alapértelmezett kulcs esetén sikeres. |
| **Passzív SDR sniff** | **átvihető** | Rádiófizikai, node-független: a STM32WL ugyanúgy EU868-on, 868.1/868.3/868.5 MHz, SF/BW szerint sugározza a Join~Request/uplink kereteket. A gateway-frame log (frame_logger) ezt már rögzítette on-air (frekvencia, SF, PHYPayload). **Önálló RTL-SDR-es élő demó** (nem a gateway-en át) az RTL-SDR bedugásával végezhető el, a gateway-frame loggal keresztvalidálva. |

## A debug-port valódi tartalma: KONFIGURÁCIÓ-JELLEMZÉS (nem a dump)

Egy olvasásvédelem nélküli (RDP0) chip flash-ének kiolvasása SWD/USB-n
**triviális** - bármelyik debugger megteszi, önmagában nem eredmény. A
tudományos tartalom az, hogy a debug-porton keresztül **jellemezhető a
device tényleges biztonsági konfigurációja**: mely védelmek vannak be- vagy
kikapcsolva. Ez reprodukálható a FLASH védelmi regisztereinek kiolvasásával
(2026-10-03, élő mérés, NUCLEO-WL55JC1):

| Regiszter (cím) | Mért érték | Dekódolás |
|---|---|---|
| `FLASH_OPTR` (0x58004020) | `0x3BFFF0AA` | RDP-mező `0xAA` -> **olvasásvédelem: Level 0 (KI)**; BOR_LEV=0 |
| `FLASH_WRP1AR` (0x2C) | `0x0000007F` | STRT=0x7F > END=0x00 -> **write-protect A: KI** |
| `FLASH_WRP1BR` (0x30) | `0x0000007F` | STRT>END -> **write-protect B: KI** |
| `FLASH_PCROP1ASR/AER` (0x24/0x28) | `0xFF` / `0x00` | STRT>END -> **PCROP (kód-olvasásvédelem) A: KI** |
| `FLASH_PCROP1BSR/BER` (0x34/0x38) | `0xFF` / `0x00` | -> **PCROP B: KI** |

**Megállapítás:** a vizsgált eszközön **minden flash-szintű védelem
alapértelmezett, azaz kikapcsolt állapotban van** (RDP0, WRP ki, PCROP ki).
Ez a konkrét sérülékenység-jellemzés - nem az, hogy „ki lehet olvasni",
hanem hogy a konfiguráció semmilyen védelmet nem aktivál, így a firmware és
a benne tárolt kulcsanyag teljesen exponált. Ugyanez volt igaz a gyári
„ipari" egységre is (RDP0, ld. `2026_09_19_stm32wl_rdp_register_confirmed.md`).

**Hardening (igazolva/ajánlott):** RDP~Level~1 (kiolvasás tiltása -
hardveresen igazolva), kiegészítve a bootloader-régió write-protectjével
(WRP) és a kulcstároló PCROP-jával. Ezek bekapcsolt állapotát ugyanezekkel a
regiszterekkel lehet ellenőrizni/auditálni - a módszer tehát egyben
**konfiguráció-audit eszköz** is, nem csak támadás.

## Következtetés

- A STM32WL (integrált rádió) **nem old meg mindent**: a rádiós (SDR sniff) és
  a kulcs-szintű (brute-force) támadások változatlanul érvényesek, a
  firmware-dump is, amíg RDP0-n van.
- Az **egyetlen, amit a platform architektúrája önmagában megszüntet, a
  fizikai SPI-tap** - mert nincs külső rádió-busz. Ez konkrét, a hardverből
  következő előny a külső rádiós (Heltec/ESP32) node-okhoz képest.
- A firmware-dump **nem platform-hiba**, hanem konfiguráció: RDP1-gyel zárható
  (igazolva).

## Ami élő demóhoz eszközt igényel
- **Passzív SDR sniff élőben:** RTL-SDR vevő + antenna bedugása (a tooling a
  `oe-rtl-sdr-radio-scanner` külön repóban). A gateway-frame log a ground-truth.
