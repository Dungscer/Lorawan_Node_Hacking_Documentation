# NUCLEO otaa_secure - Ed25519 manifest-aláírás ellenőrzés (2026-10-31)

Az `otaa_secure` tier eddig csak hordozta a manifest 64 bájtos Ed25519-aláírását
(`payload[41..104]`), de nem ellenőrizte - ez volt a dokumentált nyitott lyuk.
Most bekötöttem a tényleges ellenőrzést.

## Mit csináltam
Az `otaa_secure` fát felhoztam a mostani `otaa_basic` OTA-vevő szintjére (STOP2
re-init, nem-blokkoló W25Q-állapotgép, readback-verify), és hozzáadtam az
aláírás-ellenőrzést. A kriptóhoz a **TweetNaCl**-t vendoroltam be (public domain,
egy fájl, a `crypto_sign_open` Ed25519-verify a saját SHA-512-jével; a `randombytes`
csak az aláíró úthoz kell, azt kistubboltam, mert csak verifikálok). A publikus
kulcs (`ota_signing_key.pem` publikus fele) 32 bájtos tömbként a firmware-be
égetve - ez nem titok.

Az Ed25519-verify CPU-nehéz, ezért nem a rádió-downlink-callbackben fut (ugyanaz a
real-time korlát, ami a W25Q-flash-műveleteknél), hanem a main-loopban: a callback
csak eltárolja a manifestet és `OTA_FLASH_VERIFY` fázisba lép, a verify + az
"elfogad/elutasít" döntés a main-loopban történik. Az aláírt üzenet pontosan
`version(BE32) || total_bytes(BE32) || sha256(32)` = 40 bájt, amit a
`sign_firmware.py` is aláír.

## Hardveres mérés (NUCLEO + W25Q64 + ChirpStack, confirmed downlink)
Három forgatókönyv, 250 B-os képpel (a node a `se-identity.h` jó kulcsaival joinol,
DevAddr `01F61664`):

- **Érvényes aláírás (v5):** `manifest signature VERIFIED (Ed25519)` -> staging ->
  `whole-image SHA-256 verified OK (read back from W25Q)`. A push SUCCESS 5/5, 0 retry.
- **Manipulált aláírás (v6, egy bit átbillentve):** `manifest signature INVALID -
  update rejected`, a node `active` marad hamisban, a beérkező chunkokat
  `data frame received with no active manifest, ignored`-dal eldobja - nem töröl,
  nem stage-el. (A push-oldali SUCCESS itt félrevezető: a confirmed downlinkeket a
  MAC-réteg nyugtázza, függetlenül attól, hogy az app elfogadta-e; a mérvadó jel a
  node-log.)
- **Érvényes aláírás, de régebbi verzió (v4, a perzisztált 5 alatt):**
  `rejected, version 4 <= persisted 5 (anti-rollback)` - a verzió-ellenőrzés az
  aláírás-verify elé kerül. Ez egyúttal igazolja, hogy az előző (v5) futás az
  anti-rollback verziót a belső flashbe (`OTA_STATE_FLASH_ADDR`) sikeresen
  elmentette, és vissza is olvasódott.

A node a verify-blokk alatt és után végig adott (nem állt le), tehát a main-loopos,
egyszeri Ed25519-verify belefért a rádió-időzítésbe. A verify pontos idejét nem
mértem.

## Megfigyelés a méréshez
Két `cat /dev/ttyACM0` egyszerre futtatva szétszedi a bájtfolyamot, a log
olvashatatlanul összekeveredik; a tampered-futást emiatt elsőre újra kellett
venni egyetlen olvasóval. Flash után a VCP néha más `ttyACM` számra kerül.

## Korlát
A node-oldali SHA-256 a readback után a W25Q-tartalmon fut (ez igazolt), de az
aláírás csak a manifest `sha256` mezejét köti a képhez; a staged kép és a manifest
`sha256` egyezését a whole-image hash adja, az aláírás pedig e hash hitelességét.
Az aktiváció (a stagelt kép tényleges futtatása) továbbra sincs megírva.
