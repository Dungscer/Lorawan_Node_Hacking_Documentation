# Fuzzer gateway-oldali hatás - variáns-mátrix hardveren igazolva (2026-10-04)

## Testbed és eszközök
- **Adó:** Heltec WiFi LoRa 32 V4 (SX1262), `fuzzer.ino`, `ttyACM0`.
- **Gateway:** RAK7268V2 (EUI `ac1f09fffe1f4131`), `sx130x_lora_pkt_fwd`.
- **NS + log:** ChirpStack docker-stack a 192.168.0.166-on; a ChirpStack a
  saját logját **GELF UDP:12201**-en Graylogba küldi (ez az input **már
  létezett**). Lekérdezés a Graylog REST API-ból (web/API creds: ld.
  `Information.md`, gitignore-olt), a gépről
  (`192.168.0.11`, LAN TCP OK; a `.166` ICMP-t tilt, de 22/8080/9000 nyitva).
- **Kontroll-vevő:** Heltec WiFi LoRa 32 V2 (SX1276), `fuzz_receiver.ino`,
  `ttyUSB0` - ld. lent, ez NEM hallotta a fuzzert (külön, még nyitott gond).

## A 10-03-i rejtély megoldása
A 10-03-i note szerint a fuzzer „fut+sugároz", de sem a ChirpStack API, sem
semmi nem látta a kereteit. Két külön tévedés volt benne:

1. **A V2 kontroll-vevő „süket" volt - a hiányzó antenna miatt (MEGOLDVA).**
   Megírtam és felflasheltem a V2-re a `fuzz_receiver.ino`-t (raw LoRa RX,
   868.1/SF7/BW125/sync 0x34, ugyanaz a config). A rádió-init OK, „Listening",
   de **60+50 s alatt 0 keret**. A sync kompatibilis (SX1262 0x3444 ↔ SX1276
   0x34), a config egyezik -> a vevő-sketch jó volt, csak **nem volt rajta
   antenna**. Antenna felrakása után a V2 azonnal tisztán veszi a BASELINE-t
   (`FUZZ-BASELINE`, 13 B, **RSSI ~-64 dBm**, SNR ~9 dB, CRC OK). A V2 ezzel
   működő **független on-air tanú**.

2. **A gateway VÉGIG hallotta a fuzzert - csak rossz helyen kerestem.** A
   ChirpStack a `.166`-on 10s-enként ezt logolja:
   ```
   INFO  chirpstack::gateway::backend::mqtt: Message received from gateway .../event/up
   ERROR chirpstack::uplink: Deduplication error error=unexpected major
   ```
   A 10s-es periódus = a fuzzer intervalluma (a valódi node-ok 15s). Az
   `unexpected major` pontosan a `FUZZ-*` payload érvénytelen LoRaWAN-MHDR
   major-bitjeiből jön (`'F'`=0x46 -> major=2, érvénytelen). **Bizonyíték, hogy
   a fuzzertől jön:** amikor újraflasheltem a V4-et, a 10s-es ritmusban
   **pontosan egy 17.8 s-es szünet** keletkezett (06:59:56->07:00:13), más sehol
   (median gap 10.0 s). Egy valódi node nem állna le, amikor én a fuzzert
   flashelem.

## A `setDio2AsRfSwitch` fix - hozzáadva, de NEM ez volt a blokkoló
Gyanítottam, hogy a fuzzer azért nem sugároz, mert a raw-PHY út sosem hívja a
`radio.setDio2AsRfSwitch(true)`-t (a `base_node` ezt a `LoRaWANNode`-tól
kapja). Betettem egy sort a `begin()` után. **Az adatok szerint viszont a
reflash ELŐTTI keretek (06:43->06:59) is megérkeztek a gatewayhez** -> az eredeti
firmware **is** sugárzott, a gateway hallotta. Vagyis a fix **nem** a TX-et
javította meg (az végig ment). A sor technikailag helyes az SX1262 raw-PHY
TX-hez, lehetséges link-margin haszonnal, de ez **méretlen**. Nyitott döntés:
megtartani (korrekt gyakorlat) vagy visszavenni (a feladathoz nem tartozó változtatás ne maradjon bent). A
kódban jelenleg BENT van.

## Variáns-mátrix (hardveren, 2026-10-04 ~07:10–07:26 UTC)
Módszer: variánsonként a fuzzer web-dashboardjáról beállítva, ~55 s figyelés,
majd a Graylog adott 50 s-es ablaka. (Jelenleg a fuzzer az EGYETLEN aktív adó a
hálózaton, így minden `event/up` egy fuzzer-keret - tiszta jel.)

| # | Variáns | PHY | Átjut ChirpStackbe? | NS-eredmény |
|---|---|---|---|---|
| 0 | BASELINE | explicit hdr, CRC be, "FUZZ-BASELINE" | igen | `unexpected major` |
| 1 | CRC_DISABLED | explicit hdr, **CRC ki** | ⛔ **nem** | - (gateway eldobja) |
| 2 | HEADER_LEN_LIE | **implicit hdr(255)**, 8 B | ⛔ **nem** | - (gateway eldobja) |
| 3 | MAX_LENGTH | explicit hdr, CRC be, 255 B 0xFF | igen | `unexpected major` |
| 4 | MIN_LENGTH | explicit hdr, CRC be, 1 B | igen | `unexpected major` |
| 5 | EXTREME_PREAMBLE | explicit hdr, CRC be, 1000 sym preambulum | igen | `unexpected major` |

## Értelmezés
- **A gateway-koncentrátor / `sx130x_lora_pkt_fwd` PHY-szinten szűr:** csak az
  **érvényes PHY-CRC-vel és explicit headerrel** érkező kereteket küldi fel.
  A **CRC_DISABLED** (nincs CRC -> a forwarder `forward_crc_disabled=false`
  alapértelmezése eldobja) és a **HEADER_LEN_LIE** (implicit header - a LoRaWAN
  explicit-header konfigban nem demodulálható / CRC-re bukik) **fel sem jut**
  ChirpStackbe.
- **A LoRaWAN-tartalom tetszőlegesen torz lehet, ha a PHY ép:** 255 B, 1 B és
  1000 szimbólumos preambulum **mind átmegy** a gatewayen; a ChirpStack az
  MHDR-parse-nál utasítja el (`unexpected major`). A hossz/preambulum tehát a
  gatewayt nem zavarja meg.
- **Robusztusság / nincs DoS:** sem a forwarder (a keretek és a gateway-stats
  végig folytak), sem a ChirpStack (minden torz keretet logolt hibaként, panic/
  restart nélkül) nem omlott össze egyik variánstól sem. A gateway+NS **ellenáll**
  ezeknek a PHY-szintű torzításoknak.

## On-air tanú (V2) - a dobott variánsok sugárzása
Az antennázott V2-vel megnéztem, hogy a gateway által ELDOBOTT két variáns
tényleg kimegy-e a levegőbe:
- **CRC_DISABLED:** a V2 **5 keretet** hallott, **-68 dBm**, 10 B (`FUZZ-NOCRC`).
  Tehát **sugárzik**, a gateway mégsem küldi fel -> a dobás **forwarder-politika**
  (`forward_crc_disabled=false`), nem adási hiba. (A V2 nem-CRC soft error-kóddal
  [-24] veszi, de a 10 bájtos payload és az erős RSSI egyértelmű vétel.)
- **HEADER_LEN_LIE:** a V2 csak 1 „keretet" fogott **-130 dBm**-en (zajszint) ->
  az explicit-header vevő NEM tudja dekódolni az implicit-header(255) adást
  (ahogy a gateway sem). A sugárzása ezzel a vevővel **nem igazolható**; implicit-
  header, 255 B-ra állított vevő kéne hozzá. (Nyitott árnyalat.)

## Mit igazol ez, és mit NEM
- IGAZOLT hardveren: a fuzzer sugároz; a gateway veszi; variánsonként
  forwardolja-vagy-eldobja a fentiek szerint; a NS minden forwardoltat
  `unexpected major`-ral kezel; nincs összeomlás.
- **Következtetett, nem közvetlenül olvasott:** a DROP *mechanizmusa* a
  gatewayen (CRC-disabled / implicit-header nem forwardolódik) - ezt a
  ChirpStack-oldali HIÁNYból vezetem le, **nem** a `sx130x_lora_pkt_fwd` saját
  stdout-jából (ahhoz RAK-SSH kell, az nincs). A „forwardolt vs. eldobott"
  megkülönböztetés viszont közvetlenül megfigyelt (ChirpStack megkapta-e).
- A független on-air kontroll (2. vevő, Testing TODO #3) **nem** teljesült a V2
  süketsége miatt; helyette maga a gateway a tanú (erősebb is).

## Tooling-tanulságok
- A user által beállított „gateway syslog -> .166:9000" **nem** működött és
  **nem is kellett**: a 9000 a Graylog webportja (nincs ott syslog-input), és a
  szükséges láthatóság **amúgy is megvolt** a ChirpStack saját GELF-logjában.
  A gateway-syslog út csak a *forwarder saját* PHY-drop-logját adná pluszban -
  ahhoz viszont syslog-input (514/1514) + a RAK forwarder syslog-loggolása kell.
- Graylog-query: az idézőjeles frázis (`"unexpected major"`) működik; a
  nyers `event=up` / `chirpstack::uplink` (`=`, `::`) **csendben üres**
  eredményt ad - egy korábbi sweep emiatt hamis 0-kat adott, a kontroll
  BASELINE buktatta le. Javított query után a mátrix konzisztens.
- A V4 `Serial`-je az UART0-n van (nem a `ttyACM0` natív USB-n), így a
  variánst csak a WiFi-dashboardról lehet állítani (a soros út nem járható).

## Mély `lrwn`-parser fuzzing (2. réteg, join nélkül) - 2026-10-04 ~09:08–09:16 UTC
Öt új variáns (`fuzzer.ino` 6–10) **érvényes MHDR-rel** (major=0), de beljebb
torz szerkezettel, hogy a kereteket a ChirpStack Rust LoRaWAN-parserébe
(`lrwn`) juttassuk - ez a parse a MIC/session-ellenőrzés *előtt* fut, így join
nélkül is elérhető. Baseline-vokabulária előtte: az egyetlen ERROR az
`unexpected major` volt (se WARN, se panic). Eredmény:

| # | Variáns | Gatewayen átjut? | `lrwn` eredmény (mind ÚJ, eddig nem látott) |
|---|---|---|---|
| 6 | LW_TRUNC_FHDR | igen | `at least 5 bytes are required to decode PhyPayload` |
| 7 | LW_FOPTS_LIE | igen | `MACPayload requires at least 7 bytes` |
| 8 | LW_FOPTS_FPORT0 | igen | **nincs parse-hiba** -> `chirpstack::uplink::data: No device-session exist` (INFO) |
| 9 | LW_JOIN_TRUNC | igen | `join-request payload must be exactly 18 bytes` |
| 10 | LW_PROPRIETARY | igen | `Unexpected f_type: Proprietary` |

**Megállapítások:**
- Mind az 5 keret **átjutott a gatewayen** (ép CRC + explicit header) és
  **elérte az `lrwn` parsert** - a különböző, specifikus hibák bizonyítják,
  hogy az MHDR-major ellenőrzésnél **mélyebbre** jutottam.
- **Az `lrwn` robusztus:** minden torz szerkezetre pontos, grációzus hibaüzenet;
  **egyetlen panic/crash/DoS sem**, és a ChirpStack/forwarder végig futott.
- **Legmélyebb behatolás - a 8-as variáns:** a LoRaWAN-spec-sértő `FPort=0 +
  nem-üres FOpts` kombinációt az `lrwn` **nem utasítja el** (szerkezetileg
  elfogadja), ez az egyetlen, amelyik eljut a tényleges adat-feldolgozóig
  (`chirpstack::uplink::data`). Ott viszont **ártalmatlanul elhal a
  session-keresésnél** (ismeretlen DevAddr `fc000008` -> INFO „No device-session
  exist"). A parser-engedékenység tehát megvan, de következmény nélkül: session
  nélkül nincs tovább.
- **A fal, és a kapcsolat a 3. réteghez:** a session-ellenőrzésen túl (MIC +
  joinolt eszköz) a PHY-fuzzer nem jut - a payload-codec (`decodeUplink`)
  támadásához joinolt node kell, az a `chirpstack_js_engine` sáv.
