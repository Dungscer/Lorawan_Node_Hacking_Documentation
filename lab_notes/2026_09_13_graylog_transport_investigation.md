# Graylog transport-választás ESP32 node logoláshoz - nyomozás (2026-09-13)

## Előzmény

Cél: az ESP32 LoRaWAN node-ok WiFi-n keresztül logoljanak a Graylogba
(ROADMAP.md 0. sáv). Az eredeti terv GELF UDP (12201) volt, ez már meglévő,
működő input a szerveren.

## 1. lelet - GELF UDP: hálózati útvonal probléma, NEM a szerveren

A szerveren futtatott diagnosztikai lépéssor
(`udp_reachability_diagnostic_prompt.txt`) megerősítette: **loopback
(127.0.0.1) ÉS LAN-IP (192.168.0.166) célú UDP is sikeresen megérkezik** a
Graylogba, ha a szerver gépéről küldik. Tehát a Windows Firewall szabály
(`Graylog GELF UDP 12201`, amit korábban felvettem) és maga a
Docker/Graylog-oldal **rendben van**.

Ugyanakkor, **erről a gépről** (ami - kiderült - ugyanazon a LAN-on van,
`192.168.0.11`, közvetlen route, nincs VPN/tunnel közte és a szerver
között) UDP a 192.168.0.166-ra **semmilyen porton nem érkezik meg** -
tesztelve mind a 12201-en, mind egy teljesen friss, semmilyen szabályhoz
nem kötött porton (natív, Docker-mentes PowerShell UDP listenerrel is).
Kizártam: ezen a gépen (ahonnan én dolgozom) az `iptables OUTPUT` lánc
üres (default ACCEPT, nincs szabály), `ufw` nincs telepítve, és az
outbound UDP általánosságban működik (DNS-feloldás rendben megy UDP/53-on).
A `sendto()` hívás maga sem dob hibát.

**Következtetés:** valahol a WiFi router/AP szintjén van egy szűrés, ami az
ismeretlen/nem-jól-ismert UDP-forgalmat (pl. a 12201-es GELF-et) blokkolja
két LAN-kliens között, miközben a TCP-t (SSH, HTTP) átengedi. Ez nem
javítható a szerver oldali Windows Firewallal - a router/AP konfigurációját
kellene megnézni, amihez itt nincs hozzáférés.

## 2. lelet - GELF TCP és GELF HTTP: fogadja, dekódolja, de SOSEM indexelődik

Mivel az UDP-út járhatatlannak tűnt (router-szintű, nem javítható innen),
átváltottam GELF TCP-re. Első lépésként létrehoztam egy GELF TCP inputot
a már publikált `12201/tcp` porton (`ESP32 Node GELF TCP`,
id `6aa67882f0db07231fd46904`).

**Teszt:** több GELF 1.1 JSON üzenetet küldtem nyers TCP-n (null-bájttal
lezárva, ahogy a GELF TCP spec kéri), közvetlenül Python `socket`-tel.

**Eredmény, ami nem várt volt:**
- Az input metrikái (`/api/system/metrics/namespace/org.graylog2.inputs.gelf.tcp.GELFTCPInput`)
  szerint `incomingMessages` növekszik (minden küldött üzenetre +1),
  `invalidMessages: 0`, `emptyMessages: 0` - tehát az input **fogadja és
  sikeresen dekódolja** az üzeneteket.
- A Graylog konténer saját logjában (`docker compose logs graylog`)
  **nincs semmilyen hiba/kivétel** a teszt-üzenetek idejére - összevetve
  egy korábbi, tényleg hibás UDP-üzenettel (`test` szó JSON helyett), ahol
  a log azonnal `JsonParseException`-t dobott. A JSON üzeneteimre
  semmi ilyen nem jelent meg - csendben "eltűnnek" valahol dekódolás után.
- **Mégsem jelennek meg SOHA a keresésben** - sem a `facility:...` mezőre,
  sem a `gl2_source_input:<input-id>` mezőre, sem egy teljesen tág (`*`,
  100 találatig, 10 perces ablak) keresésre.
- A keresési módszertant magát is ellenőriztem: **a korábban
  confirmed UDP teszt-üzeneteim (`facility:udp-diagnostic`)
  ugyanazzal a keresési móddal simán megtalálhatók** - tehát a keresési
  API/módszer működik, a probléma tényleg az, hogy a TCP-s üzenetek nem
  jutnak el az indexbe.
- Ugyanezt megismételtem egy **teljesen más input-típussal is (GELF HTTP,
  `12203/tcp`, új docker-compose port + `graylog` konténer újraindítva)** -
  **pontosan ugyanaz a viselkedés**: `incomingMessages` nő, nincs hiba a
  logban, de a keresésben soha nem jelenik meg.
- Index-set szinten nincs eltérés: mind a "Default Stream", mind a
  működő "Chirpstack" stream ugyanabba az index-setbe ("Default index
  set", `6a5ef56c8a3f836eda9357c0`) írnak - tehát nem egy külön,
  "láthatatlan" indexbe kerülnek a TCP/HTTP üzenetek, egyszerűen valahol a
  dekódolás UTÁN, az indexbe írás ELŐTT tűnnek el.
- Közös vonás a két hibás inputnál: mindkettő `AbstractTcpTransport`
  alapú (a működő GELF UDP nem az), és mindkettőnél megjelenik egy
  `receiveBufferSize (SO_RCVBUF) ... should be >= 1048576 but is 425984`
  figyelmeztetés induláskor - ez önmagában "csak" teljesítmény-figyelmeztetés
  szokott lenni, de itt gyanús egybeesés, hogy pont a két TCP-alapú
  inputnál jön elő, a working UDP-nél nem.

**Amit ez sugall:** ez nem hálózati/tűzfal probléma (a gépen belül,
loopback-en keresztül is reprodukálódik - a teszteket innen, a szerverrel
azonos LAN-ról küldtem, TCP-n keresztül, ami korábban minden más célra
(SSH, HTTP API-k) rendben működött). Ez valószínűleg egy Graylog-oldali,
ezen a konkrét telepítésen (Docker Desktop / Windows / WSL2 hálózati
verem) jelentkező, TCP-transport-specifikus feldolgozási hiba/bug - a
`AbstractTcpTransport`-alapú inputoknál valahol a dekódolás és az
index-írás között némán elvesznek az üzenetek, hiba/log nélkül.

## Jelenlegi állás

**Egyik tesztelt transport sem bizonyítottan működő végponttól-végpontig
egy tetszőleges LAN-kliens (mint amilyen az ESP32 is lenne) felől:**

| Transport | Szerver-oldal | Kliens-oldali probléma |
|---|---|---|
| GELF UDP (12201) | működik (loopback+LAN-IP a szerverről tesztelve) | a router/AP feltehetően blokkolja LAN-kliensek között |
| GELF TCP (12201) | fogadja+dekódolja, de sosem indexelődik | - (a hálózat nem a gát, a szerver-oldali feldolgozás) |
| GELF HTTP (12203) | ugyanaz, mint a TCP-nél | - |
