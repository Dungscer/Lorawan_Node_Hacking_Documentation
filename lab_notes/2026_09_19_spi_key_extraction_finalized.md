# SPI_Key_Extraction - PoC lezárása, SX1262 (V3/V4) támogatás hozzáadva (2026-09-19)

## Amit találtam induláskor

A `sx1262.py` fájl már régóta létezett, **teljesen kidolgozott** portolási
tervvel és kész helper-függvényekkel (`is_fifo_write`,
`payload_from_transaction`, `CMD_NAMES`) - csak sosem lett bekötve a
tényleges detekciós logikába (`csv_parser.py`/`atkdl_loader.py` TODO-kommentek
mutattak rá, mit kellene csinálni). A tényleges munka tehát a már megtervezett
integráció elvégzése volt, nem új tervezés.

## Elvégzett munka

1. `csv_parser.py` és `atkdl_loader.py` `find_fifo_writes()` - `chip`
   paraméter (`"sx1276"` alapértelmezett, `"sx1262"` opció), ami a
   meglévő `sx1262.py` helpereket hívja SX1262 esetén.
2. `main.py` - `--chip {sx1276,sx1262}` CLI-kapcsoló mindkét (`csv`,
   `atkdl`) subcommandhoz; payload-offset (`mosi[1:]` vs. `mosi[2:]`),
   banner, összegző tábla és a JSON-mentés chip/board mezői mind
   chip-függővé téve.
3. `utils.py` - a statikus `BANNER` stringből `banner(chip, board)`
   függvény lett, hogy a fejléc a tényleges chipet/boardot mutassa.
4. Új unit tesztek (`tests/test_csv_parser.py`,
   `tests/test_atkdl_loader.py`): SX1262 WriteBuffer detekció,
   nem-WriteBuffer parancsok kiszűrése, és hogy az alapértelmezett
   SX1276-út nem változott.

## Ellenőrzés

- **Regresszió:** a valós V2 capture (`capture.atkdl`) pontosan ugyanazt
  az eredményt adja, mint a módosítás előtt (`--chip` argumentum nélkül
  is, alapértelmezett `sx1276`-tal is).
- **Új teszt-suite:** 31/31 pytest teszt zöld (27 régi + 4 új).
- **Végponttól-végpontig smoke teszt (CLI szinten, szintetikus adat):**
  egy kézzel összerakott, valós Join Request formátumú (23 bájt: MHDR +
  AppEUI + DevEUI + DevNonce + MIC) SX1262 `WriteBuffer` tranzakciót
  `python3 main.py csv --format raw --chip sx1262`-vel futtatva a teljes
  lánc (detekció -> payload-kinyerés -> LoRaWAN-dekódolás -> megjelenítés)
  helyesen működött, minden mező (AppEUI, DevEUI, DevNonce, MIC) pontosan
  visszakapva.
- **Nem ellenőrizve:** valódi V3/V4 hardveren, logikai analizátorral
  rögzített SPI capture - ilyen jelenleg nincs a repóban. Amíg nincs
  ilyen, ez egyelőre "implementálva", nem
  "hardveren ellenőrizve".

## Mérési eredmények (V2, valós capture)

A meglévő `capture.atkdl`-t lefuttatva pontos számok kerültek elő:
33 554 432 minta 20 MHz-en = **~1.68 másodperces** capture ablak, 571 SPI
tranzakció, 1 FIFO write (a Join Request), a kinyert AppEUI/DevEUI
pontosan egyezik a node valós regisztrált OTAA hitelesítő adataival (a
konkrét kulcsértékeket nem jegyeztem le, azok titkosak).

## Pin-kiosztás pótolva

A README korábbi TODO-i (V3/V4/STM32 pin-táblák) kitöltve: V3 és V4
azonos pinout-tal rendelkezik (NSS=8, SCK=9, MOSI=10, MISO=11, RST=12,
BUSY=13, DIO1=14) - V3-nál a Heltec ESP32 core `pins_arduino.h`-jából,
V4-nél a `base_node.ino` saját `#define`-jaiból megerősítve, mindkettő
egyezik. STM32-nél jelezve, hogy nem alkalmazható (integrált rádió, nincs
külső SPI-kapcsolt transceiver, ld. `STM32WL_Industrial` külön szál).

## Ismert, nem javított hiányosság

A `--verbose` tábla (`atkdl_display.py`) még mindig SX1276 regiszter-nevekkel
jelenít meg minden tranzakciót, `--chip sx1262` esetén is - ez csak a
diagnosztikai kimenetet érinti, a tényleges FIFO-write detekciót/kulcs-
kinyerést nem. Dokumentálva a README "Known Limitation" szakaszában.
