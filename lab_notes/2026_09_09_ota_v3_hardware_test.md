# OTA protokoll v3 - hardveres teszt jegyzet (2026-09-09)

## Környezet / módszertan

- Eszköz: Heltec WiFi LoRa 32 V2 (ESP32-D0WDQ6 rev v1.1, MAC 88:13:bf:fd:36:e4),
  USB-n `/dev/ttyUSB0`.
- ChirpStack: tenant "ChirpStack" (`75445599-...`), application "Application_01"
  (`5f34788a-...`), device-profile "OTAA" (`c3ee3bd1-...`) - ezek **más**
  azonosítók, mint az `Information.md`-ben rögzített régi 5 node/alkalmazás,
  tehát ez tényleg egy másik (az új, ROADMAP-ban is említett) ChirpStack
  szerver, nem ugyanaz, ahol az 5 demo-eszköz van regisztrálva.
- Ideiglenes teszt-eszközt regisztráltam gRPC-n (`DeviceService.Create` +
  `CreateKeys`) friss, véletlen DevEUI/JoinEUI/AppKey-vel - **nem** a meglévő
  V2 Node1/2 credentialokat használtam, hogy ne keveredjen a demo-eszközök
  állapotával.
- OTA aláíró kulcspárt is frissen generáltam (`keygen.py`) csak erre a
  tesztre; a `otaa_secure.ino`-ba a valós DevEUI/AppKey/OTA_PUBKEY-t csak a
  teszt idejére írtam bele, utána `git checkout`-tal visszaállítottam
  placeholder nullákra (valós kulcs nem kerülhet a repóba).
- A teszt payload nem valódi firmware volt, hanem 300 bájtos szintetikus
  adat (a valódi ~380 KB-os otaa_secure image LoRaWAN-on 50 bájtos chunkkal
  kb. **21 óráig** tartott volna - ez a `README.md`-ben dokumentált "Slow by
  design" korlát direkt következménye, ~10s/chunk minimum a TX_INTERVAL
  miatt).

## Megfigyelés 1 - a fordítási lánc önmagában nem működött

A `otaa_secure.ino` a Library Manager-es "Heltec ESP32 Dev-Boards" 2.1.6
csomaggal nem fordult a jelenleg telepített `Heltec-esp32:esp32` 3.3.8
core-ral:

```
sx1276.c:1289:5: error: implicit declaration of function 'SpiInOut'
```

A hiba a könyvtár `src/driver/sx1276.c`-jében van, amit **a sketch nem is
használ** (a LMIC közvetlenül vezérli a rádiót, nem a Heltec-driveren
keresztül) - csak azért fordult volna bele, mert az Arduino build a teljes
library `src/`-jét lefordítja, függetlenül attól, mit használ ténylegesen a
sketch. A `SpiInOut` egy régebbi ESP32 core belső SPI API-ja, amit az IDF
v5.5 (a jelenlegi core alapja) már nem tartalmaz - verzió-inkompatibilitás
a könyvtár és a core között.

**Megoldás (dokumentálva `otaa_secure/README.md`-ben is):** csak a ténylegesen
használt `HT_SSD1306Wire.h` + `HT_Display.h/.cpp` + `HT_DisplayFonts.h`
fájlokat másoltam a sketch mellé, a teljes könyvtárat ideiglenesen kivettem
a `~/Arduino/libraries/`-ből a fordítás idejére, utána visszatettem.

## Megfigyelés 2 - a protokoll v3 érdemi része hibátlanul működött

Egy tiszta, megszakítás nélküli futtatásban (lásd a soros log kivonatát):

```
[OTA] Manifest verified - version=1, total_bytes=300
[OTA] 50/300 bytes  seq=0
[OTA] 100/300 bytes seq=1
[OTA] 150/300 bytes seq=2
[OTA] 200/300 bytes seq=3
[OTA] 250/300 bytes seq=4
[OTA] 300/300 bytes seq=5
[OTA] SHA-256 verified - finalising...
E (170061) boot_comm: mismatch chip ID, expected 0, found 22352
[OTA] Update.end() FAILED: Could Not Activate The Firmware
```

Ez pontosan a várt viselkedés:
- a manifest aláírás- és anti-rollback ellenőrzése átment,
- mind a 6 data frame session_id-je és CRC8-a helyes volt (egyik sem lett
  elutasítva "stale/foreign transfer" vagy CRC-mismatch miatt),
- a futó SHA-256 hash a teljes 300 bájt felett pontosan egyezett a
  manifestben aláírt hash-sel,
- az `Update.end()` **helyesen** utasította el a végén - mivel a teszt
  payload nem volt valódi, lefordított ESP32 image (hiányzott az érvényes
  image header/chip-ID mező), ez az ESP-IDF saját bootloader-szintű
  image-validációja, nem a kódom hibája. Az eszköz ezután zökkenőmentesen
  visszaállt normál üzemre (folytatta az uplinkeket).

## Megfigyelés 3 - a session-védelem véletlenül "éles" tesztet is kapott

Egy korábbi, megszakadt futtatás után (ahol a host már elküldött egy
manifestet + néhány chunkot, de a transzfer nem fejeződött be, mert
közben többször resetáltam az eszközt esptool-lal a hibakereséshez) egy
**új** manifestet küldtem (új, véletlen `session_id`-vel), de a
`Update.begin()` erre a második manifestre hibával tért vissza:

```
[OTA] Manifest verified - version=1, total_bytes=300
[OTA] Update.begin() FAILED: No Error
```

Mivel a `processManifestFrame()` ezen a hibaágon **nem** frissíti
`ota.sessionId`-t és nem hívja meg `Update.abort()`-ot (ez egy létező,
a v3-tól független apró robusztussági rés - a hibás `Update.begin()` után
a device state befagyva marad a régi session-nél), a device a régi,
korábbi session_id-t tartotta aktívnak. Az utána érkező (már az új,
sikertelen manifesthez tartozó) 6 data frame mindegyike helyesen lett
elutasítva:

```
[OTA] Data frame session=01E2395C != active=AC3F3CEC, ignoring (stale/foreign transfer)
```

Ez **pozitív megerősítés** arra, hogy a session-scoping ténylegesen működik
- akaratlanul pont azt a forgatókönyvet reprodukáltam (megszakadt transzfer
utáni "idegen" chunkok), amit a védelem kivédeni hivatott. Ugyanakkor
felszínre hozott egy apró, ezzel nem összefüggő hibalehetőséget
(`Update.begin()` hiba esetén hiányzó `ota = OtaState()` reset +
`Update.abort()` hívás) - ezt **nem** javítottam most, mert nem ez volt a
feladat, csak jegyzem későbbre.

## Megfigyelés 4 - hamis pozitív a ChirpStack "kézbesítve" jelzésénél

Egy futtatásban a host tool `SUCCESS, 6/6 chunks`-ot jelzett, miközben a
device soros logja szerint csak 4 chunk (200/300 bájt) érkezett meg
ténylegesen - a queue ennek ellenére `depth=0`-ra állt, amit a
`_wait_for_queue_empty()` "kézbesítve"-ként értelmez. Ez a
`Tools/OTA_Update/README.md`-ben már dokumentált "Unconfirmed downlinks
only" korlát élesben reprodukálva: a ChirpStack a queue kiürülését (a
downlink RF-en való *kiküldését*) jelzi, nem a device tényleges *vételét*.

## Megfigyelés 5 - a RAK7268 gateway ténylegesen már üzemel

A ChirpStack `GatewayService.Get` hívás szerint a `rak7268v2`
(`ac1f09fffe1f4131`) gateway `last_seen_at` értéke a teszt idején
gyakorlatilag valós idejű volt (percek, nem napok). Ez ellentmond a
`ROADMAP.md` 2026-07-27-i "még nincs hozzákötve" állapotának - vagy azóta
megoldódott és elfelejtették frissíteni a ROADMAP-ot, vagy valaki más rakta
össze. Frissítve a `ROADMAP.md`-ben.
