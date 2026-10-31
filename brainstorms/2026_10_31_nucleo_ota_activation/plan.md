# NUCLEO OTA image-aktiváció - tervezés (2026-10-31)

Az OTA-vevő a stagelt képet a külső W25Q-ra írja és aláírás + readback-SHA-256
alapján igazolja. Ami hiányzik: a stagelt kép tényleges futtatása. Ez a terv a
lehetőségeket és a konkrét számokat rögzíti; implementáció még nincs.

## A probléma
- A futó app a belső FLASH-ből (`0x08000000`) hajtódik végre.
- A stagelt új kép a W25Q-n van (SPI NOR, NEM memória-leképzett, nem futtatható
  helyben) - tehát be kell másolni a belső FLASH-be, majd reset.
- A futó appból nem lehet felülírni azt a FLASH-régiót, amiből épp fut.
- A STM32WL55 egybankos (nincs `DBANK`), tehát az ESP32-s bank-swap nem opció.

## Flash-elrendezés (mért)
- FLASH origin `0x08000000`, a linker 248K-t ad (`0x0803E000`-ig); a teljes chip
  256K (`0x08040000`-ig).
- App mérete: basic ~125 KB (`text 123984 + data 1208`), secure ~131 KB.
- Foglalt a felső tartományban: anti-rollback lap `0x0803D800` (2 KB), LoRaWAN/
  modem/SE NVM-kontextus `0x0803F000`-tól (`lora_app.c`).
- Staging: W25Q offset `0x00000000`, max 256 KB (`OTA_STAGING_FLASH_SIZE`).

## A) RAM-rezidens másoló (bootloader nélkül)
Egy RAM-ból futó rutin (linker-attribútummal RAM-ba helyezve / RAM2 `xrw`), amit a
readback-verify után, megszakítások tiltva hívunk: letörli az app-lapokat, a W25Q-ból
laponként beolvas a belső FLASH-be, majd `NVIC_SystemReset()`.
- Előny: nincs app-áthelyezés, nincs VTOR-állítás, kicsi; a meglévő `W25Q_Read` +
  HAL flash újrahasznosul.
- Hátrány: ha a másolás közben megszakad a tápellátás, a készülék **téglásodik**
  (nincs visszaállás) - a félkész app már felülírta a futót, a bootból nincs
  recovery.
- Mikor elég: PoC "attack -> patch" demóhoz, a téglásodási kockázat dokumentálva.

## B) Kis bootloader + app-áthelyezés
Minimál bootloader `0x08000000`-nál (~16 KB), az app áthelyezve `0x08004000`-ra
(VTOR-offset). Bootkor a bootloader egy "update pending" flaget néz (flash-lap vagy
RTC backup regiszter); ha be van állítva, W25Q -> app-régió másolás, flag törlése,
majd ugrás az appra. Normál boot: csak ugrás.
- Előny: biztonságos - tápkimaradás a másolás közben a következő bootkor
  újrapróbálható, a kép a W25Q-n megvan; ez a szokásos, robusztus megoldás.
- Hátrány: app-áthelyezés kell (linker: FLASH origin `0x08004000`, csökkentett
  hossz), VTOR + `SystemInit` offset, külön bootloader build/flash. Az app
  használható flashe a bootloader méretével csökken (16 KB-tal: ~230 KB app-régió
  `0x08004000..0x0803D800`, a jelenlegi ~125-131 KB képek beleférnek).

## Javaslat
A dolgozat biztonsági eredménye (aláírt OTA) már megvan; az aktiváció mérnöki
kiegészítés a teljes "attack -> patch" demóhoz. Ha a cél egy gyors, bemutatható
end-to-end kör, az A) elég (a téglásodási kockázat kimondva). Ha robusztus, "éles"
megoldás kell a dolgozatba, a B) a helyes - de ez a nagyobb munka (app-áthelyezés +
külön bootloader). Döntés a rendelkezésre álló idő függvénye.

## Igazolás (bármelyik út)
1. Push egy új, aláírt kép (más, jól felismerhető viselkedéssel, pl. eltérő
   uplink-periódus vagy LED/log) -> staged + verified.
2. Aktiváció kiváltása -> reset.
3. Boot után a node UART-ja/on-air viselkedése az ÚJ képet mutatja (nem a régit) ->
   az aktiváció igazolt hardveren.
4. Negatív eset: szándékosan rossz kép / megszakított másolás viselkedése (A-nál ez
   a téglásodás, B-nél a recovery) - dokumentálandó.
