# STM32WL - RDP hardening kivédi a debug-port bypass-t (2026-10-02)

**Kérdés:** a debug-porton át végzett
flash-kiolvasás ("debug-port bypass"), amit a gyári firmware-en demonstráltam
és egész nap használtam a `base_node` flasheléséhez/olvasásához, a platform
sebezhetősége-e, vagy pusztán **hardening-elmaradás** (RDP Level 0)?

**Válasz: hardening-elmaradás.** Egyetlen option-byte lépés (RDP Level 1)
teljesen kivédi a kiolvasást, miközben a node funkcionálisan sértetlen marad.

## Módszer

- Firmware megállítása: `pyocd cmd -t stm32wl55jcix -M under-reset -c "reset halt"`.
- RDP írása/olvasása: OpenOCD `stm32wlx option_write 0 0x20 <val> 0x000000ff`
  (CSAK az RDP-bájt, maszkolva) + `option_load`, majd friss session a
  latch-elt állapotra.
- **Biztonság:** kizárólag `0xAA` (L0) és `0xBB` (L1). `0xCC` (L2, VÉGLEGES)
  soha nem került kiírásra.

## Eredmények

### RDP Level 0 (0xAA) - a bypass működik
`OPTR = 0x3bfff0aa`. SWD-n a teljes flash olvasható:
```
mdw 0x08000000 8
0x08000000: 20008000 08019e05 08017697 08017699 0801769b 0801769d 0801769f 00000000
```
(a `base_node` vektortáblája - a firmware bitről bitre kinyerhető.)

### RDP Level 1 (0xBB) - a bypass kivédve, a node működik
`OPTR = 0x3bfff0bb`. A debug-probe még **csatlakozik** (`Cortex-M4 detected`),
de utána:
- `mdw 0x08000000` (flash) -> **semmi** (olvasás elutasítva).
- `mdw 0x20000000` (RAM) -> **semmi**.
- halt után a core-regiszterek: `pc: 0xfffffffe msp: 0xfffffffc` - azaz a
  debugger mindenre `0xffffffff`-et lát. **Se flash, se RAM, se
  regiszterállapot nem nyerhető ki.**
- **Közben a node tovább működik:** az RDP1 OBL-reset után újracsatlakozott
  (DevAddr `00558445`, fCntUp=4, friss last_seen) - a hardening nem rontja
  el a működést, csak a kiolvasást zárja.

### Regresszió RDP1 -> RDP0 - mass-erase (a firmware megsemmisül)
`option_write` RDP=`0xAA` + `option_load` után (friss session):
```
OPTR: 0x3bfff0aa          (RDP0 vissza)
mdw 0x08000000 4: ffffffff ffffffff ffffffff ffffffff   (TÖRÖLVE)
```
A szint-csökkentés **hardveres teljes-törlést** vált ki: a tartalom nem
kinyerhető, csak megsemmisíthető. (A boot-option bitek - nBOOT0=1 - túlélték.)

## Következtetés

- A gyári "ipari" NUCLEO-WL55JC1 kiolvashatósága (ld.
  `2026_09_19_stm32wl_rdp_register_confirmed.md`) azért állt fenn, mert az
  eszköz **RDP Level 0-n szállt** - nem a hardver/toolchain hibája.
- A védelem **egyetlen option-byte lépés** (RDP1), ami: (a) megállítja a
  debug-porton át történő flash/RAM/regiszter kiolvasást, (b) nem rontja a
  node működését, (c) visszafordítható, de **csak a tartalom törlésével**
  (regresszió = mass-erase) - tehát a kulcsok/firmware nem szivárognak ki.
- Ezért a `otaa_secure` tier egyik konkrét hardening-lépése **RDP1 beállítása**
  a gyártási/provisioning végén. (RDP2-t nem teszteltem: végleges, és
  éles eszközön ritkán indokolt.)

## Állapot / helyreállítás

A kísérlet után a `base_node` újraflashelve (OpenOCD `Verified OK`), de a sok
option-byte művelet miatt a boot-latch **tápciklust igényel** a flash-boothoz
(az `option_load` ST-Linken át itt is "failed"-et jelez; a valódi POR latch-el).
Tápciklus után a node a flashből indul és újracsatlakozik.
**Megerősítve:** tápciklus után reset-halt PC=`0x08019e04` (flash-boot), és a
node újracsatlakozott (DevAddr `00c10866`, friss uplinkek) - a `base_node`
teljesen helyreállítva, a kísérlet nem hagyott maradandó nyomot a boardon.
