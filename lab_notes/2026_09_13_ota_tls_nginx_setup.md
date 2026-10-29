# OTA gRPC TLS - nginx reverse proxy setup (2026-09-13)

## 1. döntés: natív ChirpStack TLS vs. nginx reverse proxy

Megnéztem a ténylegesen futó ChirpStack (v4.19.0) teljes, annotált
konfigsablonját közvetlenül a konténerben:

```
docker compose exec chirpstack chirpstack --config /etc/chirpstack configfile
```

A `[api]` szekció (bind + secret) **nem tartalmaz** `tls_cert`/`tls_key`
mezőt - más szekciókban (pl. `[redis]`, `[join_server]`, `[roaming]`) van
ilyen, csak az API-ban nincs. Tehát a natív TLS ezen a verzión **nem
elérhető**, kizárólag az nginx reverse proxy út járható. Ez véglegesen
eldönti a korábban nyitva hagyott kérdést.

## 2. CA + szerver cert generálása

A recept a `ROADMAP.md`-ben leírtak szerint, helyben (nem a szerveren):

```bash
openssl req -x509 -newkey ed25519 -keyout lab-ca.key -out lab-ca.crt -days 3650 -nodes -subj "/CN=LoRaWAN Lab CA"
openssl req -newkey rsa:2048 -keyout chirpstack.key -out chirpstack.csr -nodes -subj "/CN=chirpstack.lab.internal"
echo "subjectAltName = DNS:chirpstack.lab.internal, IP:192.168.0.166" > chirpstack.ext
openssl x509 -req -in chirpstack.csr -CA lab-ca.crt -CAkey lab-ca.key -CAcreateserial -out chirpstack.crt -days 825 -extfile chirpstack.ext
```

A SAN-ba szándékosan mindkettő bekerült (DNS név ÉS az IP is) - így a
kliens akár az IP-t, akár a DNS-nevet használja `device_api_url`-ként, a
tanúsítvány-ellenőrzés megy `ssl_target_name_override` nélkül is (ezt
külön leteszteltem mindkét formában).

**Kulcsok elhelyezése:** `lab-ca.key` és `chirpstack.key` a
`Tools/OTA_Update/keys/` alá kerültek (gitignore-olt), a publikus
`lab-ca.crt` a `Tools/OTA_Update/certs/lab-ca.crt`-ba (ez commitolható,
nincs benne titok). **Fontos:** a `lab-ca.key`-t érdemes lenne biztonságos
helyre (jelszókezelő, titkosított meghajtó) is lementeni - ha ez elveszik,
egy új CA-t kell csinálni és minden klienst újrakonfigurálni, ha valaha új
certet kell kiállítani ugyanezzel a CA-val.

## 3. nginx bekötése a szerveren - két váratlan akadály

A szerver (`192.168.0.166`, Windows, docker-compose stack,
`C:\Users\Szabolcs\Desktop\LoRaWAN`) SSH-val elérhető
(`chirpstack-tech`, ld. `Information.md`).

### 3a. Docker Desktop hitelesítő-helper nem működik SSH session alól

`docker pull` / `docker compose up -d <új image>` minden esetben elhasalt:

```
error getting credentials - err: exit status 1, out: `A specified logon session does not exist. It may already have been terminated.`
```

Ez **bármilyen** image pull-ra jelentkezett (még `docker pull alpine` is),
tehát nem image-specifikus. Ok: Docker Desktop Windows-on a hitelesítő
helperje (`docker-credential-desktop.exe`) a Windows-os interaktív
desktop-session hitelesítő tárolójához (DPAPI/Credential Manager) nyúl,
ami egy sima SSH `exec` session alól (nincs interaktív desktop-token) nem
érhető el. Ez nem oldható meg config-fájl szerkesztéssel
(`~/.docker/config.json`-t üresre állítva sem segített) - ez egy
Windows-session-szintű korlát.

**Megkerülés:** mivel `alpine:latest` már cache-elve volt helyben (egy
korábbi telepítésből), egy saját nginx image-et építettem belőle
`apk add nginx`-szel, ami **nem** megy át a registry hitelesítésen (az
Alpine csomagkezelő sima HTTPS-t használ, nem Docker registry auth-ot).

### 3b. Build-context path feloldás elakad egy Windows path-traverse problémán

`docker compose build` (relatív `context: ./configuration/nginx` úttal) így
bukott:

```
failed to evaluate path "C:\Users\Szabolcs\Desktop\LoRaWAN\configuration\nginx": CreateFile C:\Users\Szabolcs\Desktop: Access is denied.
```

Ugyanez a minta, mint amit korábban (2026-09-09) a `git rebase` során is láttam: a `chirpstack-tech` fiók sima fájlműveletei
(`Get-Content`, `Get-ChildItem`, SFTP put) a teljes elérési úton
hibátlanul működnek, de a **PowerShell `Set-Location`** és a **Docker
buildkit "local bake" path-kiértékelése** külön-külön elakad ugyanazon a
`...\Desktop` szegmensen - feltehetően valamilyen Windows-oldali
traverse-ellenőrzési különbség (nem sima NTFS ACL-deny, mert akkor a
fájlműveletek sem mennének át).

**Megkerülés:** `docker build -f - .` - a Dockerfile-t stdin-en át küldve
(paramiko `exec_command` + `stdin.write(...)`), így buildkit nem próbál
lokális build-context útvonalat feloldani egyáltalán. Mivel az nginx.conf
és a certek amúgy is volume mountként kerülnek be futáskor (nem `COPY`-val
az image-be), a Dockerfile-nak nem is kellett más fájl, csak
`FROM alpine:latest` + `RUN apk add --no-cache nginx`.

A `docker-compose.yml`-ben mindkét munkakörülményt kommentben
dokumentáltam, hogy legközelebb ne kelljen újra kideríteni.

## 4. Végeredmény - ellenőrizve

```
openssl s_client -connect 192.168.0.166:8443 -CAfile lab-ca.crt -servername chirpstack.lab.internal
# -> verify return:1, teljes cert-lánc rendben

python3 -c "... grpc.secure_channel('192.168.0.166:8443', creds) ... isvc.Login(...)"
# -> sikeres, JWT vissza (IP-cím alapú célnál, ssl_target_name_override NÉLKÜL is)

python3 diagnose.py  (device_api_url=192.168.0.166:8443, tls.enabled=true)
# -> GetQueue OK, FlushQueue OK - a hibák (GetDevice/Enqueue NOT_FOUND) kizárólag
#   a teszt dev_eui nemlétezéséből jönnek, nem TLS/kapcsolati hibák
```

A teljes `Tools/OTA_Update` lánc (config -> `grpc_client.build_channel()` ->
hitelesített hívás) működik az új TLS végponton, IP-cím alapú
`device_api_url`-lel is, plusz szerver-oldali sértetlenség (a meglévő
konténerek, a web UI :8080-on, nem sérültek - csak egy új service jött
létre).
