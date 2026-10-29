# NUCLEO-WL55JC1 - SWD-elérési kísérlet újraflasheléshez (2026-10-02)

**Cél:** a NUCLEO-t rádugni és előkészíteni a `base_node` firmware
flashelésére (ld. `Firmwares/Nodes/NUCLEO-WL55JC1/base_node/README.md`),
útközben „mai dátummal" újra megerősíteni a debug-elérést és az RDP-szintet
(ld. `2026_09_19_stm32wl_rdp_register_confirmed.md`).

## Hardver-állapot

- `lsusb`: `0483:374e STMicroelectronics STLINK-V3` - egyetlen ST-eszköz,
  a board beépített ST-Linkje (egybevág a 09-19-i megfigyeléssel).
- `/dev/ttyACM0` jelen van (`ID_MODEL=STLINK-V3`, firmware `V3J7M3`).
- OpenOCD szerint **target feszültség 3.27 V** - a céloldali STM32WL tápon van.

## Mit próbáltam és mi lett az eredménye

### 1. `st-info --probe` (nyílt forrású stlink-tools) - elhasalt
```
Failed to enter SWD mode
Found 1 stlink programmers  (version V3J7)
chipid: 0x000   dev-type: unknown
```
Az ST-Link maga látszik, de a target magba nem lépett be. A nyílt forrású
`stlink` (`st-info`/`st-flash`) nem boldogul ezzel a STM32WL + ST-Link V3
kombóval.

### 2. OpenOCD 0.12.0 + `target/stm32wlx.cfg` (hla_swd) - részben
- **Első próba:** a mag **detektálódott** -
  `[stm32wlx.cpu0] Cortex-M4 r0p1 processor detected`. A `FLASH_OPTR`
  (`0x58004020`) olvasás viszont `0x00000000`-t adott, mert a CPU **nem
  volt megállítva** (a `connect_assert_srst` nem lépett életbe, ld. lent).
- `reset_config connect_assert_srst` -> `Error: BUG: can't assert SRST`:
  az ST-Link **hla** transportja ebben a buildben nem tud SRST-t
  asszertálni, tehát reset alatti csatlakozás így nem megy.
- `transport select dapdirect_swd` -> `Debug adapter doesn't support
  'dapdirect_swd' transport`: ez a transport nincs belefordítva ebbe az
  OpenOCD-be, úgyhogy ezen az úton sem lehet reset alatt csatlakozni.
- **Init+halt, 6 ismételt próba** egy ébren-ablak elkapására ->
  mind a 6: `init mode failed (unable to connect to the target)`.

## Értelmezés

- **Nem RDP.** A mag egyszer detektálódott (`Cortex-M4 r0p1`) - egy
  RDP-zárolt chipnél ez sem menne. Ez egybevág a 09-19-i,
  regiszter-szinten mért **RDP Level 0** (`FLASH_OPTR = 0x3ffff0aa`,
  RDP-bájt `0xAA`) eredménnyel. A korábban felvetett „a debug-portot RDP
  zárja" hipotézis **ma is megdőlt.**
- **A valódi ok:** a chipen jelenleg a **gyári (alvó) firmware** fut, ami
  STOP/low-power módban elejti a debug-órajelet, így az SWD csak egy rövid
  ébren-ablakban élne. Ehhez **reset alatti csatlakozás** („connect under
  reset") kellene - amit a jelenlegi OpenOCD (csak `hla_swd`, nincs
  `dapdirect_swd`) **nem tud** ezen a gépen.

## Megoldás ugyanaznap - pyOCD under-reset (st.com nélkül)

A CubeIDE letöltése nem ment (az st.com oldal nem engedte,
már korábban sem). Third-party mirror helyett nyílt eszközökkel kerültem
meg, **mind elérhető st.com-fiók nélkül**:

1. `sudo apt install python3-pyocd gcc-arm-none-eabi` (Debian trixie).
2. A `pyocd pack` alparancshoz hiányzott a `cmsis-pack-manager` (nincs
   apt-ban) -> `pip install --user --break-system-packages cmsis-pack-manager`.
   (A pip figyelmeztet pyocd dep-konfliktusra - libusb-package/pylink/
   capstone -, de az apt-pyocd ettől működik, figyelmen kívül hagyható.)
3. `pyocd pack update && pyocd pack install stm32wl55jcix` - a Keil
   STM32WL DFP pack (`Keil.STM32WLxx_DFP` 2.1.0) a **keil.arm.com**-ról
   jön, nem st.com-ról.

Working parancs és eredmény:
```
pyocd cmd -t stm32wl55jcix -M under-reset -c "read32 0x58004020"
58004020:  3ffff0aa
```
`FLASH_OPTR = 0x3ffff0aa`, RDP-bájt `0xAA` = **RDP Level 0** - bit-pontosan
egyezik a 09-19-i OpenOCD-s méréssel, most reset alatt, mai dátummal.
(Mellékes zaj: `AP#1 IDR fault` / a `0x5c001000` olvasás elhasal - az
ST-Link a második AP-t nem éri el; a `FLASH_OPTR` olvasását nem érinti.)

**Kulcs tanulság (tooling):** ehhez a boardhoz ezen a gépen a működő SWD-út
a **pyOCD `-M under-reset` a `stm32wl55jcix` targettel**. Az `st-info`
(nyílt stlink) és az OpenOCD 0.12 `hla` (nincs `dapdirect_swd`, „can't
assert SRST") NEM jó. STM32CubeProgrammer/CubeIDE **nem szükséges** a
debug/flash csatlakozáshoz - a korábbi ilyen következtetés elavult.
