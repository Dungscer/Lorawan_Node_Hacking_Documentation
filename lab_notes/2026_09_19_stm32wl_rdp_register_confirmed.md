# STM32WL - RDP regiszter-szintű megerősítése (2026-09-19)

## OpenOCD DPIDR probléma megoldva

A 2026-09-09-i jegyzetben nyitva hagyott kérdés ("az elérhető
`stm32l4x.cfg` más DPIDR-t vár, mint amit ez a chip ad") megoldódott: a
telepített OpenOCD-ben (és az Arduino-hoz telepített ESP32/Heltec OpenOCD
csomagokban is) már **van dedikált `target/stm32wlx.cfg`** - ezzel a
csatlakozás (reset alatt, `connect_assert_srst`) tökéletesen működött,
`[stm32wlx.cpu0] Cortex-M4 r0p1 processor detected`, helyesen megállt a
`Reset_Handler`-en (`pc: 0x0800014c`, `msp: 0x20000aa0` - pontosan egyezik
a 2026-09-09-i dumpból kiolvasott vektortábla-értékekkel).

## RDP szint - most már regiszter-szinten is megerősítve

```
mdw 0x58004020 1
0x58004020: 3ffff0aa
```

A `FLASH_OPTR` regiszter alsó bájtja (`0xAA`) az RDP mező - ez **RDP
Level 0**-t (nincs olvasásvédelem) jelent. Ez véglegesen megerősíti a
2026-09-09-i, a sikeres teljes flash-dumpból LEVEZETETT feltételezést:
nem csak valószínű, hanem **regiszter-szinten bizonyított**, hogy ezen az
eszközön nincs RDP bekapcsolva.

## UART AT-parancsos verzió-lekérdezés - nem sikerült (hardver-limitáció)

A `SubGHz_Phy_AT_Slave` alkalmazás (ld. előző jegyzet) UART-on 9600 8N1
konfiguráción várja az AT-parancsokat. Megpróbáltam `/dev/ttyACM0`-n
(9600 8N1, `AT?`, `ATZ`), illetve reset-toggle után passzívan is
figyeltem - **semmilyen válasz/kimenet nem jött**.

**Ok:** `lsusb` csak **egy** ST-eszközt mutat (a külső ST-Link/V3
debug-probe, `0483:374e`) - a NUCLEO-WL55JC1 board saját, beépített
USB-portja **nincs csatlakoztatva** ehhez a géphez. A jelenlegi
`/dev/ttyACM0` a külső probe saját vezérlőcsatornája, nem a target UART-ja
- a valódi Nucleo-boardok saját onboard ST-Link-je adná a UART-hidat, de
azt csak a board saját USB-portján keresztül (külön kábellel) lehetne
elérni, amit most nem kötöttem be.

**Pontosítás:** kiderült, hogy csak **egy** USB-port van a boardon (nincs
külön "külső probe" + "board saját portja" - ez feltehetően maga a NUCLEO
board saját, beépített ST-Link-je, `lsusb -v` szerint van rajta "ST-Link
mass storage" interfész is, ami tipikusan Nucleo-specifikus drag-and-drop
flash funkció, nem külön álló probe-oké). Tehát ugyanaz az egyetlen
kapcsolat adná a VCP-t is, ha be lenne hidalva.

**Négy elterjedt sebességen** (9600, 19200, 57600, 115200 - 8N1, reset
után) sem jött semmilyen kimenet. Ez arra utal, hogy a firmware **nem az
ST-Link VCP-hez alapértelmezetten hidalt USART-ot** használja az
AT-interfészhez (AliExpress-es klón/variáns esetén ez nem meglepő
eltérés a stock Nucleo huzalozástól). Ennek továbbviteléhez fizikailag,
logikai analizátorral kellene megkeresni, melyik lábon van tényleges
UART-forgalom - ez távolról nem megy, helyben kellene elvégeznem, ha van rá
kapacitás.
