# Testbed outage diagnózis + RTL-SDR demod bringup, 2026-10-17

Két dolog történt ma: (1) a nap közben beállt ChirpStack-kiesés
diagnózisa (a node-join flakiness valódi oka), (2) az `oe-rtl-sdr-radio-scanner`
listener első valódi-hardveres futtatása és a demod jelenlegi korlátjának
jellemzése.

## 1. ChirpStack testbed KIESETT - a szerver konzolra be kell jelentkezni

Induláskor a ChirpStack élt (dual-log joinolt, provisioning ment).
~15:24 körül a node OTAA-joinja elkezdett `no JoinAccept`-et adni. Ez **nem**
node/RF/firmware/antenna hiba - a node végig helyesen küldi a join requesteket.

Hálózati diagnózis a `192.168.0.166` szerverre:
- `ping` 100% veszteség (Windows tűzfal az ICMP-re - nem árulkodó).
- **SSH (22) NYITVA**, de **8080 / 8443 / 1883 / 3000 / 9000 mind zárt** -> a
  host fut, de az egész ChirpStack Docker-stack áll.
- SSH-n (paramiko, PowerShell shell): `docker ps` ->
  `cannot connect to docker_engine ... file not found` = **a Docker engine
  (Docker Desktop) nem fut**. `Get-Process *docker*` üres. `query user` ->
  **"No User exists"** = nincs bejelentkezett interaktív session.

**Következtetés:** a Windows host (újra)indult vagy kijelentkezett, senki
nincs bejelentkezve, így a Docker Desktop (ami interaktív desktop-sessiont
igényel) nem indult el, és vele a teljes stack áll. **SSH-ból, más userként,
bejelentkezett session nélkül a Docker Desktop nem indítható megbízhatóan -
szándékosan nem erőltettem** (félkész/kaotikus állapot kockázata a termelő
szerveren).

**Teendő:** bejelentkezni a `192.168.0.166` konzoljára
(fizikailag vagy RDP-vel, `Szabolcs` user) - a Docker Desktop és a stack
onnan automatikusan feljön. Ellenőrzés: `docker compose ps` a
`C:\Users\Szabolcs\Desktop\LoRaWAN` mappában; kívülről a 8080 (web UI) és a
8443 (OTA gRPC proxy) legyen nyitva. Amíg ez nincs meg, minden hálózat-függő
teszt (SDR ground-truth, OTA, node-join) blokkolt.

## 2. RTL-SDR listener - első valódi-hardveres futtatás

Dongle: **RTL-SDR Blog V4** (R828D tuner), USB-C porton, `rtl_test` root nélkül
megy. Repo: `/home/nagy_sz/source/repos/oe-rtl-sdr-radio-scanner` (saját git,
origin/main; ide committoltam, átnézés után pusholom).

### Crash-bug javítva (commit `ca1d523`)
Első futtatáskor a dekóder-szál azonnal elszállt:
`AttributeError: 'LoraDemodulator' object has no attribute '_search_misses'`
(`lora_demod.py:386`). A `_search_misses` a preamble-hunt miss-ágán nő/nullázódik,
de **sosem volt inicializálva** az `__init__`-ben; valódi RF-en az első `feed()`
gyakran a miss-ágra ér preamble-hit előtt -> szál meghal -> **semmi nem
dekódolódhat**. A szintetikus tesztek nem fogták meg (mindig preamble-t lockolnak
előbb). Javítva `self._search_misses = 0` init; 59/59 teszt zöld.

### Demod jelenlegi korlátja (jellemezve, NEM megoldva)
Repró-jel: a node reset után ad egy join requestet; `rtl_sdr`-rel rögzítettem
14 s nyers IQ-t (868.3 MHz center, 2.048 Msps). Elemzés (`numpy`):
- **Egyetlen tiszta burst 744-807 ms-nál (63 ms, +37 dB a zajszint felett)**
  **868.46 ≈ 868.5 MHz**-en - pont egy SF7/BW125 join request. A jel tehát
  erős és egyértelműen jelen van. (Test vektor eltéve:
  `oe-rtl-sdr-radio-scanner/data/jr_868300k_2048k_joinreq.iq`, gitignore-olt.)
- Az IQ-t a listener **pontos** csatornázásával (DC-levonás, /4 decim -> 512 kHz,
  fázisfolytonos csatorna-shift) offline a demodra engedve: **a valódi 868.5-ös
  burstöt NEM dekódolta**, viszont 2 **szemét** frame-et igen (868.1 és 867.5-ön,
  random payload, `has_crc=False`).

**Diagnózis:** a demod `max_snr`-je **minden csatornán ~12-13 dB jel nélkül is**
(a zaj dechirp peak-to-average szintje egy ~128 pontos FFT-n), a detektálási
**küszöb viszont csak 9.9 dB** -> a küszöb a zajszint ALATT van -> folyamatos
ál-preamble-találat (szemét frame-ek), miközben a valódi jel nem ad megkülönböztethető,
tartós preamble-korrelációt. Ez érdemi signal-processing átdolgozást kér
(küszöb/SNR-metrika újragondolása, CFO-kezelés - a mért CFO nagy volt -, és a
512 kHz decimálás csak 4 minta/chip, ami szűkös); **több menetes kutatómunka,
nem egy-soros fix.** Elkapkodott átírást szándékosan nem csináltam, mert nem
tudnám most rendesen validálni (a ground-truth úgyis a ChirpStack-kiesés miatt
nem elérhető).

**Következő menet SDR-oldalon:** (a) küszöböt a zaj peak-to-average fölé emelni
hogy a szemét-decode-ok eltűnjenek; (b) a 868.5-ös burstre dedikáltan
debugolni a preamble-korrelációt az eltett IQ-n; (c) megfontolni a decimálás
csökkentését (8x oversampling / nagyobb sample rate). A `data/`-beli IQ ehhez
kész repró-vektor.
