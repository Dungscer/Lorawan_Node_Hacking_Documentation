# NUCLEO OTA bootloader - Fázis 2 (telepítő logika) (2026-10-31)

A bootloaderbe megírtam a tényleges telepítő utat: csupasz-regiszteres belső-flash
író (lap-törlés + 64-bites doubleword program), read-only W25Q-olvasó, a bemásolt
kép SHA-256-ellenőrzése, és az aktivációs flag-lap (`0x0803D000`) kezelése. A logika:
bootkor ha a flag `magic`-je be van állítva, a W25Q-ról az app-régióba másol, a
bemásolt flash-tartalmat hasheli és a flaghez méri; egyezésnél törli a flaget és
ugrik, eltérésnél (pl. tápkimaradás másolás közben) reset -> következő bootkor
újrapróbál (a W25Q-kép ép, ezért nem téglásodik). Az app-oldalon
`OTA_ACTIVATION_ENABLED` alatt (default 0, a standalone basic/secure változatlan) a
teljesen igazolt kép a flag-lapra írja `magic|len|sha256`-ot és `NVIC_SystemReset()`-et
hív. Minden fordul, a bootloader 3,4 KB.

## Hardveres próba és ami elakadt
A Fázis-2 bootloader + a relokált, aktivációs basic app (v1) felflashelve: v1
**joinolt** (ChirpStack `last_seen` 6 s) - tehát a bootloader a normál bootot nem
bontja el. Ezután egy apró (1080 B) relokált teszt-képet (v2) pusholtam v1-re az
aktiváció kiváltásához. A manifest kézbesült, de az első chunknál a kézbesítés
elakadt, és a `last_seen` a manifest óta nem frissült: **v1 a manifest után
elnémult**, és egy tiszta reset után sem joinolt újra. Ugyanez az OTA-kód a
standalone (nem relokált) basic appban végig hibátlanul ment (ebben a sessionben
250 B / 600 B / 4500 B transzfer, mind SUCCESS). A különbség tehát a relokáció
és/vagy az aktivációs build.

A gyökérokot nem sikerült bemérni: a VCP flash után megbízhatatlanul néma volt, az
SWD `DEBUGGER_ENABLED=0` miatt nem csatlakozott, és az RTL-SDR ebben a menetben nem
enumerálódott - így a node-oldali tracelés hiányzott, a ChirpStack `last_seen` volt
az egyetlen független jel. A boardot visszaállítottam ismert-jó standalone képre
(joinol), tehát a hardver és a LoRaWAN-NVM ép, a jelenség v1-specifikus.

## Amit biztosan tudok
- A bootloader belső-flash írója + W25Q-olvasója + SHA-verify-je fordul és
  önmagában konzisztens; a Fázis-1 (relokáció + ugrás) külön igazolt.
- A standalone basic/secure OTA útja ép (külön igazolt ugyanebben a menetben).
- A relokált+aktivációs app (v1) kezdetben joinol, de az OTA-manifest után
  elakad - ez a bemérendő pont.
- A saját flashelés nem téglásítja a boardot: az ST-Link under-reset bármilyen
  app-állapot fölé újraír.
