# Kapcsolódó munkák - külső SPI flash OTA-staging fizikai támadhatósága

Azt kerestem, foglalkozott-e már valaki azzal, amit a NUCLEO + W25Q64 felállásban
csinálok: OTA-image stagelése külső SPI NOR flashre, majd bootloaderes átmásolás a
belső flashbe, és ennek fizikai (buszszintű) támadhatósága. A keresés 2026-11-04-én
készült, webes kereséssel; a forrásokat egy kivételével (CVE-2024-28183) csak a
keresési összefoglalók alapján ismerem, a teljes szöveget még nem olvastam el.

## Röviden

- A három elem külön-külön jól dokumentált: LoRaWAN FUOTA külső flash-sel
  (ST AN5554, több diplomamunka), SPI-flash lehallgatás logikai analizátorral
  (gyakorlati write-upok, eszközök), és a check-then-use (TOCTOU) hiba
  bootloaderben külső flash esetén (ESP-IDF CVE).
- Olyan munkát, amely ezt a hármat együtt, egy LoRaWAN node-on, valódi hardveren
  méri (staging-lehallgatás + a bootloader "másol, aztán ellenőriz" sorrendjéből
  adódó boot-loop), nem találtam. Ez az a rés, amit a 8. forgatókönyv lefed.
- Magyar nyelvű diplomamunkát vagy szakdolgozatot a témában nem találtam; a BME,
  ELTE, ÓE tárakban célzottan még nem kerestem.

## 1. LoRaWAN FUOTA külső flash-sel (diplomamunkák, gyártói anyag)

- **ST AN5554** - LoRaWAN FUOTA STM32CubeWL-lel. DualCore megoldás külső flash
  download slottal és SBSFU (Secure Boot / Secure Firmware Update) komponenssel.
  A legközelebbi gyártói referencia az én felállásomhoz (ugyanaz a chipcsalád).
  <https://www.st.com/resource/en/application_note/an5554-lorawan-firmware-update-over-the-air-with-stm32cubewl-stmicroelectronics.pdf>
- **Corsin Obrist, HSLU (2022), MSE master's thesis** - LoRaWAN-on több node-ra
  egyszerre küldött firmware-patchek, saját bootloader fésüli össze a régi
  firmware-rel; teljes lánc (FUOTA szerver, network server, gateway, node).
  <https://sites.hslu.ch/ta-bachelorarbeiten/wp-content/uploads/sites/6/2022/07/MAT_MSE_2022_Obrist_Corsin.pdf>
- **Mohamed Ahmed, Turku (2019), master's thesis** - ipari LoRaWAN node
  firmware-fejlesztése, OTA-konfiguráció és micro-update séma, FUOTA
  energiaköltség-becslés.
  <https://utupub.fi/bitstream/handle/10024/148375/Mohamed_Ahmed_Thesis.pdf?sequence=1>
- **Hess, UCLouvain (2020)** - ultra-low-power OTA frissítés biztonságos LoRaWAN
  hálózatban. A rekordot csak a repozitórium teszt-példányán találtam, idézés előtt
  a végleges DIAL.mem oldalon ellenőrizni kell.
  <https://dial-mem.test.bib.ucl.ac.be/handle/2078.2/16747>
- **Mbed OS fórum (2017), "Towards FOTA over LoRa"** - a korai Mbed FUOTA biztonsági
  hiányai: a bootloader nem ellenőrzött aláírást, a verifikáló és az alkalmazás kód
  közös memóriát használt; a külső SPI flash BlockDevice API-n át.
  <https://os.mbed.com/forum/blog-towards-fota-lora-crypto-delta-updates-458-forum/post/53315>

Ezek a FUOTA működésére és energiaigényére koncentrálnak, a stagelt kép fizikai
expozíciójával a keresési összefoglalók alapján nem foglalkoznak.

## 2. Bootloader check-then-use hiba külső flash esetén

- **CVE-2024-28183 (ESP-IDF bootloader), Joseph Surin / elttam** - ezt a write-upot
  elolvastam. A bootloader az anti-rollback ellenőrzést az image egy olvasásán
  végzi, de a betöltéskor újraolvassa a flasht; ha a támadó a két olvasás között
  más tartalmat ad vissza, egy régebbi (verzió 0) app bootol, holott az eFuse-ban
  10-es minimum verzió van. A PoC emulált (QEMU + egy flash-t kiszolgáló `nbdkit`
  plugin, amely a 6. olvasásra más képet ad), valódi hardveres interposert nem mutat.
  <https://github.com/elttam/publications/blob/master/writeups/CVE-2024-28183/CVE-2024-28183.md>

Kapcsolat a saját esetemmel: ugyanaz a hibaosztály (az ellenőrzés és a felhasználás
külön olvasáson alapul), de a következmény más. Az ESP-IDF-nél integritás-sértés
(régi kép fut), nálam a várt SHA-256 a belső flash flag-lapján van, a bootloader a
*belsőbe már átmásolt* tartalmat ellenőrzi, így idegen kód nem fut - viszont mivel
előbb töröl és másol, csak utána ellenőriz, a manipulált W25Q-kép végtelen
boot-loopot okoz (rendelkezésre állás sérül). Ezt a kódban (`ota_bootloader/main_boot.c`
`install_image()`) olvastam ki, hardveren még nem reprodukáltam.

## 3. Biztonságos frissítési sémák, amelyek a külső flasht kezelik

- **MCUboot - encrypted images** - a titkosított image kifejezetten arra szolgál, hogy
  a külső SPI flashen (secondary slot) lévő firmware bizalmas maradjon; a belső flash
  JTAG-es kiolvasása ellen nem véd. Ha a swap scratch-terület külső flashen van, a
  `MCUBOOT_SWAP_SAVE_ENCTLV` kell, különben egy megszakított frissítésnél a
  dekódoló kulcs nyíltan kiolvasható a külső flashből. A hash és az aláírás a
  titkosítatlan adaton van, ellenőrzés a visszafejtés után.
  <https://docs.mcuboot.com/encrypted_images.html>
- **Fan et al., "A Secure IoT Firmware Update Scheme Against SCPA and DoS Attacks",
  JCST 2021** (DOI 10.1007/s11390-020-9831-8) - AES-CCM alapú frissítés, a
  bootloader korlátozza a frissítések gyakoriságát (teljesítmény-oldalcsatorna
  ellen), DoS elleni kézfogás a szerverrel.
  <https://link.springer.com/content/pdf/10.1007%2Fs11390-020-9831-8.pdf>
- **Secure firmware update MCU-alapú IoT-hoz (NSF PAR)** - secure bootloader
  integritás-ellenőrzéssel és visszafejtéssel; UART-on és OTA-n keresztüli
  támadások PurpleAir eszközökön.
  <https://par.nsf.gov//servlets/purl/10149711>
- **ITEA projekt D2.5b, Secure Firmware Updates** - fenyegetés-katalógus, benne a
  "Brick Device" mint DoS-kategória; ez pontosan az én boot-loop esetem besorolása.
  <https://itea4.org/index.php/project/workpackage/document/download/8064/D2.5b%20-%20Secure%20Firmware%20Updates%20-%20M36.pdf>

## 4. SPI-flash lehallgatás és firmware-kinyerés (gyakorlati anyagok)

Tudományos cikket erre nem találtam, gyakorlati write-upok és eszközök vannak:

- **Matthew Alt (wrongbaud), Hackaday-cikk 2026-09** - W25Q-sorozatú flash boot
  közbeni SPI-olvasásainak rögzítése, Scapy-vel dekódolva, binwalk-kal elemezve.
  Ugyanaz a chipcsalád, mint nálam.
  <https://hackaday.com/2026/09/24/reconstructing-device-firmware-from-spi-reads/>
- **sniffROM** - Saleae-exportból passzívan rögzített SPI/I2C parancsokból
  rekonstruálja a flash tartalmát (Logic 2 exporttal 2023 végén ismert gond volt).
  <https://github.com/alainiamburg/sniffROM/wiki>
- **Nozomi Networks** - firmware kinyerése OT eszközökből, a flash lábainak
  azonosítása logikai analizátorral.
  <https://nozominetworks.com/blog/extract-firmware-from-ot-devices-for-vulnerability-research>
- **Payatu IoT security note** - SPI-forgalom lehallgatása; aki a flasht olvasni és
  írni tudja, backdoort patchelhet be, ha a firmware nincs biztonságosan kezelve.
  <https://payatu.com/wp-content/uploads/2022/12/c15.pdf>
- **ECSC "SPI capture" CTF write-up** - Winbond flash PulseView-felvételének
  dekódolása zavarjelek javításával.
  <https://challenges.ecsc.eu/challenges/spi-capture/edit/++widget++form.widgets.write_ups/@@download/0>

## 5. Általános LoRaWAN-biztonsági háttér

- LoRaWAN fizikai rétegű támadások áttekintése (Sensors, 2022, CC BY 4.0):
  <https://www.ncbi.nlm.nih.gov/pmc/articles/PMC9100101/>
- Szelektív jamming LoRaWAN ellen (arXiv 1712.02141):
  <https://arxiv.org/pdf/1712.02141.pdf>

Ezek a node rádiós oldalát vizsgálják, a node belső felépítését (külső flash, busz)
nem.

## Korlátok

- A források többségénél csak a keresési összefoglalót láttam; idézés előtt a
  teljes szöveget el kell olvasni, különösen az Obrist- és a Hess-dolgozatnál, mert
  ott lehet a stagelés biztonságáról szóló rész, amit az összefoglaló nem említett.
- Tudományos adatbázisokban (IEEE Xplore, ACM DL, Google Scholar) és magyar
  egyetemi tárakban célzott keresést még nem végeztem.
