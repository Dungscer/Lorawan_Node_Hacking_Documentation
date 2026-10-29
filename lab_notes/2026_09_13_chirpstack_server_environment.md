# ChirpStack/Graylog szerver - tesztkörnyezet felderítés (2026-09-13)

Hozzáférés részletei: `Information.md` (gitignore-olt, nem kerül a repóba).

## Host

- Windows gép, LAN-on elérhető (`192.168.0.166`), nem publikus internetről.
- SSH: OpenSSH for Windows 9.5, jelszavas authentikáció, PowerShell az
  alapértelmezett shell (nem cmd.exe) - a távoli parancsoknak PowerShell
  szintaxist kell használni (`;` a `&&` helyett stb.).
- A `chirpstack-tech` fiók egy nem-admin, korlátozott jogú felhasználó:
  csak a projekt mappájára van írás/olvasás/módosítás joga, plusz a
  `docker-users` csoporttagság miatt `docker`/`docker compose` parancsok
  futtatására. Semmi máshoz (más mappák, admin funkciók) nincs hozzáférése.
  Ez maga is egy tudatos, minimál jogosultságú hozzáférés-tervezési döntés.

## Stack (docker-compose, `C:\Users\Szabolcs\Desktop\LoRaWAN\docker-compose.yml`)

| Szolgáltatás | Image | Port(ok) | Szerep |
|---|---|---|---|
| `chirpstack` | `chirpstack/chirpstack:4` (v4.19.0) | 8080 (API+UI) | LoRaWAN network server |
| `chirpstack-gateway-bridge` | `chirpstack/chirpstack-gateway-bridge:4` | 1700/udp | Gateway↔NS packet forwarder híd |
| `chirpstack-rest-api` | `chirpstack/chirpstack-rest-api:4` | 8090 | REST/JSON ↔ gRPC gateway (a natív web UI grpc-web-et használ közvetlenül, ez egy külön, önálló REST felület) |
| `postgres` | `postgres:14-alpine` | belső (5432) | ChirpStack adatbázis |
| `redis` | `redis:7-alpine` | belső (6379) | ChirpStack cache/session |
| `mosquitto` | `eclipse-mosquitto:2` | 1883 | MQTT broker (integráció) |
| `graylog` | `graylog/graylog:5.2` | 9000 (UI), 12201 (GELF udp+tcp), 1514 (syslog udp) | Log-aggregáció |
| `mongodb` | `mongo:6` | belső | Graylog metaadat-tár |
| `opensearch` | `opensearchproject/opensearch:2` | belső (9200) | Graylog log-index |

Minden konténer docker GELF logging driverrel logol Graylogba
(`udp://127.0.0.1:12201` - a docker daemon a host loopbackjén éri el a
publikált portot, mert a logging driver a docker daemon névterében fut, nem
a compose-hálózaton belül, így a `graylog` service-név nem oldható fel neki).
Ez **más**, mint a ROADMAP-ban még nyitott "ESP32 node-ok Graylogba
logolása" tétel - az alkalmazás-szintű LoRaWAN node logokról szól, ez itt
csak a konténerek saját stdout/stderr logja.

## ChirpStack konfiguráció (`configuration/chirpstack/chirpstack.toml`)

Releváns kivonat (a teljes, annotált alapértelmezett sablon lekérdezhető a
konténerben: `docker compose exec chirpstack chirpstack --config /etc/chirpstack configfile`):

```toml
[api]
  bind="0.0.0.0:8080"
  secret="$CHIRPSTACK_SECRET"   # .env-ből, nincs benne a repóban
```

**Fontos, régóta nyitott kérdés eldőlt:** a `[api]` szekció **nem
tartalmaz** `tls_cert`/`tls_key` mezőt ebben a verzióban (v4.19.0) - a
natív gRPC TLS ChirpStack-oldalon **nem támogatott**. Ez véglegesen
eldönti a `ROADMAP.md`-ben eddig nyitva hagyott "natív TLS vs. nginx
reverse proxy" kérdést: **csak az nginx reverse proxy út járható.**
(Más szekciókban, pl. `[redis]`, `[join_server]`, `[backend_interfaces]`,
`[roaming]`, van `tls_cert`/`tls_key` - csak az `[api]`-ban nincs.)

PostgreSQL DSN: `sslmode=disable` (belső docker-hálózaton belül, nem
publikusan elérhető, ezért ez elfogadható kockázat - tudatos, dokumentált döntés, nem mulasztás).

## Amit ez lehetővé tesz

A `chirpstack` CLI-nek van `create-api-key` subcommandja - ezzel egy
rendes, minimál jogosultságú API-kulcs generálható a jelenlegi admin/admin
webes bejelentkezés helyett, ha az OTA tool számára production-szerű
tokent akarunk (ROADMAP "API token fájljog + minimál jogosultságú token"
tétele).
