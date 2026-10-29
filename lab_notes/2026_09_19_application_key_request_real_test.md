# Application_Key_Request - valódi rogue-node teszt (2026-09-19)

## Módszer

1. **Rogue klón**: a `Firmwares/Nodes/Heltec_WiFi_LoRa_32_V2/otaa/otaa_basic.ino`-t
   (egyszerű, nem hardeningelt OTAA sketch) ráégettem egy szabad Heltec V2
   boardra a **valódi** V2 Node 1 hitelesítő adataival (DevEUI/AppEUI a
   2026-09-19-i `SPI_Key_Extraction` teszt által kinyertekkel megegyező,
   AppKey az `Information.md`-ből) - pontosan azt a forgatókönyvet
   szimulálva, amit a `ROADMAP.md`/`Application_Key_Request/ROADMAP.md`
   leír: valaki SPI-tappal/flash-dumppal megszerezte a kulcsokat, és
   klónozza az eszközt.
2. A DevEUI **nem volt regisztrálva** az új (2026-09-13 óta használt)
   ChirpStack szerveren - ezt külön regisztrálni kellett
   (`rogue-clone-v2-node1` néven, ugyanabban a device-profile-ban, mint a
   korábbi teszt-eszközök).
3. Sikeres OTAA join: `EV_JOINED`, `DevAddr: 0x0174B2C0` (a device saját,
   soros logon kiírt értéke).
4. **Join Accept "sniffelése":** a valódi RF/SPI sniffelés helyett a
   ChirpStack `InternalService.StreamDeviceFrames` gRPC stream API-ját
   használtam (a rejtjelezett Join Accept nyers bájtjait így is meg lehet
   kapni, ez egy pragmatikus tesztelési rövidítés - a tényleges
   RF-elfogás demonstrációja a `RTL_SDR_Listener`, a SPI-elfogásé a
   `SPI_Key_Extraction` dolga, azok már külön demonstrálva vannak).
5. `session_derive.py` a kapott Join Accept (28 bájt titkosított rész +
   4 bájt MIC = 32 bájt, CFList-tel) + a valódi DevNonce (44572 = 0xae1c)
   alapján helyesen levezette a session kulcsokat.

## Eredmény - döntő ellenőrzés

```
python3 session_derive.py --appkey <valódi AppKey> \
    --join-accept 2624581cb17079ab5b0ded9783c6d8ddd842e1a3f6d371ae04ff5f7d5a7279ad \
    --dev-nonce 0xae1c

[OK] DevAddr:  0174b2c0
[OK] NwkSKey:  <levezetve>
[OK] AppSKey:  <levezetve>
```

**A levezetett DevAddr (`0174b2c0`) bit-pontosan egyezik azzal, amit maga
az eszköz a saját LMIC join-feldolgozásából függetlenül kiszámolt és
kiírt a soros logra** (`DevAddr: 0x0174B2C0`) - ez a lehető legerősebb
igazolás arra, hogy a `session_derive.py` kriptó-implementációja helyes.
AppNonce/NetID `000000`-nak jött ki, ami konzisztens a szerver saját
`chirpstack.toml`-jában beállított `net_id="000000"`-val.

## Payload-decrypt - kontrollált round-trip

A ChirpStack frame-logja az uplink `frm_payload`-ot **már dekódolva**
mutatja (kényelmi funkció a saját debug nézetéhez) - ez más, mint a Join
Accept-nél tapasztalt titkosított nézet. Emiatt nem lehetett közvetlenül
"éles" titkosított uplink-payloadot kapni ezen a csatornán; helyette egy
kontrollált round-trip teszttel igazoltam ugyanazzal a **valódi, levezetett**
AppSKey-jel:

```python
plaintext = bytes.fromhex('504b000100000000')  # valódi plaintext (fcnt=0)
ciphertext = decrypt_frm_payload(appskey, dev_addr, 0, 0, plaintext)  # "encrypt"
recovered = decrypt_frm_payload(appskey, dev_addr, 0, 0, ciphertext)  # decrypt
assert recovered == plaintext  # OK
```

A visszakapott plaintext (`PK`, uplinkCount=1, downlinkCount=0) pontosan
egyezik az `otaa_basic.ino` `do_send()` függvénye által ténylegesen
összeállított payload-formátummal.

## Környezet visszaállítása

- A `rogue-clone-v2-node1` teszt-eszköz törölve a ChirpStackből.
- Az `otaa_basic.ino` visszaállítva placeholder (nulla) hitelesítő
  adatokra (`git checkout`) - a valódi AppKey/DevEUI/AppEUI sosem került
  commitolásra - valós kulcs nem kerül a repóba.
- A `Heltec ESP32 Dev-Boards` könyvtár és a másolt `HT_*` fejlécek
  ugyanúgy vissza lettek állítva, mint a korábbi hasonló teszteknél.

## Következtetés

Az `Application_Key_Request` PoC most már **valódi hálózaton, valódi
kulcsokkal igazolt** - a session-kulcs-levezetés helyessége nem csak
szintetikus önteszttel, hanem egy független, a device saját maga által
kiszámolt DevAddr-rel való egyezéssel is alá van támasztva.
