# STM32WL "ipari" eszköz - első felderítés (2026-09-09)

## Környezet

- Debug probe: ST-Link/V3, USB-n csatlakoztatva (`0483:374e`).
- Eszközök: `openocd` 0.12.0 + `stlink-tools` 1.8.0, most telepítve
  (`sudo apt-get install -y openocd stlink-tools`).
- **Jogosultsági akadály:** a nyers USB debug-interfész (`/dev/bus/usb/...`)
  csak root/`plugdev` csoportnak volt elérhető, és a friss telepítéssel
  érkező udev-szabály csak új eszköz-csatlakozáskor lép életbe - az
  ST-Link kihúzása/visszadugása után rendeződött (`crw-rw-rw-+ plugdev`).

## Megfigyelés 1 - SWD nem elérhető normál csatlakozással

```
$ st-info --probe
Failed to enter SWD mode
chipid: 0x000, dev-type: unknown

$ openocd -f interface/stlink.cfg -c "transport select hla_swd" -f target/stm32f1x.cfg -c init
Info : Target voltage: 3.274183
Error: init mode failed (unable to connect to the target)
```

A tápfeszültség-mérés (3.27V) működik - ez azt jelenti, hogy legalább a
VDD-sense vonal jól van huzalozva -, de a debug port maga nem válaszol.

## Megfigyelés 2 - reset alatti csatlakozással viszont igen

```
$ openocd ... -c "reset_config srst_only srst_nogate connect_assert_srst" -f target/stm32f1x.cfg -c init
Warn : UNEXPECTED idcode: 0x6ba02477
Error: expected 1 of 1: 0x1ba01477

$ st-info --probe --connect-under-reset
chipid: 0x497
dev-type: STM32WLEx
flash: 262144 (pagesize: 2048)
sram: 65536
```

A `connect_assert_srst`/`--connect-under-reset` azt jelenti: az ST-Link
lenyomva tartja a target NRST-jét, MIALATT csatlakozik SWD-n - így a debug
port a reset-állapot szerinti (alapértelmezett, engedélyezett) módban van,
mielőtt a firmware egyáltalán elindulna és bármit tehetne a lábakkal.

**Értelmezés:** a firmware valószínűleg fut közben átkonfigurálja/tiltja a
SWD lábakat (GPIO-ra állítja őket, vagy explicit letiltja a debug portot) -
ez egy szoftveres, tehát **reset-alatti csatlakozással megkerülhető**
védelem, nem egy hardveres/RDP-szintű zár. A tapasztalat: ez a fajta
"védelem" gyakorlatilag semmit nem ér egy fizikai hozzáféréssel rendelkező
támadó ellen, mert az `openocd`/`st-flash` alapból támogatja ezt a
csatlakozási módot, nem kellett hozzá semmi extra trükk.

## Megfigyelés 3 - teljes, sikeres firmware-dump -> RDP valószínűleg Level 0

```
$ st-flash --connect-under-reset read flash_dump_20260909_085958.bin 0x08000000 0x40000
```

262144 bájt (teljes flash) sikeresen kiolvasva, hibaüzenet nélkül. A
tartalom valódi, struktúrált kód:
- Első szó (SP): `0x20000aa0` - érvényes SRAM-cím (SRAM bázis
  `0x20000000` + eltolás), pontosan illik a 64 KiB SRAM-hoz.
- Második szó (Reset_Handler): `0x0800014c` (Thumb-bit levágva) - érvényes
  flash-cím.
- Entrópia szegmensenként: `0x0000-0xFFFF` valódi kód-jellegű (6.8, 6.0
  bit/byte), `0x10000`-től a dump végéig (`0x40000`) végig `0xFF` -
  vagyis programozatlan flash, a tényleges image csak ~64 KiB.

Ha RDP Level 1 vagy 2 aktív lenne, ez a fajta teljes, struktúrált olvasás
tipikusan nem sikerülne (vagy csak nullákat/hibát adna vissza már a
kapcsolódás szintjén, jóval a `--connect-under-reset` trükk előtt). Mivel a
kapcsolódás után egy tökéletesen értelmes image jött vissza, ez erős jel
arra, hogy **RDP nincs bekapcsolva** ezen a konkrét eszközön - a
"biztonság" itt kizárólag a (megkerülhető) szoftveres SWD-tiltásra épül.
Az RDP pontos regiszter-szintű megerősítése (FLASH_OPTR kiolvasása) még
hátravan - az OpenOCD `stm32l4x.cfg`-je más DPIDR-t vár
(`0x2ba01477` vs. a tényleges `0x6ba02477`), ehhez a WL-családra pontosan
illő konfiguráció vagy explicit ID-felülbírálás kell.

## Megfigyelés 4 - a firmware string-jei alapján ez gyári/demo firmware-nek tűnik

`strings` a dumpon: `AT+LOST=`, `AT+MOD_OK=`, `AT+RCV=`, `AT+VL`,
`REGION`/`SUBREGION`, `Beacon Tx at %d`, `HDR OK`/`SYNC OK`/`PRE OK` - ez
egy AT-parancsos, régió-/moduláció-váltós LoRa sniffer/analizátor
referencia-alkalmazásra hasonlít (Semtech/ST gyári demo stílus STM32WL
Nucleo boardokhoz), nem tűnik nyilvánvalóan egyedi, gyártó-specifikus
alkalmazásnak.

**Ezt tisztázni kell, mielőtt bármilyen erős következtetést
vonnék le a termék biztonságáról** - lehet, hogy ez csak
egy fejlesztői board gyári firmware-rel, nem a tényleges vizsgálandó
"ipari" termék éles szoftvere.
