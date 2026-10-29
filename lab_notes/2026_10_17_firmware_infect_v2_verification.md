# Firmware_Infect end-to-end igazolás valódi V2 dumpon, 2026-10-17

**Cél:** a `Hacking/Firmware_Infect` README + mérési eredmények véglegesítése
(ROADMAP 1. sáv, eredeti határidő 2026-10-10 - lecsúszott, most behozva). A tool
eddig kész volt, de **nem volt dokumentált valódi, végponttól-végpontig futtatás**;
ez pótolja azt.

## Célpont (ismert kulcsokkal, hogy a kinyerés ellenőrizhető legyen)
- Heltec **WiFi LoRa 32 V2** (`/dev/ttyUSB0`), rajta a `base_node` OTAA firmware a
  valódi V2 Node 1 kulcsokkal beégetve (ugyanaz a bináris, amit a dual-log
  bringuphoz flasheltem, ld. `2026_10_17_v2_base_node_dual_log_bringup.md`).
- Chip: ESP32-D0WDQ6 rev v1.1, 8 MB flash (esptool `flash_id`).

## Lépések és eredmények
1. **Dump:** `esptool --port /dev/ttyUSB0 read_flash 0 0x200000 v2_victim_dump.bin`
   (első 2 MB, 0-tól, hogy a tool abszolút offsetjei - app @0x10000 - stimmeljenek).
2. **Auto-detektálás** (`main.py dump.bin --no-compile`, semmi hint):
   - Board: **Heltec V2 / SX1276, 100 %**
   - Régió: **EU868** (40 %, string_pattern)
   - Payload: **counter / fPort 2**, `sx1276_lmic` template
   - Kulcs (**Strategy B, entrópia-scan**): az **első** entrópia-klasztert vette
     `0x0100b0`-nál -> **HIBÁS** (AppKey `08F408C8…`), nem a valódi kulcsblokk.
3. **Strategy A** (`--appkey <ismert>`):
   - AppKey megtalálva `0x019cfc`-nél, DevEUI `0x019d0c` (+16), AppEUI `0x019d14`
     (+24) a szomszédos LMIC-layoutból -> **mindhárom bit-pontosan helyes**, helyes
     endiannal (DevEUI/AppEUI LSB->MSB visszafordítva). Egyezett az
     `Information.md`-beli V2 Node 1 értékekkel (itt nem reprodukálva - titok).
4. **Generálás + fordítás:** `rogue_node.ino` legenerálva, és `arduino-cli`-vel
   **tisztán lefordult** a `heltec_wifi_lora_32_V2` FQBN-re (9 % flash). A sketch
   sima LMIC (nincs OLED), így a V2 `base_node`-nál látott Heltec-library
   ütközések itt fel sem merültek.

## Tanulság
A **Strategy B (entrópia-scan) önmagában megbízhatatlan**: egy valódi ESP32
app-image-ben az első magas-entrópiájú 16 bájtos blokk gyakran **nem** a
kulcsblokk (itt `0x0100b0` vs valós `0x019cfc`). Fekete-dobozos dumpnál tehát vagy
`--appkey` kell, vagy a klaszter-jelöltek rangsorolása a mögöttük lévő
DevEUI/AppEUI hihetőségével (ez utóbbi egy érdemi továbbfejlesztési irány). A
board/régió/payload auto-detektálás viszont valódi dumpon is pontos volt.

## Szándékos korlátok
- A rogue node-ot **nem flasheltem/joinoltam** - duplikált eszköz a live
  hálózaton; a rogue-klón join amúgy is külön igazolt
  (`Hacking/Application_Key_Request`, 2026-09-19). Ha kell élő rogue-join demo,
  az külön, tudatos lépés.
- Artefaktumok `/tmp`-ben (nem repó, valódi kulcsot tartalmaznak):
  `v2_victim_dump.bin`, `fi_out/`, `fi_out_a/`.
