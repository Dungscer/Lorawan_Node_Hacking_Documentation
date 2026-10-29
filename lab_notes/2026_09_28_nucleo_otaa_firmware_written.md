# NUCLEO-WL55JC1 otaa_basic/otaa_secure: OTA-fogadó megírva, hardveresen nem tesztelve (2026-09-28)

**Kontextus:** ugyanakkor, amikor a `base_node` (ld.
`2026_09_28_nucleo_base_node_vendored.md`) elkészült, megírtam az
OTA-firmware-eket (`otaa_basic`, `otaa_secure`) is,
explicit tudatában annak, hogy az eszköz most nem tesztelhető - a cél egy
kiinduló állapot, amiről később, valódi hardveren lehet folytatni,
határidőkkel a teszteléshez és a további hardening-hez.

## Mi készült el

- `LoRaWAN/App/sha256.c/.h` - önálló, streaming SHA-256 (FIPS 180-4),
  külső függőség nélkül. Nem az STM32CubeWL-ben amúgy elérhető
  `mbed-crypto` (ST-s mbedTLS-fork) API-ját használja, mert az nincs
  bekötve a projekt `.cproject`-jébe - a bekötés (include path-ok,
  `mbedtls_config.h`) vak, fordítás nélküli módosítása kockázatosabbnak
  tűnt, mint egy jól ismert, ~120 soros algoritmus önálló megírása.
- `LoRaWAN/App/ota_update.c/.h` - a tényleges OTA-fogadó: fPort 10-en
  érkező manifest/data frame-eket parseol (ugyanaz a keretformátum, mint
  a `Tools/OTA_Update/framing.py` v3 protokollja - a host-eszköz csak
  aláírt firmware-t hajlandó küldeni, tehát `otaa_basic` és `otaa_secure`
  ugyanazt a vezetéki formátumot beszéli, csak más mértékben kényszerítik
  ki, amit a manifest hordoz), chunk-onkénti CRC8-at ellenőriz, a képet
  flash-be stage-eli, a teljes kép SHA-256-ját ellenőrzi. `otaa_secure`-nál
  emellett anti-rollback verzió-ellenőrzés is fut (a legutóbb elfogadott
  verzió flash-be perzisztálva).
- Egy hook `lora_app.c` `SMTC_MODEM_EVENT_DOWNDATA` ágába:
  `OTA_Update_ProcessDownlink(...)`, közvetlenül
  `smtc_modem_get_downlink_data()` után.
- A flash-írás a már vendorolt, ST-s `FLASH_IF_Write()`-ot
  (`Core/Src/flash_if.c`) használja - ezt a stack már amúgy is használja
  a saját kontextus-mentéséhez, tesztelt, biztonságos primitívet ad a
  lap-szintű törlés/újraírás/8-byte-igazítás kezelésére, nem kellett
  saját HAL-flash-logikát írni.

## Két explicit, dokumentált, nyitva hagyott hiányosság

1. **Az Ed25519-aláírás nincs ellenőrizve.** A manifest 64 byte-os
   aláírása be van olvasva, de `ota_update.c` csak egy hangos
   `WARNING - signature NOT verified` logot ír, nem ellenőrzi. Emiatt
   `otaa_secure` jelenleg bármilyen manifestet elfogad, aminek a
   verziója magasabb és a SHA-256-ja stimmel - ez integritás- és
   anti-rollback-védelem, **nem hitelesség**. Az ok: az `mbed-crypto`
   nincs bekötve a build-be, és a bekötéshez szükséges API-válogatás
   (PSA `psa_verify_message`/`PSA_ALG_PURE_EDDSA` vs. classic mbedtls,
   ami natívan nem tud Ed25519-et) vakon, fordítás nélkül nem lett volna
   felelős döntés.
2. **Nincs image-aktiváció.** A staging-régióba került, ellenőrzött kép
   nem kerül át a futó programba, nincs újraindítás/bootváltás. A
   STM32WL55 **egybankos flash** (nincs `DBANK` bit - ellenőrizve a
   CMSIS eszközfejléc ellen), tehát az ESP32-s "írj a másik bankba,
   válts" trükk itt nem alkalmazható. Egy RAM-rezidens másoló rutin vagy
   önálló bootloader (ST saját, a publikus repóból kizárt Secure
   Engine/Key Management Services keretrendszere pont ezt oldja meg)
   kellene - ez nem lett vakon megpróbálva.

## Egy harmadik nyitott kérdés: konkrét flash-cím

Az `ota_update.h` három flash-cím placeholder-t `#error`-ral blokkol
(`OTA_STAGING_FLASH_ADDR`, `OTA_STAGING_FLASH_SIZE`,
`OTA_STATE_FLASH_ADDR`) - nem lehetett biztonságos értéket megadni,
mert a LoRaWAN Basics Modem middleware már lefoglal flash-lapokat a saját
kontextusának (`ADDR_FLASH_MODEM_CONTEXT` stb., a `lora_app.c` hivatkozik
rájuk, de a `smtc_modem_utilities.h`-ban vannak definiálva, ami a
`Middlewares/`-ben van - nincs vendorolva ebbe a repóba, ld.
`base_node/README.md`) Build előtt meg kell néznem ezt
a fejlécet vagy a `.map` fájlt, és szabad, lapigazított régiót kell
kijelölnie.

## otaa_vulnerable: tudatosan kihagyva

Nincs sehol referencia-implementáció (a Heltec V2 `otaa_vulnerable` is
csak spec, `.ino` nélkül) - a NUCLEO-verzió megírása most azt jelentette
volna, hogy a vulnerable-tier viselkedését a nulláról, csak ezen a
boardon tervezem meg, ami eltérhetne attól, amit később Heltecre írnak.
Emellett több tulajdonság (FCnt-reset védelem hiánya, RDP-szint)
hardver/option-byte vagy modem-belső beállítás, nem app-kód - ennek
app-kód-oldali kifejezhetősége sem tisztázott. Lásd `otaa_vulnerable/README.md`.
