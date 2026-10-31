# NUCLEO OTA bootloader - Fázis 1 (relokáció + csak-ugró átadás) (2026-10-31)

A robusztus aktiváció-út első fázisa: egy minimál bootloader `0x08000000`-nál, az
app áthelyezve `0x08008000`-ra. Ebben a fázisban a bootloader még nem másol semmit,
csak átadja a vezérlést az appnak - így a flash-tartalomra nulla a kockázat, és
külön igazolható a relokáció + VTOR-átadás, mielőtt a Fázis 2 flash-írást hoz.

## Mit csináltam
A bootloadert a bevált vendorolt startup + system + CMSIS-re építettem, külön 32 KB-os
FLASH-linkerrel; a `main_boot.c` beolvassa az app MSP-jét és reset-vektorát a
`0x08008000`-ról, `SCB->VTOR`-t az appra állítja, betölti az MSP-t és ugrik.
Az appot (basic) relokáltam: linker `FLASH ORIGIN = 0x08008000, LENGTH = 212K`, és
`SCB->VTOR = 0x08008000` a `main()` legelején. Mindkettőt külön flasheltem
(`st-flash --connect-under-reset`, app a `0x08008000`-ra, bootloader a
`0x08000000`-ra).

## Ami elsőre elromlott, és miért
Az első próbán a node bootolt, kiírta a bannerét (`LoRaWAN End Node LBM`), de nem
joinolt - a ChirpStack `last_seen` több percig nem frissült. A bootloader első
változata `__disable_irq()`-et hívott az átadás előtt, és nem engedélyezte újra;
így az app `PRIMASK=1`-gyel indult (miközben egy hardveres reset engedélyezett
megszakításokkal indít), és az első SysTick-alapú `HAL_Delay()`-nél örökre
beragadt - a banner még kiment, a join már nem. A `cpsid i` eltávolítása (az app a
reset-állapotnak megfelelően, engedélyezett megszakításokkal indul) megoldotta.

## Igazolás
A javítás után: reset, majd 70 s érintetlen ablak, és a ChirpStack `last_seen` 3 s
friss lett, **új DevAddr `016535b5`** - tehát a relokált app a bootloaderen át
bootol és joinol. A független jel itt a ChirpStack `last_seen` volt, mert a VCP
flash után újra megbízhatatlanul néma volt (gyakori ebben a setupban). A readback
is igazolta, hogy a bootloader `0x08000000`-on és az app `0x08008000`-on helyesen
van flashelve.

## Megjegyzés a biztonságról
A saját fejlesztői flashelés nem téglásíthatja a boardot: az ST-Link
`--connect-under-reset` a mag leállításával bármilyen app-állapot fölé újraír;
option byte-okhoz nem nyúltam. A tápkimaradás-biztos recovery (a lényegi cél) a
Fázis 2/3 flag + újrapróbálás logikájával valósul meg.
