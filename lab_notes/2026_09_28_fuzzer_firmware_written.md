# 2026-09-28 - fuzzer.ino megírva (teszteletlen, nincs csatlakoztatva eszköz)

## Miben más, mint a chirpstack_js_engine

Ez egyáltalán nem LoRaWAN node - nincs join, nincsenek OTAA-kulcsok. A meglévő
tervezési README saját scope-megjegyzése szerint a gateway rádió-koncentrátora
a beállított frekvencián/SF-en bármilyen LoRa PHY jelet vesz, regisztrációtól
függetlenül; a LoRaWAN-szintű validáció (MIC, FCnt) csak később, a network
serveren történik. Ezért a firmware közvetlenül a SX1262 chippel beszél nyers
LoRa PHY módban (a RadioLib sima `SX1262`/`Module` API-ja, nem a `LoRaWANNode`).

## Mit csináltam

- Végigolvastam a meglévő `fuzzer/README.md` tervezési jegyzetet, beleértve a
  kifejezett kétfázisú tervét: 1. fázis (a tényleges packet-forwarder forrás
  offline fuzzolása, rádió nélkül, aminek elsőként kellene jönnie) és 2. fázis
  (RF-reprodukció, aminek csak az 1. fázis találata után kéne megtörténnie).
  Az 1. fázis még nincs kész. A firmware-t ennek ellenére megírtam, ugyanúgy,
  ahogy a `chirpstack_js_engine.ino`-t a ChirpStack codec/integráció igazolása
  előtt - ezt expliciten dokumentáltam a README státuszsorában és egy számozott
  Testing TODO pontban ("1. fázis újranézése"), ahelyett hogy csendben
  átugranám a terv saját sorrendjét.
- Megnéztem a RadioLib `SX126x.h`/`SX1262.h`-ban a nyers-PHY API-felületet:
  `setFrequency`, `setBandwidth`, `setSpreadingFactor`, `setCodingRate`,
  `setOutputPower` (a `PhysicalLayer::setOutputPower(int8_t)`-n át),
  `setPreambleLength`, `setCRC(len, ...)` (len=0 kikapcsolja a CRC-t, a saját
  doc-kommentje szerint), `implicitHeader(len)`/`explicitHeader()`, és
  `transmit(data, len)`. Megerősítettem, hogy a LoRaWAN-publikus sync word
  konstans (`RADIOLIB_SX126X_SYNC_WORD_PUBLIC`, 0x34) létezik, és kell is, hogy
  a gateway (LoRaWAN-forgalomra hangolt) koncentrátora ténylegesen demodulálja
  ezeket a nem-LoRaWAN kereteket.
- **Megnéztem, elérhető-e valódi header-CRC-hamisítás vagy
  SF/CR-vs-header-eltérésű keret** (mindkettő az eredeti README céllistájában
  volt: "rossz header CRC, inkonzisztens coding rate/SF"). A
  `writeRegister()`/`readRegister()` létezik a `SX126x`-en, de `protected`
  láthatóságú - sketch-kódból subclass nélkül nem hívható. Úgy döntöttem, ezt a
  két variánst nem valósítom meg, ahelyett hogy meghamisítanám vagy csendben
  elejteném a célt - a README-ben explicit, megerősített hiányosságként
  dokumentáltam ("Explicit gap, not a bug") a pontos okkal, hogy egy jövőbeli
  nekifutásnak ne kelljen ezt újra levezetnie.
- Megírtam a `fuzzer.ino`-t: 6 keret-variáns (`BASELINE`, `CRC_DISABLED`,
  `HEADER_LEN_LIE`, `MAX_LENGTH`, `MIN_LENGTH`, `EXTREME_PREAMBLE`), teljesen a
  fenti publikus API-ból. Ráraktam ugyanazt a WiFi AP + web dashboard mintát,
  mint a `chirpstack_js_engine.ino`-nál (saját AP `LoRaWAN-Fuzzer-<chip-id>`,
  `192.168.4.1`, csak eszközoldali eredmények, heurisztikus címkézés nélkül) -
  ez a fuzzer saját tervezési README-jében is "nice-to-have"-ként szerepelt,
  úgyhogy újrahasználtam a dizájnt anélkül, hogy újra feltettem volna a
  WiFi-mód/értékelési-kör kérdéseket, amiket a `chirpstack_js_engine.ino`-nál
  már korábban tisztáztam ugyanaznap.
- A ChirpStack node-dal ellentétben itt a config-változások
  (frequency/BW/SF/CR/power/preamble/interval/variáns) azonnal, újraindítás
  nélkül érvényesülnek - nincs join/session-állapot, amit érvényteleníteni kéne.
- Felvettem egy szabályozási megjegyzést (firmware header-komment + README): a
  nyers PHY mód teljesen megkerüli a LoRaWAN saját duty-cycle-könyvelését, így
  a TX-intervallum/teljesítmény EU868-határokon belül tartása az üzemeltető
  felelőssége. Az alapértékeket konzervatívan hagytam (10s intervallum,
  10 dBm), de nincsenek kikényszerítve.
- Ezúttal gondosan ellenőriztem a deklarációs sorrendet (a
  `PayloadVariant`/állapot-globálisok-a-`setup()`-után hiba a ChirpStack-fájlban
  ott elő is jött és javítottam) - a `RadioConfig`/`FrameVariant`/`ResultEntry`-t
  és minden globálist eleve a `setup()`/`loop()` elé írtam, majd grepeltem a
  fájlt a sorrend megerősítésére, mielőtt késznek vettem.

## Eredmény / ami NINCS igazolva

- **Semmi nem futott valódi hardveren vagy valódi gateway ellen.** Se
  `radio.begin()`, se transmit, se megerősítés arról, hogy a gateway packet
  forwardere egyáltalán látja-e a `BASELINE` keretet, nemhogy arról, hogyan
  kezeli a malformált variánsokat. Ez "implementálva", nem "igazolva".
- Az eredeti terv 1. fázisa (a pontos packet-forwarder forrás/verzió
  azonosítása, offline fuzzolása) továbbra is teljesen nyitott, és elvileg
  elsőként kellett volna jönnie - ezen a firmware nem változtat, ez egy
  általános célú eszköz, nem egy bug-reprodukáló.
