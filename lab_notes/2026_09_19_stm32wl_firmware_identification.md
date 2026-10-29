# STM32WL "ipari" eszköz - firmware azonosítás és mélyebb elemzés (2026-09-19)

## Hardver azonosítás - megerősítve hivatalos ST adatokkal

A korábban `st-info`-val kiolvasott adatok (chipid `0x497`, 256 KiB flash,
64 KiB SRAM) **pontosan egyeznek** az ST hivatalos NUCLEO-WL55JC
specifikációjával: STM32WL55JC, dual-core (Cortex-M4/M0+, 48 MHz),
UFBGA73, 150–960 MHz-es RF transceiver (LoRa/(G)FSK/(G)MSK/BPSK), 256 KB
flash, 64 KB SRAM. (Forrás: ST hivatalos termékoldal és UM2592
felhasználói kézikönyv, ld. lent.) Ez a hardver-oldal genuine - legalábbis
funkcionálisan/specifikáció-szinten pontosan úgy viselkedik, ahogy egy
valódi STM32WL55JC-nek kellene.

## Firmware azonosítás

A dumpból korábban kinyert string-ek (`AT+VL`, `AT+MOD_OK=`, `AT+RCV=`,
`REGION`/`SUBREGION`, `Beacon Tx at %d`) alapján utánanéztem az ST
hivatalos STM32CubeWL csomagjának:

- **`AT+VL` szó szerint megtalálható** az ST hivatalos
  `STM32CubeWL/Projects/NUCLEO-WL55JC/Applications/SubGHz_Phy/SubGHz_Phy_AT_Slave/SubGHz_Phy/App/subg_at.c`
  forrásfájlban (GitHub:
  `STMicroelectronics/STM32CubeWL`, `main` branch, ellenőrizve
  2026-09-19-én). Ez az "AT verbose level" beállító parancs, generikus
  diagnosztikai funkció az ST AT-parancs keretrendszerében.
- A `SubGHz_Phy_AT_Slave` alkalmazás leírása (readme.txt) pontosan
  illik a megfigyelt viselkedésre: *"This application is just for RF TEST
  controlled though AT command interface over UART by an external host"*
  - ez egy NUCLEO-WL55JC-re szánt, AT-parancsos RF teszt/PHY-eszköz, nem
  egy csatlakoztatott LoRaWAN végpont.
- Közvetett megerősítés webes keresésből (ST hivatalos anyagokból
  idézve): *"Frequency hopping and listen before talk are used only in
  subregions where necessary due to regulations in the STM32CubeWL
  demonstration firmware"*, illetve *"A concentrator sends one beacon
  frame every 16 seconds"* - ezek **pontosan** megfelelnek a dumpban
  talált `REGION`/`SUBREGION` és `Beacon Tx at %d` string-eknek.

**Fontos árnyalás:** a `REGION`/`SUBREGION`-váltós, több-modulációs,
beacon-detektáló "sniffer" funkcionalitás **nem** része a jelenlegi
mainline `SubGHz_Phy_AT_Slave` alkalmazásnak (a GitHub-on elérhető
forrásban ez nincs benne) - tehát vagy egy **korábbi/eltérő STM32CubeWL
verzió** tartalmazta ezt (a csomagnak több kiadása volt), vagy ez egy, az
ST AT-keretrendszerére épülő **egyedi/közösségi kiterjesztés** (találtam
egy ST community fórumbejegyzést is, ahol valaki kifejezetten
"beacon sniffer és packet sniffer" használatot keresett STM32WL55-ön -
tehát ez egy ismert, létező igény/irány, nem elképzelhetetlen egyedi
build).

**Következtetés (óvatosan megfogalmazva):** a firmware **ST hivatalos AT-parancs keretrendszerére
épül** (megerősítve konkrét kódegyezéssel), de a pontos, teljes
alkalmazás-verzió nem azonosítható 100%-osan a jelenlegi mainline
forrásból - ez **nem** von le semmit a hitelességéből (ez lehet simán egy
korábbi CubeWL-kiadás vagy egy AT-keretrendszerre épülő bővített
sniffer-build), de nem is jelenthető ki teljes bizonyossággal, hogy
bit-pontosan egyezik egy adott, publikusan elérhető gyári image-dzsel.

## Célzott kulcskeresés a dumpban (a 2026-09-09-i elmaradt feladat)

A `Hacking/Firmware_Infect/extractor.py`-ban már létező, kidolgozott
entrópia-alapú kulcskereső logikát (`_scan_region`/`_entropy_ok` - két
egymáshoz közeli, 16 bájtos, magas-entrópiájú blokk keresése, ami egy
AppKey+DevEUI+AppEUI LMIC-stílusú, egymás melletti tárolási mintára utal)
lefuttattam a dump kód-tartalmú részén (`0x0000`–`0x10000`):

```python
from extractor import _scan_region
result = _scan_region(data, 0, 0x10000)
# -> talált egy jelöltet offset 0x160-nál
```

**A talált jelölt hamis pozitívnak bizonyult**: az "AppKey" bájtjai
`FE E7 FE E7 FE E7 FE E7 ...` mintázatot mutatnak - ez az ARM Thumb `b .`
(önmagára ugró végtelen ciklus, gyakori "trap"/kitöltő utasítás
megszakítás-vektoroknál) opkódja (`0xE7FE`) ismételve, nem egy valódi
kriptográfiai kulcs. A `set(bájtok) >= 8` entrópia-küszöb véletlenül
átment rajta a környező, ténylegesen változatos vektortábla-tartalom
miatt.

**Értelmezés:** ez a firmware valószínűleg **nem tárol LoRaWAN join
kulcsot (AppKey) hagyományos, LMIC-stílusú elrendezésben** - ami
konzisztens azzal, hogy ez egy PHY-szintű AT-teszt/sniffer eszköz, nem egy
OTAA-csatlakoztatott végpont. (Ez nem zárja ki, hogy máshol, más
formátumban legyen benne titkosítási anyag - csak azt jelenti, hogy az
ESP32/LMIC-stílusú "AppKey+DevEUI+AppEUI egymás mellett" minta nem
található meg.)

## Dokumentáció (linkek, nem letöltve - ld. lent miért)

- **UM2592** - NUCLEO-WL55JC (MB1389) felhasználói kézikönyv:
  https://www.st.com/resource/en/user_manual/um2592-stm32wl-nucleo64-board-mb1389-stmicroelectronics.pdf
- **AN5481** - LoRaWAN AT commands for STM32CubeWL (a teljes AT-parancs
  referencia): https://www.st.com/resource/en/application_note/an5481-lorawan-at-commands-for-stm32cubewl-stmicroelectronics.pdf
- **UM2786** - STM32CubeWL demonstration firmware (a demo-alkalmazások
  leírása): https://www.st.com/resource/en/user_manual/um2786-stm32cubewl-demonstration-firmware-stmicroelectronics.pdf
- Forrás-egyezés: `STMicroelectronics/STM32CubeWL` GitHub repó,
  `Projects/NUCLEO-WL55JC/Applications/SubGHz_Phy/SubGHz_Phy_AT_Slave/`

**Miért nincs letöltve/bemásolva a PDF maga a repóba:** a rendelkezésre
álló eszközökkel (WebFetch/WebSearch) nem tudtam nyers PDF-bájtokat menteni
lemezre (csak feldolgozott/összefoglalt szöveget kapok vissza), a direkt
`curl` pedig nem ér el külső internetes hosztokat erről a gépről. A
linkek stabilak (st.com hivatalos anyagok), és nem tűnt indokoltnak
ST szerzői jogvédett PDF-jeit újraterjeszteni a repóban, amikor egy stabil
hivatkozás ugyanazt a célt szolgálja.
