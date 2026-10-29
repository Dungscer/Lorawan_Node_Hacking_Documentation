# 2026-09-28 - chirpstack_js_engine.ino megírva (teszteletlen, nincs csatlakoztatva eszköz)

## Mit csináltam

- Elolvastam a meglévő tervezési README-t és a testvér V4 `base_node.ino`-t
  (az egyetlen nem üres V4 vázlat) implementációs referenciának - ugyanazok a
  board-pinek, RadioLib `LoRaWANNode` + SSD1306 OLED váz, `Preferences` a
  DevNonce megőrzéséhez.
- A telepített RadioLib forrásban
  (`~/Arduino/libraries/RadioLib/src/protocols/LoRaWAN/LoRaWAN.h`) közvetlenül
  megnéztem a pontos `sendReceive()` túlterhelést, ami visszaadja a downlink
  payloadot + a `LoRaWANEvent_t`-t (a downlink port olvasásához), valamint a
  `getMaxPayloadLen()` / `RADIOLIB_LORAWAN_MAX_PAYLOAD_SIZE` (242) értéket,
  mert a fuzzernek a datarate szerinti valós max payload-méretet kell
  ismernie, nem egy beégetett konstanst.
- Megírtam a `chirpstack_js_engine.ino`-t: ugyanúgy OTAA-val joinol, mint a
  `base_node`, majd minden uplink-intervallumban az 5 payload-variáns
  (`ALL_ZERO`, `MIN`, `MAX_LEN`, `LEN_LIE`, `INJECTION` - mindegyik célját ld.
  a mappa README-jében) egyikét küldi valódi szenzorérték helyett. A variáns
  NVS-ben perzisztált, és Serialról vagy egy külön porton (99) érkező
  downlinkkel is állítható (ez a port a variánsválasztó parancsoknak van
  fenntartva), így iteráláshoz nem kell újraflashelni - ez nyitott
  követelmény volt az eredeti tervezési README-ben.
- Frissítettem a mappa `README.md`-jét: a státuszsor "not started"-ról
  "firmware megírva, hardveren még nem futott"-ra változott, kitöltöttem a
  "What needs to be implemented" 2. pontját azzal, amit a firmware ténylegesen
  megvalósít, és felvettem egy új "Testing TODO" szakaszt a konkrét
  lépésekkel, amik a "verified" státuszhoz kellenek (fordítási ellenőrzés,
  valós kulcsok, uplink/downlink-variánsválasztás igazolása a levegőn,
  codec-viselkedés megfigyelése variánsonként, majd státuszemelés).
- Frissítettem a `Firmwares/Nodes/README.md` implementációs-státusz tábláját:
  a `chirpstack_js_engine` / V4 cella most "megírva (2026-09-28), hardveren
  NEM tesztelve", a korábbi "tervezési dokumentum" helyett.

## Megkísérelt, de nem használható: fordítási ellenőrzés

Lefuttattam az `arduino-cli board listall`-t, hogy legalább a szintaktikai
hibákat elkapjam, mielőtt késznek mondom. Eredmény: a telepített
`Heltec-esp32:esp32` core-ban (3.3.8) csak V2/V3 Heltec board van regisztrálva
- V4 (ESP32-S3) FQBN lokálisan nem létezik. Ugyanennek a boardnak a
`base_node.ino`-ja is láthatóan ugyanebbe ütközik (nincs korábbi lab note
sikeres V4 fordításról). A V3 FQBN-nel mint helyettesítővel nem fordítottam le
- eltérő pinek és chip (ESP32 vs ESP32-S3), így egy "pass" hamis jelzés lenne,
egy "fail" pedig félrevezető. Zöld fordítás meghamisítása helyett nyitott
TODO-ként hagytam.

## Eredmény / ami NINCS igazolva

- **Semmi nem futott valódi hardveren.** Se join, se uplink, se downlink, se
  codec-megfigyelés. Ez "implementálva", nem "igazolva" - ezt a
  megkülönböztetést tudatosan megtartottam, nem mostam össze "kész"-re.
- A tényleges kutatási kérdés, amiért ez a firmware létezik - sérülékeny-e a
  testbed ChirpStack device-profile codec-szkriptje DoS-ra/erőforrás-
  kimerítésre, vagy továbbít-e tisztítatlan dekódolt kimenetet egy
  integrációnak - továbbra is teljesen nyitott. Ráadásul még azt is igazolni
  kell, hogy egyáltalán létezik-e a testbeden egyedi codec-szkriptes device
  profile (nyitott pont az eredeti tervezési README óta).

## Kiegészítés (aznap): WiFi audit dashboard hozzáadva

Az eredmények jelenleg semmilyen más módon nem láthatók (nincs hardver
csatlakoztatva, nincs más kimeneti csatorna), ezért a célom, hogy az eszköz
önálló terepi audit-eszközként működjön - böngészőből elérhető legyen pusztán
a bedugásával, és ugyanarról az oldalról be lehessen állítani a teszt-kulcsokat.

Implementáció előtt két elágazásban döntöttem, amit egyébként csak tippelni
lehetett volna:
- **WiFi mód** - saját access point vs. meglévő hálózatra csatlakozás. A
  **saját AP**-t választottam (`LoRaWAN-Audit-<chip-id>`, `192.168.4.1`) - nem
  kell lab-WiFi-kulcs, és illik a "dugd be és érd el" elvhez.
- **Értékelési kör** - az eszköz csak azt tudja jelenteni, mit küldött, és
  hogy jött-e vissza downlink és mikor; a ChirpStack saját codec-futását/
  viselkedését közvetlenül nem látja. A **csak eszközoldali adat, heurisztikus
  "gyanús" címkézés nélkül** opciót választottam - nyers mért értékek, nem egy
  tippet ítéletnek álcázva.

Implementáció: felvettem a `WiFi.h`/`WebServer.h`-t (mindkettő a telepített
ESP32 core része, nincs új library-függőség) a `chirpstack_js_engine.ino`-ba.
A `loop()`-ot a blokkoló `delay(uplinkIntervalSeconds*1000)`-ról
`millis()`-időzítésű, nem-blokkoló ellenőrzésre írtam át, hogy a
`server.handleClient()` minden iterációban fusson, és a dashboard reszponzív
maradjon az uplink-ciklusok között (minden `node.sendReceive()` hívás *közben*
még így is blokkol pár másodpercig, mert a RadioLib LoRaWAN API-ja szinkron -
ezt elfogadott kompromisszumként dokumentáltam, nem javítottam, mert az
aszinkronná tétel sokkal nagyobb változtatás lenne egy v0.2 firmware-ben, amit
még le sem fordítottam).

Hozzáadva: egy 20 elemű, RAM-beli ring buffer az uplinkenkénti eredményeknek
(variáns, TX-hossz, downlink érkezett/port/bájtok/RTT, RadioLib hibakód), egy
`/config` POST-handler, ami a hex DevEUI/JoinEUI/AppKey/NwkKey-t a meglévő
credential-globálisokba és NVS-be parse-olja (csak akkor indít újra a
rejoinhoz, ha ténylegesen beküldtek egy credential-mezőt; a csak-variáns
változás újraindítás nélkül érvényesül), és `/api/status` + `/api/results`
JSON-végpontok, amiket egyetlen statikus dashboard-oldal inline JS-e pollozik
(szándékosan nincs külső CSS/JS-függőség - az AP-nek nincs internet-elérése,
így egy CDN-ről húzó oldal csak törött lenne).

**Elkaptam és javítottam, mielőtt késznek vettem volna:** az első változat a
`PayloadVariant` enumot és az állapot-globálisokat (`currentVariant`,
`uplinkCount` stb.) a `setup()`/`loop()` *után* tette a fájlban, a
nem-blokkoló loop-átírást a deklarációs sorrend ellenőrzése nélkül másolva. Az
Arduino build-lépése a függvényekhez automatikusan generál forward
deklarációt, de az enumokhoz/globális változókhoz/structokhoz nem - a
`setup()` nem fordult volna le, ha a `VARIANT_ALL_ZERO`-ra az enum deklarációja
előtt hivatkozik. Átrendeztem, hogy minden típus/globális a `setup()` elé
kerüljön; a függvényeket a helyükön hagytam, mert azok *kapnak*
auto-prototípust. Eltávolítottam egy nem használt `u64ToBytes()`
segédfüggvényt is, amit spekulatívan írtam (egy végül meg nem valósított
credential-előtöltő funkcióhoz), és sosem hívtam meg.

**Hardveren továbbra sem futott** - ugyanaz a gát, mint fent (nincs eszköz
csatlakoztatva, nincs arduino-cli V4 board-csomag telepítve). Ez a kiegészítés
maga is csak statikus átnézés (az átrendezett fájl újraolvasása), nem fordítás
vagy futásidejű ellenőrzés.
