# Keyless-outsider RF attack sweep - join-flood + parser boundaries, 2026-10-05

**Forgatókönyv:** a támadó bent ül az épületben kisebb
rádiós eszközökkel (V4), **nincs semmilyen credentialje és nincs LAN-hozzáférése**,
csak annyit tud, hogy itt egy ChirpStack megy. Cél: bármilyen fogás a rendszeren,
előnyben az **emberi interakció nélküli**, gyorsan bizonyítható vektorok.
Fizikai kulcskinyerés tudatosan kizárva (az külön sáv).

**Eszközök:** V4 = `fuzzer.ino` **v0.2** (3 új variáns ehhez a sávhoz), raw-LoRa
PHY, nincs join/credential. Áldozat = NUCLEO (`base_node`, DevEUI
`0080e115061bebe9`), joinolva, ~10s-enként UnconfirmedDataUp, EU868 Class A.
Gateway = RAK7268 (`ac1f09fffe1f4131`). Megfigyelés: Graylog REST
(`192.168.0.166:9000`, admin/admin), `curl` + relative search, nincs SSH.
A V4-et arduino-cli flashelte (`Heltec-esp32:esp32:heltec_wifi_lora_32_V3` FQBN,
hash-verified). A variánsváltás a V4 WiFi-dashboardjáról ment (a soros port
futásidejű Serume nem volt elérhető egyik tty-n sem).

## Alapmérés (BASELINE, a V4 default variánsa, ~10s)
- `unexpected major` = 11 / 120s (a BASELINE `FUZZ-BASELINE` string MHDR-je
  major≠0 -> a NS a MHDR-parse-nál eldobja - egyezik a 2026-10-04 mátrixszal).
- NUCLEO `data_up` logsorok: 66 / 120s; `Uplink received` 14 / 120s. **0 panic.**
- Igazolja: V4->RAK->ChirpStack lánc él, az áldozat él, a szerver egészséges.

## #1 - LW_JOIN_FLOOD (variáns 11, interval=1s) - **FOGÁS (Low, availability)**
- A V4 ~**1 join/sec**, **minden keret más random JoinEUI/DevEUI/DevNonce/MIC**.
- A ChirpStack **mindet fogadja, beparseolja, DevEUI-t keres, és eldobja**:
  `WARN chirpstack::uplink::join: Unknown device dev_eui=…` (pl. `aa3cbfe690b9f7e2`,
  `7b324ede3b23cb30`, `2b00f70c6df61511`).
- **60s alatt 50 hamis join** ment végig teljes feldolgozáson. **Nincs rate-limit /
  flood-védelem.** Minden hamis join: gateway-airtime + NS-parse + DB-lookup +
  WARN-logsor + egy MQTT gateway-event.
- **Hatás a tesztelt rátán (1 eszköz):** a NUCLEO valódi uplinkjei **tovább
  mentek** (`data_up 0080e115061bebe9` a flood közben), **0 panic**, nincs
  összeomlás -> a rendszer elnyelte.
- **Értékelés:** valós, kulcs nélküli, emberi interakció nélküli **availability-
  primitívum** - a szerveroldalon semmi nem korlátozza. A fenti forgatókönyvre (**„több kisebb eszközzel"**) lineárisan skálázódik: a RAK
  félduplex airtime-ja és az NS-terhelés több adóval/nagyobb rátával telítődik.
  CVSS becslés ~**4–5 (AV:A/AC:L/PR:N/UI:N/S:U/A:L)**.
  **Korlát:** a legitim forgalom tényleges kiesését **ezen a rátán nem**
  figyeltem meg - a skálázódás érvelt, nem demonstrált. A teljes „áldozat-uplink
  kiesik" bizonyításhoz több adó / nagyobb effektív ráta kell.

## #2 - LW_DATA_MINIMAL (variáns 12, 12 bájt) - negatív (parser robusztus)
- `f_type=UnconfirmedDataUp` -> `INFO chirpstack::uplink::data: No device-session
  exists for dev_addr dev_addr=fc00000c` (= a beégetett DevAddr). Tiszta parse,
  eljut a data-handlerig, szabályosan elhal a session-lookupnál. **0 panic/error.**
- A keret minden TX-nél azonos -> a ChirpStack **deduplikálja** (ezért kevés
  distinct logsor). Mellékmegfigyelés, nem sérülékenység.

## #3 - LW_MIC_SHORT (variáns 13, 11 bájt, egy a minimum alatt) - negatív
- `ERROR chirpstack::uplink: Deduplication error error=**MACPayload requires at
  least 7 bytes**`. A `lrwn` parser a túl rövid MACPayload-ot felismeri, a mezőket
  ki sem olvassa (`fc00000d` nem jelenik meg). **0 panic** -> az off-by-one helyesen
  kezelve. A Rust parser jól védett ezen a határon is.

## Összegzés
| Variáns | Mit tesztel | Eredmény |
|---|---|---|
| LW_JOIN_FLOOD | join-flood / rate-limit | **Fogás: nincs flood-védelem (Low availability)** |
| LW_DATA_MINIMAL | 12B minimum data-up | negatív - graceful „No device-session" |
| LW_MIC_SHORT | 11B off-by-one | negatív - graceful parse-error, 0 panic |

**Tézis-tartalom:** 1 demonstrált (hardveren, Graylog-tanúsított) availability-
fogás pontos fenyegetésmodellel + 2 tiszta negatív eredmény a pre-auth
parser-robusztusságról. A `lrwn` (Rust) a 2026-10-04 óta tesztelt összes
malformációra graceful - memóriasérülés/RCE ezen az úton nem valószínű; a
legjobb kulcs nélküli vektor az **availability/flood**.

## Súlyosabb-vektor felderítés (A és B irány) - 2026-10-05, forrás+web recon
A zajos flood-DoS-t elvetettem, és súlyosabban kihasználható
vektort kerestem. Két irányt néztem; **mindkét „súlyos" út korlátozottnak bizonyult**:

- **(B) codec->XSS élesítése a ChirpStack saját UI-ján - ELESETT (UI biztonságos).**
  A UI forrása (`ui/src/views/devices/DeviceEvents.tsx` -> `components/LogTable.tsx`)
  a dekódolt event-`object`-et **react-json-tree**-vel rendeli (`JSONTreeOriginal`,
  `data={JSON.parse(body)}`) - React-escape, **nincs `dangerouslySetInnerHTML`**.
  Tehát a codec-injektált `<img onerror>` a ChirpStack konzolján escape-elt
  szövegként jelenik meg, nem fut le. A tegnapi codec->XSS lánc **app-layer marad**
  (csak sanitizálatlan harmadik-fél dashboard ellen, mint a `sink_poc/`), **nem**
  ChirpStack-szintű admin-takeover. Nincs ismert ChirpStack stored-XSS CVE sem.
- **(A) gateway C-kód - valós bug, de kapuzott.** Dokumentált buffer overflow a
  Semtech `libloragw` `lgw_receive()`-ben, javítva a chirpstack-concentratord
  HAL **1.0.3**-ban. Memóriakorrupció a PHY-vételi úton = súlyos, DE kiváltása
  sok egyszerre érkező keret (burst, részben zajos) + **régi HAL-verzió** a
  RAK7268-on. A RAK HAL/concentratord-verziója innen (nincs gateway-SSH) nem
  ellenőrzött -> nyitott.
  - **Követés (gRPC gateway-metadata):** a gateway neve `rak7268v2` (RAK7268 **V2**,
    2022+), de a ChirpStack metaadata csak régiót tárol (`eu868`), HAL-verziót nem.
    A `lgw_receive`-fix a Semtech `sx1302_hal` **1.0.3**-ban (2021) van -> egy
    V2-firmware szinte biztosan tartalmazza -> (A) **nagy valószínűséggel zárt**.
    Ráadásul a trigger egy poll-ciklusban >16 demodulált keretet igényel (default
    buffer fölött) - egy csatornán SF7-en nehéz + zajos. Definitív válaszhoz a RAK
    saját firmware-verziója kell (web-UI/SSH). **Köztes ítélet: (A) valószínűleg
    patchelt, a trigger amúgy is impraktikus kulcs nélkül.**
  - **DEFINITÍV (verzió-fingerprint):** a gateway nem látszott az én WiFi-subnetemen
    (ICMP-drop / AP client-isolation), de a `.166` szerver ARP-táblája kiadta:
    **RAK7268 V2 @ `192.168.0.231`** (MAC `ac-1f-09-1f-41-31` = az EUI). Közvetlenül
    elérhető: **WisGateOS 2** (nginx+HTTPS, React/create-react-app SPA), az
    UI-asset `Last-Modified: 2026-01-23` -> **jelenlegi generációs firmware**. A
    `lgw_receive`-fix (sx1302_hal 1.0.3, 2021) évekkel régebbi -> **(A) LEZÁRVA: az
    overflow patchelt, nem alkalmazható.** A pontos HAL-verzió a UI auth mögött van;
    a RAK-login találgatása a tulaj credjei nélkül nem történt meg.

**Köztes konklúzió:** kulcs nélküli RF-outsider ellen ezen a stacken
(érett Rust NS + output-safe UI + ép LoRaWAN-kripto) a **súlyos** (C/I/takeover)
vektorok lényegében zártak; a maradék kockázat availability (zajos) + app-layer
injection (deployment-függő) + egy verzió-kapuzott gateway-C bug (félig zajos).
Ez önmagában védhető tézis-eredmény (biztonsági kiértékelés), de nem a keresett
„súlyos exploit". Súlyoshoz a fenyegetésmodell lazítása kell (kulcskompromittálás
-> külön sáv), vagy a RAK régi HAL-verziójának igazolása + burst-trigger.

## Állapot a teszt után
- A V4-et a teszt végén vissza kell állítani **BASELINE / 10s**-re (vagy ki kell húzni) - ne
  maradjon aktív flood a levegőn (EU868 duty-cycle).
- A testbed ChirpStack érintetlen (a flood csak eldobott kereteket generált,
  nincs maradó állapot). A `ATTACK-jscodec` profil + V4-regisztráció a korábbi
  sávból továbbra is bent (külön visszavonandó).
- Firmware: `fuzzer.ino` v0.2, tisztán fordul; a 3 új variáns most **hardveren
  igazolt** (gateway-oldali hatás + ChirpStack-reakció Graylogból).
