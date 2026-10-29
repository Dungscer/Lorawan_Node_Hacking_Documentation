# API token minimál jogosultságra cserélve + Docker Desktop leállás (2026-09-19)

## Docker Desktop nem futott - megoldva

Induláskor a ChirpStack/Graylog szerver nem válaszolt
(`chirpstack: 000`), a Docker daemon sem volt elérhető SSH-n keresztül
(`failed to connect to the docker API at npipe:////./pipe/docker_engine`).
A gép fel volt kapcsolva (SSH ment), de a Docker Desktop nem futott -
valószínűleg egy újraindulás óta. `Start-Process` SSH-ból nem tudta
elindítani (ugyanaz a Windows-session-korlát, mint korábban a hitelesítő-
helpernél: Docker Desktop interaktív desktop-sessiont igényel).

**Megoldás:** fizikailag elindítottam a Docker Desktopot, és
bekapcsoltam a "Start Docker Desktop when you sign in" beállítást - ez után
minden konténer (`chirpstack`, `graylog`, `nginx-grpc-tls` stb.) rendben
felállt, TLS is működik változatlanul.

## API token - tenant-scope-olt kulcsra cserélve

A ROADMAP "API token minimál jogosultságú token" tétele: ChirpStack API
kulcs modellje csak két szintet ismer (`is_admin=true` = globális admin,
`is_admin=false` + `tenant_id` = egy tenantra korlátozott) - nincs
finomabb, erőforrás-szintű jogosultság-rendszer. A `Tools/OTA_Update`
eddig admin/admin webes belépéssel generált, ideiglenes tokeneket
használt teszteléshez; most egy **állandó, tenant-scope-olt** kulcs lett
létrehozva (`ota-update-tool`, id `945b6e00-...`), ez van a
`Tools/OTA_Update/config.json`-ban (chmod 600, gitignore-olt) és az
`Information.md`-ben.

## Váratlan lelet - ismert ChirpStack hiba, ártalmatlannak bizonyult

Az első teszt (tenant-scope-olt kulccsal, egy **nemlétező** all-zero
DevEUI-re `GetQueue`) `UNAUTHENTICATED`-del elhasalt, miközben egy
globális admin kulcs ugyanarra a hívásra rendben lefutott. Ez pánikra
adhatott volna okot, de utánanézve ez egy **dokumentált, ismert ChirpStack
hiba** (upstream GitHub issue
[#1016](https://github.com/chirpstack/chirpstack/issues/1016) és
[#1018](https://github.com/chirpstack/chirpstack/issues/1018), plusz több
fórumbejegyzés): tenant-scope-olt kulcsok `UNAUTHENTICATED`-et adnak
`NOT_FOUND` helyett, ha a lekérdezett DevEUI nem létezik - egy hibás
permission-check subquery miatt.

**Ellenőriztem valódi (regisztrált) eszközzel is**: ugyanaz a tenant-
scope-olt kulcs **hibátlanul működött** (`GetQueue` OK, depth 0). Tehát ez
a bug **nem gyakorlati akadály** a tool tényleges használatában (mindig
létező eszközzel dolgozik), csak szintetikus/nemlétező-DevEUI-s tesztekben
és diagnosztikában tud megtévesztő legyen - ezt a `README.md`-ben is
dokumentáltam, hogy legközelebb ne okozzon feleslegesen pánikot.

## Környezet visszaállítása

A teszt közben létrehozott ideiglenes API-kulcsok (`test-fresh-key`,
`test-fresh-admin-key`, egy duplikált korai `ota-update-tool` kulcs) és a
valós-eszközös teszthez létrehozott ideiglenes device törölve lettek -
csak a végleges `ota-update-tool` kulcs maradt meg.
