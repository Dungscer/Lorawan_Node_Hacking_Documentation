# NUCLEO fizikai/oldalcsatorna mérési protokoll (logikai analizátor + szkóp)

A 8. forgatókönyvhöz (ROADMAP). Cél: (a) biztonsági szög - a W25Q SPI-buszon
fizikai hozzáféréssel a stagelt firmware tisztán lehallgatható; (b) mérnöki
hullámformák a dolgozatba (OTA-dekód, STOP2 before/after, időzítés). Turnkey
lépéssor, hogy a hardvernél csak csipesz + rögzítés legyen.

## Bekötés (NUCLEO Arduino-fejléc)
- SCK  = PA5 / D13
- MISO = PA6 / D12
- MOSI = PA7 / D11
- CS   = PA4 / D10
- GND  = bármelyik GND tüske (közös föld a LA-val kötelező)

## Logikai analizátor beállítás
- Mintavétel: >= 2 MHz bőven elég (a SPI ~187 kHz az appban, ~125 kHz a
  bootloaderben); 24 MHz-es olcsó LA is tízszeres ráhagyással jó.
- SPI-dekóder: CPOL=0, CPHA=0 (mode 0), MSB-first, 8 bit, CS aktív-alacsony,
  órajel SCK, adat MOSI (write) és MISO (read).
- Trigger: CS lefutó él.

## 1. felvétel - OTA SPI-forgalom dekódolva (működő firmware)
1. Flash: `otaa_basic_GOOD_fixed.bin` @0x08000000
   (`st-flash --connect-under-reset --reset write <bin> 0x08000000`). Join bevárása.
2. Push egy kis aláírt képet (`Tools/OTA_Update`, pl. 250 B, chunk-size 50).
3. A LA-n dekódolva láthatók a W25Q-parancsok: `0x06` write-enable, `0x20`
   sector-erase, `0x02` page-program (+ az adatbájtok a MOSI-n), `0x05` read-SR1.
4. **Biztonsági megfigyelés:** a `0x02` page-program adatbájtjai a MOSI-n a
   firmware-kép bájtjai, titkosítás nélkül -> fizikai buszhozzáféréssel a stagelt
   firmware kiolvasható (és a `0x20`/`0x02` befecskendezhető -> a külső flash
   manipulálható). Ez a külső-flash staging fizikai expozíciója. Ábra: a dekódolt
   bájtfolyam egy darabja a push adatával egyeztetve.

## 2. felvétel - STOP2/SPI1 hang before/after
1. **Hibás:** flash `otaa_basic_BROKEN_stop2_demo.bin`. Push egy képet; a manifest
   utáni első STOP2-ébredés után az SCK **lapos marad** (a driver örökké vár) ->
   a hang vizuális bizonyítéka. (A bin: a STOP2 re-init visszavonva, minden más
   azonos.)
2. **Javított:** flash `otaa_basic_GOOD_fixed.bin`. Ugyanaz a push; az SCK az
   ébredés után újra kapcsol, a transzfer végigmegy.
   (Mindkét bin elő van készítve: `/home/nagy_sz/nucleo_ota_measurement_bins/`.)

## 3. felvétel - időzítés (szkóp vagy LA)
1. Működő firmware, push közben.
2. Mérd CS felfutásától (parancs vége) a következő CS lefutásáig / a `0x05`-ös
   SR1-poll BUSY-bitjének tisztulásáig egy **sector-erase** (`0x20`) és egy
   **page-program** (`0x02`) tényleges idejét.
3. Ez számszerűsíti, miért kell nem-blokkoló állapotgép: egy blokkoló
   egész-image erase ennyiszer a szektorszám ideig tartaná fel a rádió-stacket.

## Szkóp másodlagosan
- Jel-integritás / órajel-élek a /256 (~187 kHz) és /32 (~125 kHz, bootloader)
  sebességen; ha később gyorsabb SPI-ra mennénk, a breadboard-huzalozás
  jelromlása itt látszódna.

## Mentés
- Hullámforma-képernyőképek -> a dolgozat ábrái (`diploma/`).
- Rövid szöveges értékelés (mit mutat, számok) -> aznapi `lab_notes/` megfigyelés.
