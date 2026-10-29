# ChirpStack JS-codec attack chain - hardveren végigvíve (2026-10-04)

## Terep (recon, gRPC + SSH)
- Hozzáférés: `chirpstack_api` + `grpc` Python az API-tokennel (Information.md),
  és paramiko-SSH a `.166`-ra. `.166` = **Windows** docker-compose stack.
- ChirpStack **4.19.0** (`docker exec ... chirpstack --version`). Konténerek:
  chirpstack:4, chirpstack-rest-api:4 (:8090), **mosquitto:2 (:1883)**, postgres,
  redis, graylog, opensearch, mongodb, nginx-grpc-tls.
- Egyetlen app (`Application_01`), egyetlen integráció: **`MQTT_GLOBAL`** (a
  dekódolt uplink ide publikálódik -> ez a lehetséges injection-nyelő).
- Eredeti device-profile „OTAA": codec **runtime=NONE** (a tárolt script az alap-
  sablon, nem is futott). Csak a NUCLEO volt regisztrálva; a V4 **nem**.

## Célpont felállítása
- Új device-profile **`ATTACK-jscodec`** (az OTAA-profil klónja, de JS-codec
  runtime-mal), hogy a NUCLEO profilját ne bántsam. A codec egy reális „hibás
  vendor-codec" két klasszikus buggal:
  ```js
  function decodeUplink(input) {
    var b = input.bytes;
    if (b.length > 0 && b[0] === 0xFF) { while (true) {} }   // (1) payload-vezérelt végtelen ciklus
    var raw = "";
    for (var i = 0; i < b.length; i++) raw += String.fromCharCode(b[i]); // (2) nyers, sanitizálatlan átvezetés
    return { data: { fPort: input.fPort, len: b.length, raw: raw } };
  }
  ```
- V4 node (DevEUI: ld. `Information.md`, gitignore-olt) regisztrálva `Application_01`-be ezzel a
  profillal; AppKey a `nwk_key` mezőben (LoRaWAN 1.0.x konvenció).

## Firmware - sosem fordult le (javítva)
A `chirpstack_js_engine.ino` a RadioLib egy **megszűnt konstansára** hivatkozott:
`RADIOLIB_LORAWAN_MAX_PAYLOAD_SIZE` (nincs a 7.6.0-ban). Ez **compile error** volt
(a korábbi „exit 0" félrevezetett: `arduino-cli ... | tail` a tail exitjét adta).
Javítás: `RADIOLIB_LORAWAN_MAX_DOWNLINK_SIZE` (=250, bármely payloadnál nagyobb
puffer), 4 helyen. Ezután tisztán fordul és flashel. (A creds-et flasheléshez
helyileg beégettem, majd **visszaállítottam nullára** - commitba nem kerül.)

## A lánc - hardveren igazolt
Megfigyelés: Graylog (ChirpStack-logok GELF-en) + MQTT-sub a `.166:1883`
mosquittóra. Variáns-váltás **ChirpStack-downlinkkel** a 99-es porton (a firmware
`variantSelectPort`-ja) - WiFi nélkül.

1. **Delivery + codec-exec + MQTT (ALL_ZERO kontroll):** a V4 **joinolt**
   (DevAddr `0114f46d`), az uplink megérkezett, a **codec lefutott** a támadó
   bájtjain, és a dekódolt objektum **kiment MQTT-re**:
   `object = {"fPort":2,"len":8,"raw":"\u0000…\u0000"}`.

2. **Injection -> MQTT-nyelő (SIKERES a nyelő határáig):** INJECTION payload
   `'";</script><img src=x>${7*7}--` -> a codec **sanitizálatlanul** átvezeti, és
   az MQTT-n **bájtra pontosan** megjelenik:
   `object.raw = "'\";</script><img src=x>${7*7}--"`.
   Vagyis a támadó-vezérelt SQL/XSS/template-marker **átér az integrációra**.

3. **Codec-DoS (MITIGÁLVA a ChirpStack 4.19.0-ban):** LEN_LIE (`bytes[0]=0xFF`)
   -> a codec `while(true)` ágába fut. Hatás: az érintett keretekre **`object=null`**,
   **de a NS végig válaszképes** - a 0xFF-keretek up-eventjei normál ~12s ütemben
   folytatódtak (fCnt 25, 26, …), nincs crash, és Graylog **0** codec/timeout/panic
   hibát logolt. Tehát a ChirpStack **korlátozza a codec-futásidőt** (issue #1
   gyakorlatilag javítva) -> a codecen át **nem DoS-olható** a hálózati szerver.
   (Teszt után azonnal visszaállítva benign `[0]`-ra; fCnt 29–30 újra ép objektum.)

## Mit jelent ez - és a határ
- **Teljes, hardveren igazolt lánc** a **támadó payloadtól az MQTT-integrációig**:
  egy joinolt (vagy kompromittált) eszköz a payload-bájtjaival **tetszőleges,
  sanitizálatlan tartalmat juttathat a dekódolt objektumba, ami kimegy az MQTT-re**.
- **A lánc utolsó láncszeme (tényleges kihasználás)** egy olyan **downstream
  MQTT-fogyasztót** igényel, amely a `object.raw`-t sanitizálatlanul használja
  (DB-insert -> SQLi, dashboard-render -> stored XSS, template -> RCE). A testbeden
  **ilyen fogyasztó nincs konfigurálva** (a `MQTT_GLOBAL` csak publikál; az
  `Tools/OTA_Update` a gRPC-API-t használja, nem az MQTT-uplinkeket). Tehát a lánc
  **a nyelő határáig teljes és demonstrált**, de weaponizált end-to-end exploit
  (konkrét áldozat-fogyasztóval) itt **nincs** igazolva.
- A **DoS-ág lezárva negatívan:** a 4.19.0 bounded codec-runtime miatt nem jó
  támadási vektor.

## Állapot / takarítás
- A `ATTACK-jscodec` profil + a V4 regisztráció + a sebezhető codec **bent maradt**
  a testbed ChirpStackjében (a tézis-demóhoz). Visszavonható: device törlése +
  profil törlése + (ha kell) a NUCLEO-profil érintetlen.
- A V4 most `chirpstack_js_engine`-t futtat, benign `[0]` variánson, 15s-enként
  uplinkel (joinolva). Leállítás = kihúzni.
- Kulcsok a firmware-forrásban visszaállítva nullára (nincs commitolt titok).
