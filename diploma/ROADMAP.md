# ROADMAP - Diplomamunka kidolgozása

**Hallgató:** Nagy Szabolcs | **Neptun:** CGZNU9 | **Beadási határidő:** 2027-05-15
(ld. `Documentation/abstract/` feladatlap; az általános munkaterv/határidők a
gyökér `ROADMAP.md`-ben vannak).

Az EKÖP 2026 pályázat nem sikerült (2026-09-13: nincs válasz, sikertelennek
tekintjük) - a pályázathoz kötött anyagok (`plan.tex`/`plan.pdf`, EKÖP-fázisokhoz
kötött ütemterv) törölve. A `research.tex`/`research.pdf` draft (eredetileg TDK
célra indult) 2026-09-21-én átszerkesztve, mostantól ez maga a **diplomamunka**
szövege - fájlnevek `diploma.tex`/`diploma.pdf`-re átnevezve, címlap/fejléc
"Diplomamunka"-ra igazítva. Külön TDK-dolgozat nem készül.

---

## Fejezetek státusza

A dolgozat vázlata (Bevezetés, 2. LoRaWAN protokoll és biztonsági modell,
3. Tesztkörnyezet, 4--7. támadási forgatókönyvek, 8. Védelem, 9. Összefoglalás)
készen van, tartalmilag a demonstrált PoC-okkal párhuzamosan bővül
(ld. gyökér `ROADMAP.md` 1. sáv).

| Fejezet | Státusz |
|---------|---------|
| 2.1 Protokoll-architektúra | 🔄 draft kész |
| 2.2 Biztonsági modell és kulcsarchitektúra | 🔄 draft kész; 2026-10-06-án ábrával kiegészítve (OTAA Join-szekvenciadiagram, a két megfigyelési ponttal annotálva) |
| 2.3 Ismert sérülékenységek a szakirodalomban | ✅ kibővítve 2026-09-21-én (replay, bit-flipping/csonkított MIC, ACK spoofing, RF jamming, ABP/forward secrecy, fizikai rétegbeli ujjlenyomat-vétel mint ellenintézkedés, LoRaWAN 1.1 újítások) |
| 3.2 Szoftver-stack | ✅ 2026-09-21-én kiegészítve valós docker-compose service-táblával + minimál jogosultságú hozzáférés megjegyzéssel (`lab_notes/2026_09_13_chirpstack_server_environment.md`); 2026-10-06-án a 3.2 táblázat kilógó sora javítva (p{}-oszlopok + \texttt kötőjel-törés fix) |
| 3.3 A tesztkörnyezet elrendezése | 🔄 draft kész; 2026-10-06-án ábrával kiegészítve (architektúra-diagram a két passzív támadói ponttal) |
| 5. Fizikai SPI-busz tap (Eredmények) | ✅ 2026-09-21-én kiegészítve valós V2 capture-mérőszámokkal; az SX1262/V3/V4 port explicit kihagyva és megjegyezve, mert csak szintetikus adaton tesztelt, hardveren nem igazolt (`lab_notes/2026_09_19_spi_key_extraction_finalized.md`); 2026-10-06-án ábrával kiegészítve (SPI pin-kiosztás + logikai analizátor tap); ugyanaznap új 5.5 alfejezettel bővítve -- a Heltec (olcsó, külső rádiócsip) vs.\ NUCLEO-WL55JC1 (drágább, integrált STM32WL rádió) összevetés mint architekturális korlát a SPI-tap ellen, mérés nélkül, a meglévő tényekből levezetve |
| 7. Firmware/rogue node (új 7.5 alfejezet) | ✅ 2026-09-21-én kiegészítve a valódi hálózati `Application_Key_Request` teszttel (bit-pontos DevAddr-egyezés, payload round-trip) (`lab_notes/2026_09_19_application_key_request_real_test.md`) |
| 8.5 OTA-frissítési csatorna hardeningje (új alfejezet) | ✅ 2026-09-21-én felvéve - aláírt manifeszt/anti-rollback/CRC/SHA-256, munkamenet-izoláció, gRPC TLS + minimál-scope API-kulcs, mind hardveren/hálózaton igazolva; a mért unconfirmed-downlink hamis-pozitív korlát átkerült a 9. fejezet jövőbeli irányaiba (`lab_notes/2026_09_09_ota_v3_hardware_test.md`, `2026_09_13_ota_tls_nginx_setup.md`, `2026_09_19_api_token_minimal_scope.md`) |
| 8. Megerősített fw-konfig / debug-port RDP (új alfejezet) | ✅ 2026-10-02-én felvéve - hardveres RDP-demó a STM32WL55JC1-en: RDP0-n a flash teljesen kiolvasható, RDP1-en a flash/RAM/regiszter kiolvasás tiltott (`0xFFFFFFFF`) a node működése mellett, regresszió = mass-erase; a 7. fejezet firmware-kinyerése egy provisioning-lépéssel kivédhető (`lab_notes/2026_10_02_stm32wl_rdp_hardening_demo.md`) |
| 9. Jövőbeli irányok | ✅ 2026-10-02-én kiegészítve a mért STM32WL OTA-megvalósíthatósági korláttal (256K egybankos flash -> külső flash / SBSFU kell) (`lab_notes/2026_10_02_nucleo_base_node_built.md`, `brainstorms/2026_10_02_nucleo_ota_external_flash/`); 2026-10-06-án új "Kiegészítő eredmények" alfejezettel bővítve (codec DoS negatív, codec-injekció/XSS, kulcs nélküli RF-outsider sweep) (`lab_notes/2026_10_04_chirpstack_jscodec_chain.md`, `2026_10_05_jscodec_sink_xss.md`, `2026_10_05_fuzzer_keyless_outsider.md`) |
| 6. AppKey brute-force | ✅ 2026-10-06-án új alfejezettel bővítve: a teljes lánc (RF-sniff -> offline crack -> munkamenetkulcs-levezetés -> forgalom-visszafejtés) hardveren + valós kriptográfiával igazolva, negatív kontrollal (erős kulcs ellen a crack elbukik) (`lab_notes/2026_10_05_weak_appkey_otaa.md`); ugyanaznap ábrával kiegészítve (brute-force időigény oszlopdiagram, log-skála) |
| 8. Védelmi szint összefoglalása | ✅ 2026-10-06-án ábrával kiegészítve (a védelmi mátrix vizuális, szürkeárnyalatos heatmap verziója); 2026-10-07-én valódi szakirodalmi hivatkozásokkal is bővült (ESP32 flash-titkosítás/secure boot CVE-k, STM32/nRF52 glitching) -- előtte egyetlen hivatkozás sem volt a fejezetben |
| 1, 4. fejezet | 🔄 még nincs friss lab_notes-tartalommal bővítve |

Terjedelem: **30 oldal** (2026-10-07, `pdflatex`+`bibtex`-lánccal
ellenőrizve, hibátlan; 26 oldalról +1 oldal a 6.\ fejezet RF-outsider
alfejezetétől és a 9.\ fejezet "Kiegészítő eredmények" alfejezetétől,
majd +2 oldal mind az öt tervezett ábrától: tesztkörnyezet-architektúra
(3.3), OTAA Join-szekvenciadiagram (2.2), SPI pin-kiosztás diagram
(5.\ fejezet), brute-force időigény oszlopdiagram (6.\ fejezet), védelmi
mátrix heatmap (8.\ fejezet) -- minden ábra TikZ/pgfplots-ban rajzolva, a
dokumentumban meglévő táblázat-adatokból, nincs hozzá hardveres
fénykép/screenshot). **2026-10-07: hivatkozásjegyzék kibővítve** a
felhasználó Zotero-exportjából (saját könyvtár + csoportkönyvtár, RDF) --
a dolgozat áttért kézi `\bibitem`-ről `natbib`+BibTeX-re
(`manual_refs.bib` + `zotero_refs.bib`), a 9 régi hivatkozás változatlanul
megmaradt, és 25 új, ellenőrzött tétel lett ténylegesen a szövegbe szőve
(nem csak a listába bedobva) -- összesen 34 hivatkozás, +4 oldal.
**2026-10-08:** a konzulens kérdésére (van-e elméleti ajánlás az OTAA
node-tervezéshez) a 2017-es `oniga2017` mellé bekerült két aktuális IETF
szabvány, **RFC 9019 + RFC 9124 (SUIT firmware-update architektúra)**, a
8.6 "OTA-frissítési csatorna hardeningje" alfejezet elejére, új
"Tervezési alap" bekezdésként -- 36 hivatkozás, 31 oldal.

## Fájlok státusza

| Fájl | Státusz |
|------|---------|
| `diploma.tex` / `diploma.pdf` | 🔄 fejlesztés alatt (korábban `research.tex`/`research.pdf`) |
| `plan.tex` / `plan.pdf` | ❌ törölve (2026-09-13) - EKÖP-specifikus anyag, a pályázat nem sikerült |
