# Referencia-követelmények: hogyan kellene megépíteni egy biztonságos, külső flash-sel frissülő node-ot

A `related_work.md` azt gyűjti, ki támadott már hasonló rendszert. Ez a fájl a másik
irány: kik írták le, *hogyan kellene* egy ilyen node-ot megvalósítani. Ezekhez mérem
a saját NUCLEO-implementációmat (`otaa_secure` + `ota_bootloader` + W25Q64 staging),
és ezekkel a pontokkal lehet indokolni, mi hiányzik még belőle, illetve mi javítható.
A forrásokat 2026-11-04-én gyűjtöttem; amit csak keresési összefoglalóból ismerek,
azt jelöltem.

## Források

1. **IETF RFC 9019** (SUIT architektúra) és **RFC 9124** (SUIT információs modell,
   fenyegetés- és követelménylista). A legjobban hivatkozható, gyártófüggetlen
   szabvány; az RFC 9124 fenyegetés- és követelmény-azonosítóit (THREAT.*, REQ.SEC.*)
   a szövegből olvastam ki. Fontos korlát: a 4.1 szakasz szerint a fizikai hozzáférésből
   eredő fenyegetések **nincsenek a hatókörében** - pont ezt a rést vizsgálom.
   <https://www.rfc-editor.org/rfc/rfc9124.html>
2. **MCUboot design** - a legelterjedtebb nyílt bootloader leírása; a frissítési
   folyamat ellenőrzési sorrendjét a dokumentációból olvastam ki.
   <https://docs.mcuboot.com/design.html>,
   titkosított image-ek: <https://docs.mcuboot.com/encrypted_images.html>
3. **ST SBSFU az STM32CubeWL-hez** - a gyártó saját referencia-megoldása pont erre a
   chipcsaládra: AN5544 (integrációs útmutató, KMS-sel) és UM2767 (getting started).
   A download slot "firmware header + titkosított firmware"; a védelmi elemek (RDP,
   WRP, PCROP, MPU, firewall, tamper) leírása ezekben van. Csak összefoglalóból
   ismerem, a teljes dokumentumot még el kell olvasnom.
   <https://www.st.com/resource/en/application_note/dm00725183-integration-guide-of-sbsfu-on-stm32cubewl-including-kms-stmicroelectronics.pdf>,
   <https://www.st.com/resource/en/user_manual/dm00731353-getting-started-with-the-sbsfu-of-stm32cubewl-stmicroelectronics.pdf>
4. **OWASP ISVS** (IoT Security Verification Standard) - ellenőrizhető
   követelménylista: V3.4.3 (aláírt, futtatás előtt ellenőrzött frissítés), V3.1.4 /
   V5.1.2 (secure boot), V5 hardverplatform (debug-interfészek letiltása, tamper,
   OTP). Összefoglalóból ismerem; a külső tárolón lévő firmware titkosítására
   vonatkozó pontos azonosítót még nem találtam meg.
   <https://owasp-isvs.gitbook.io/owasp-isvs-pr/security-requirements/v5-hardware_platform_requirements>,
   <https://owasp.org/owasp-istg/03_test_cases/firmware/firmware_update_mechanism.html>
5. **NIST SP 800-193** (Platform Firmware Resiliency) - védelem / észlelés /
   helyreállítás hármas, "golden image" a helyreállításhoz, a jogosulatlan
   módosítást futtatás *előtt* kell észlelni. Eredetileg PC/szerver platformra
   készült, csak másodlagos (gyártói) forrásokból ismerem.
6. **LoRa Alliance FUOTA** (TS003-TS007). A fragmentációs specifikációban van egy
   integritás-/hitelesítési ajánlás-szakasz, de a working group szerint a firmware-t
   ellenőrző és telepítő ágens **nincs specifikálva**, a gyártóra van bízva - vagyis a
   LoRaWAN szabvány a node-oldali tárolás biztonságáról nem mond semmit.
   <https://lora-developers.semtech.com/library/tech-papers-and-guides/firmware-updates-over-the-air/>
7. **Zandberg et al., IEEE Access 2019** - SUIT-alapú secure update konkrét
   mikrokontrollereken; kimutatja, hogy < 32 kB RAM / 128 kB flash is elég hozzá.
   <https://core.ac.uk/download/266912339.pdf>

## Követelmények kontra a mostani implementáció

A "mostani" oszlopot a kódból és a korábbi mérésekből töltöttem ki; ahol nem néztem
utána, ott "ellenőrizendő" szerepel. Ez kódolvasás, nem hardveres teszt.

| # | Követelmény | Forrás | Mostani állapot |
|---|---|---|---|
| 1 | A kép hitelesítése (aláírás + digest) telepítés előtt | RFC 9124 REQ.SEC.AUTHENTIC, ISVS V3.4.3 | Részben. Az `otaa_secure` app Ed25519-cel ellenőriz staging előtt (hardveren igazolva). A bootloader aláírást nem, csak SHA-256-ot ellenőriz. |
| 2 | Anti-rollback (monoton verziószám) | RFC 9124 REQ.SEC.SEQUENCE | Igen, az appban (hardveren igazolva). A bootloader nem ellenőrzi újra. |
| 3 | A kép az ellenőrzés és a felhasználás között nem változhat (TOCTOU) | RFC 9124 THREAT.IMG.REPLACE, REQ.SEC.MFST.CONST | Nem. Az app a beérkező / visszaolvasott képet ellenőrzi, a bootloader később újraolvassa a W25Q-t. A belső flag-lapon tárolt hash miatt idegen kód nem fut, de a manipulált kép boot-loopot okoz. |
| 4 | A tartalék kép ellenőrzése közvetlenül a másolás előtt; hibás kép eldobása, az aktív kép megtartása | MCUboot design | Nem. A bootloader előbb töröl és másol, utána ellenőriz; hibánál resetel és újrapróbálja. Ez a javítás közvetlen mintája. |
| 5 | Futtatás előtti ellenőrzés minden bootkor (méret + digest) | RFC 9124 REQ.SEC.AUTH.EXEC, MCUboot `MCUBOOT_VALIDATE_PRIMARY_SLOT`, NIST 800-193 | Nem. Flag nélkül a bootloader ellenőrzés nélkül ugrik az appra. |
| 6 | A külső tárolón lévő kép titkosítása | RFC 9124 REQ.SEC.IMG.CONFIDENTIALITY, MCUboot encrypted images, ST SBSFU download slot | Nem. A W25Q-n a kép nyíltan van; a logikai analizátoros méréssel ezt akarom demonstrálni. |
| 7 | A dekódoló kulcs ne kerülhessen nyíltan a külső flashre | MCUboot `MCUBOOT_SWAP_SAVE_ENCTLV` | Nem releváns, amíg nincs titkosítás. |
| 8 | Eszköz-/osztály-kompatibilitás hitelesítése | RFC 9124 REQ.SEC.COMPATIBLE | Nem. Az aláírt rész csak `version` + `total_bytes` + `sha256` (`Tools/OTA_Update/framing.py`, `sign_firmware.py`), eszköz-/osztályazonosító nincs benne, így egy másik eszköznek szánt, ugyanazzal a kulccsal aláírt kép is elfogadható. A méret viszont aláírt. |
| 9 | Belső flash kiolvasás elleni védelem (RDP), debug letiltása | ST SBSFU, ISVS V5 | Nem. Az `st-flash --connect-under-reset` olvassa a belső flasht, RDP-t nem állítottam be. |
| 10 | A bootloader írásvédelme (WRP), elszigetelés (MPU / firewall) | ST SBSFU | Ellenőrizendő; nem tudok róla, hogy be lenne állítva. |
| 11 | Helyreállítás: tápkimaradás-biztos telepítés | NIST 800-193 recovery | Igen, hardveren igazolva (2026-11-01, elrontott app-régió -> újratelepítés). |
| 12 | Helyreállítás: ismert jó ("golden") kép, ha a frissítés hibás | NIST 800-193 | Nem. Nincs megtartott régi kép, ezért hibás staging esetén nincs mire visszaállni (ez is a boot-loop oka). |
| 13 | Fizikai hozzáférés kezelése | RFC 9124 4.1: hatókörön kívül; ISVS V5: tamper, debug | Nem foglalkozik vele a szabvány; ez a saját mérésem tárgya. |

## Mit jelent ez a tesztelésre

- A 3., 4., 6. és 12. pont a fizikai (SPI-busz) támadással közvetlenül mérhető. Mind
  a négyre van megnevezett követelmény egy szabványban vagy referencia-bootloaderben,
  így a hiányukat nem a saját véleményemre alapozom.
- A legerősebb állítás az lehet, hogy az RFC 9124 a fizikai fenyegetést kifejezetten
  kizárja, a LoRaWAN FUOTA a node-oldali ellenőrzést a gyártóra bízza, a külső flash
  viszont pont fizikailag elérhető. A mérésem ezt a rést mutatja meg egy konkrét
  node-on.
- Az ST SBSFU a gyártó saját válasza ugyanerre a chipre (titkosított download slot +
  hardveres védelmek). Ezzel a legkézenfekvőbb összevetni a saját megoldásomat; ehhez
  az AN5544 és az UM2767 teljes szövegét el kell olvasnom.
