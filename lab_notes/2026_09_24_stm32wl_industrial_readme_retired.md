# STM32WL Industrial - a Hacking/README lezárva, tartalma jegyzetekbe/ROADMAP-ba mentve (2026-09-24)

**Cél:** a `Hacking/STM32WL_Industrial/` mappa (README + a `dumps/`
almappa) törlésre kerül - a dump már átkerült a
`Firmwares/Nodes/NUCLEO-WL55JC1/original_firmware/`-ba (ld. lentebb), a
README tartalma pedig vagy már megvan a korábbi
(`2026_09_09_stm32wl_industrial_first_look.md`,
`2026_09_19_stm32wl_firmware_identification.md`,
`2026_09_19_stm32wl_rdp_register_confirmed.md`) jegyzetekben/`ROADMAP.md`-ban,
vagy itt kerül rögzítésre, mielőtt a README eltűnik.

## Az egyetlen adat, ami csak a README-ben volt meg sehol máshol

A flash-dump SHA-256 hash-e:

```
Firmwares/Nodes/NUCLEO-WL55JC1/original_firmware/flash_dump_20260909_085958.bin
sha256: d6365d917e4a62e29624c0933ab62a5b9019bde8930bd944828e1ab655cf5f8c
```

## Minden más, amit a README tartalmazott - hol van meg máshol

- Eszközazonosítás (ST-Link/V3 VID:PID `0483:374e`, chipid `0x497`,
  256 KiB flash/64 KiB SRAM), az SWD futás-közbeni letiltás megfigyelése,
  a reset-alatti-csatlakozás megkerülés: `2026_09_09_stm32wl_industrial_first_look.md`.
- Firmware-azonosítás (ST hivatalos `STM32CubeWL`/`SubGHz_Phy_AT_Slave`
  AT-parancs keretrendszer, `AT+VL` kódegyezés), célzott kulcskeresés
  (hamis pozitív, ARM Thumb `b .` mintázat): `2026_09_19_stm32wl_firmware_identification.md`.
- RDP regiszter-szintű megerősítés (`FLASH_OPTR = 0x3FFFF0AA`, RDP Level 0):
  `2026_09_19_stm32wl_rdp_register_confirmed.md`.
- **2026-09-22-i pontosítás** (a debug-port bypass/RDP
  Level 0 csak a gyári demo-firmware-en igazolt, éles ipari firmware-nél
  valószínűleg JTAG/SWD-védelem is be lenne kapcsolva; ez pontosan ezért
  indokolja a NUCLEO saját base/otaa_secure/otaa_vulnerable firmware-tier-jeit)
  és a **2026-09-22-i NUCLEO-döntés** (gyári firmware referenciaként
  megőrizve a dump alapján, a fizikai egység újraflashelhető, de ez után
  az élő UART/AT-parancs viselkedés nem reprodukálható tovább ezen az
  egységen) mindkettő szó szerint benne van a gyökér `ROADMAP.md`
  "STM32WL 'ipari' eszköz" szakaszában.
- Nyitott tételek (pontos CubeWL-verzió azonosítatlan, UART-hozzáférés a
  board saját USB-portján még nem próbálva) szintén a gyökér `ROADMAP.md`
  "STM32WL" szakaszában vannak követve.

## Miért törlődik a mappa

A `Hacking/STM32WL_Industrial/README.md` a 2026-09-22-i és 2026-09-24-i
munka során fokozatosan olyan tartalommal bővült (figyelmeztetések,
döntés-lezárások), ami valójában vagy a `ROADMAP.md`-be, vagy egy dátumozott
jegyzetbe való - nem a README-be, ami csak összefoglaló/pointer kellene
legyen. A `Hacking/`-ban innentől nincs önálló STM32WL-mappa; a téma a
`Firmwares/Nodes/NUCLEO-WL55JC1/` (implementáció) és a `ROADMAP.md`
"STM32WL" szakasza (állapot/nyitott kérdések) alatt fut tovább.
