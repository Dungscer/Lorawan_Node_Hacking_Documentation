# STM32WL55 hardveres védelmi mechanizmusok - mit kellene fizikailag tesztelni

**Kontextus:** a felhasználó kérdésére ("milyen védelmi mechanizmusokat
tud majd ez az eszköz, amiket tesztelni kellene fizikailag") 2026-09-28-án
összeállított lista. A STM32WL55 CMSIS eszközfejléce
(`stm32wl55xx.h`, `github.com/STMicroelectronics/cmsis_device_wl`) ellen
ellenőrizve, nem csak általános STM32-tudásból felidézve - a
perifériák/bitek *létezése* emiatt megbízható, a pontos *működésük*
viszont még a referencia-kézikönyvet (RM0453) igényli, mielőtt a
dolgozatba tényként kerülne. Semmi nincs még fizikailag letesztelve.

Ugyanez a tartalom bekerült a
`Firmwares/Nodes/NUCLEO-WL55JC1/otaa/otaa_secure/README.md` "Hardware
protection mechanisms to test physically" szakaszába is, mivel közvetlen
hatással van az `otaa_secure` hardening-backlogjára.

## Már ismert / részben tesztelt (a gyári-firmware-es egységen)

- **RDP** (`FLASH_OPTR_RDP`, 3 szint) - Level 0 már megerősítve
  regiszterszinten,
  `Documentation/lab_notes/2026_09_19_stm32wl_rdp_register_confirmed.md`.
  Fizikai teszt: RDP1->0 downgrade voltage/clock glitchinggel (a
  `2026_07_22_DPA_DCA/brainstorm.md` szál).
- **Runtime SWD debug-port letiltás** - szoftveres, nem option byte;
  már megfigyelve és megkerülve `--connect-under-reset`-tel. Fizikai
  teszt: logic analyzer időzítés-capture, pontosan mikor a boot
  folyamatban kapcsol le (ld. `2026_09_28_stm32wl_logic_analyzer_plan/brainstorm.md`
  SWD-csatornái).

## Megvan a chipen, ezen a projekten még nincs tesztelve

- **PCROP** (`FLASH_PCROP1ASR`/`FLASH_PCROP1AER`) - egy flash-terület
  csak-végrehajtható, se SWD-n, se a CPU által adatként nem olvasható,
  RDP-szinttől függetlenül. Ide kerülhetne kulcsanyag, ha `otaa_secure`
  valaha flash-ben tárolna ilyet. Fizikai teszt: megpróbálni olvasni egy
  PCROP-védett régiót SWD-n RDP Level 0 mellett is - sikertelennek
  kellene lennie, ahol egy nem-PCROP régiónál nem az.
- **`FLASH_OPTR_ESE`** ("Security Enable") - külön option bit, ami
  aktiválja a **GTZC**-t (`GTZC_TZSC`/`GTZC_TZIC`, megerősítve a
  fejlécben). A WL55-ön nincs valódi ARM TrustZone mag (Cortex-M4+M0+,
  nem M33), de ST ezt az IP-blokkot arra használja, hogy elválassza, mely
  RAM/flash/periféria-terület érhető el az M4 (alkalmazás) vs. M0+
  (rádió-stack) mag felől. **Közvetlenül releváns, hogy egy
  kompromittált M4-alkalmazás elérne-e az M0+ oldali kriptó-anyaghoz** -
  a pontos szemantika egyelőre nem megerősített, RM0453 kell hozzá.
- **`nBOOT0`/`nBOOT1` option bitek** - a system-bootloader vs. user-flash
  boot-ot szabályozzák. Fizikai teszt: ROM system bootloader
  rákényszerítése (BOOT0 láb + reset), és annak ellenőrzése, hogy az
  USART/I2C/SPI bootloader-protokollja megkerüli-e az RDP-t - más STM32-
  családokon ismert hibaosztály, itt nem ellenőrzött.
- **TAMP periféria** (`TAMP_BASE` megerősítve) - fizikai
  beavatkozás-érzékelő lábak, amik backup-regisztereket törölhetnek.
  Relevánssá válik, ha később bármilyen doboz/tamper-történet bekerül.
  Fizikai teszt: tamper-láb triggerelése, tényleg törlődik-e a védett
  tartalom.
- **PKA / AES / RNG hardveres perifériák** (mind megerősítve) - a fentebb
  nyitva hagyott Ed25519-ellenőrzési hiányossághoz releváns: a PKA elvben
  natívan gyorsíthatná az aláírás-ellenőrzést, de hogy pontosan milyen
  görbéket támogat (Ed25519 kifejezetten, vagy csak NIST-görbéket), az
  RM0453-ból dől el, nem a fejlécből. Az AES a természetes oldalcsatorna-
  (SPA/DPA) célpont a már tervezett söntellenállásos méréshez
  (DPA_DCA brainstorm).

## Ellenőrizve, hogy NINCS a chipen - ne feltételezzük

Nincs `FIREWALL_BASE` (a Firewall periféria, ami más STM32-családokon
megvan, a WL55-ön nincs jelen), nincs valódi ARM TrustZone (nincs
Cortex-M33 mag).
