# Weak/default AppKey -> OTAA takeover (RF outsider), 2026-10-05

**Szcenárió, egyértelműen keretezve (CÉLPONT-HIBA, nem a kripto törése):** tisztáztam, hogy egy rendesen bekulcsolt eszközből az RF-outsider
nem nyer ki kulcsot (ld. `2026_10_05_fuzzer_keyless_outsider.md`). Ezért a
2-es utat **„ha az üzemeltető gyenge/default AppKey-t használ, akkor…"** keretben
visszük: egyetlen, levegőből lehallgatott Join Requestből offline visszafejthető
a gyenge kulcs, abból session-kulcsok, onnan teljes eszköz-kompromittálás.
PoC: `Hacking/Weak_AppKey_OTAA/`.

## Áldozat (prop, nem valódi/titkos eszköz)
- Heltec **V3** (`ttyUSB0`), firmware: `Hacking/Weak_AppKey_OTAA/otaa_victim/`.
- **Szándékosan gyenge AppKey = `0102030405060708090A0B0C0D0E0F10`** (a mindenütt
  előforduló tutorial-kulcs). DevEUI `feedface00000001` (felismerhető PoC-érték),
  JoinEUI 0. Boot után OTAA-joinol és retry-zik -> folyamatosan ad Join Requestet.
- Compile+flash: `arduino-cli ... --fqbn Heltec-esp32:esp32:heltec_wifi_lora_32_V3`,
  hash-verified, exit 0.

## Offline cracker - valós adaton igazolva
- `appkey_crack.py`: LoRaWAN 1.0.x Join Request MIC = `aes128_cmac(AppKey,
  MHDR|JoinEUI|DevEUI|DevNonce)[:4]`. Minden mező plaintext a kereten -> minden
  AppKey-jelöltre újraszámolható és összevethető a MIC -> offline szótár/brute-force.
- **Self-test:** szintetikus gyenge-kulcsú JR -> kulcs visszafejtve + NwkSKey/AppSKey
  származtatva; **negatív kontroll:** random kulcs NEM jön ki (a támadás gyenge-kulcs-
  specifikus). Brute-force ág is OK.

## A valós lánc eddig (hardveren)
1. **Join-capture (instrumentálás, gateway-protobuf MQTT-n):** a V3 áldozat
   Join Requestje a gatewayen keresztül, dekódolva `chirpstack_api.gw.UplinkFrame`-mel:
   - **868.1 MHz, SF9, BW125, 23 bájt** (ez a join csatorna+SF -> ezzel kell a V4
     sniffert hangolni).
   - `phy = 00000000000000000001000000cefaedfe0200a7281f61`, DevEUI `feedface00000001`,
     DevNonce 0x0002, MIC `a7281f61`.
2. **Crack a VALÓS kereten:** `python3 appkey_crack.py 0000…1f61` ->
   **AppKey RECOVERED: `0102030405060708090a0b0c0d0e0f10` (via dictionary)**.

   Ezt a 23 bájtot most a **gateway MQTT**-ről olvastam
   (insider instrumentálás), hogy a join SF/csatornát megtudjam és előre
   validáljam a cracket. A bájtok azonosak, akárhonnan jönnek, és a crack matek
   nem függ a capture módjától - DE a végső, támadó-hiteles artefakthoz a
   **V4 sniffernek RF-en, függetlenül kell elkapnia ugyanezt**. Ez a nyitott lépés.

## V4 független RF-sniff - KÉSZ (támadó-hiteles capture)
- Firmware: `Hacking/Weak_AppKey_OTAA/join_sniffer/` - raw-LoRa RX 868.1/SF9-en,
  saját WiFi-AP-dashboarddal (a capture-t a támadó a saját eszközéről olvassa).
  Flash a V4-re (`ttyACM0`, V3 FQBN), hash-verified.
- A V4 **saját rádiójával** elkapott **9 Join Requestet** (RSSI ~−120 dBm, valódi
  RF-vétel), DevEUI `feedface00000001`, növekvő DevNonce (0x1d…0x33).
- **Crack a V4-sniffelt (NEM MQTT) kereteken:** mind ugyanarra a kulcsra tört:
  `0102030405060708090a0b0c0d0e0f10`. Pl. DevNonce 0x33/0x31/0x2f/0x1d -> mind OK.
- **Ezzel a capture->crack lánc teljes és támadó-hiteles:** kulcs nélküli RF-vevő
  a levegőből sniffel egy Join Requestet, és gyenge kulcs esetén offline visszafejti.

## Weaponizálás - KÉSZ (teljes session-kompromittálás, bitre igazolva)
1. Az áldozatot regisztráltam a ChirpStackbe a **feltört** kulccsal (REST API,
   app `Application_01`, profil `OTAA` 1.0.0, DevEUI `feedface00000001`,
   nwkKey=`0102..0F10`). A V3 reset után lejoinolt.
2. **Friss join-csere elkapva:** Join Request DevNonce=0x0000, Join Accept
   (titkosított) `20bbc374…7d089a`. (A Join Request RF-sniffjét a V4 külön
   bizonyította; a weaponizálás join-cseréjét MQTT-instrumentálással vettem.)
3. **Támadói levezetés CSAK a sniffelt+feltört anyagból:** a feltört AppKey-jel
   dekódoltam a Join Accept-et -> AppNonce `010000`, NetID `000000`, DevAddr
   `0120498c` -> NwkSKey/AppSKey. **Bitre egyezik a ChirpStack belső session-ével:**
   DevAddr `0120498c`, NwkSKey `1f017cfe881d7a464117025e3d600172`,
   AppSKey `14a68b1989345c76f39fbb15eb3d0575` - mind == ground truth.
4. **Confidentiality-törés élőben:** valós uplink (DevAddr `0120498c`, fCnt 6,
   fPort 2), FRMPayload `24` -> a derived AppSKey-jel dekódolva **`a5`** = pontosan
   az áldozat `0xA5` payloadja. A NwkSKey birtokában a keret-hamisítás is adott.

**EREDMÉNY:** gyenge/default AppKey esetén egy kulcs nélküli RF-outsider egyetlen
sniffelt join-cseréből **teljesen átveszi az eszköz session-ét** (olvassa és
hamisíthatja a forgalmát). High-severity, végig hardveren + valós kriptón igazolva.
**Severity-keret:** a célpont kulcskezelési hibája (gyenge kulcs), nem a LoRaWAN/
ChirpStack hibája - rendes random AppKey ellen a lánc a crack-lépésnél elbukik.

## Állapot / testbed (szándékosan bent hagyva, dokumentálva)
- ChirpStack: a `feedface00000001` PoC-eszköz (app `Application_01`, profil `OTAA`,
  gyenge kulcs) **szándékosan bent marad** a tézis-demó reprodukálhatóságáért -
  ugyanaz a konvenció, mint az `ATTACK-jscodec` profilnál. Visszavonható bármikor:
  `DELETE /api/devices/feedface00000001`. Mivel gyenge, publikus demo-kulcs és
  izolált PoC-eszköz, nem jelent valós kockázatot a testbedre.
- V3 = áldozat (gyenge kulcs), V4 = sniffer - maradhatnak flashelve a demóhoz.
- Nincs valódi titok a PoC-ban (a gyenge kulcs szándékosan publikus demo-érték).
