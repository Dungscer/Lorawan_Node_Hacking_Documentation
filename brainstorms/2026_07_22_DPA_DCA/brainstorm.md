# STM32 side-channel (DPA/CPA) + glitching kutatási irány - brainstorm (2026-07-22)

**Státusz:** ötletelési fázis, semmi nincs megvalósítva. **Eldöntött (nem csak
"talán"): sem a glitching+SCA (ChipWhisperer-Husky), sem az EMFI+valós
deployment irány NEM kerül be a szakdolgozatba** - a felhasználó explicit
döntése, mindkettő túl sok lenne bele. Mindkettő élő, egyetem utáni
kutatási/publikációs érdeklődés marad (ld. publikálási utak lentebb). Ne
feltételezd más session-ben, hogy ez a szakdolgozat része, és ne javasold
visszaemelni a thesis scope-jába, hacsak a felhasználó explicit nem kéri.

## A téma

STM32-alapú, "rendesen" hardeningelt (RDP2 + PCROP + Secure Boot) LoRaWAN node
elleni két konkrét technika:

1. **Voltage/clock glitching (fault injection)** - az RDP-ellenőrzés
   megkerülése bootoláskor. RDP1->0 downgrade jól dokumentált, RDP2 ellen
   nehezebb, aktív kutatási terület.
2. **Side-channel kulcskinyerés (SPA -> DPA -> CPA)** - a kriptó-műveletek
   (AES join/session számítás) áramfelvétel/EM-kisugárzás alapú megfigyelése,
   RDP-szinttől függetlenül.

Fogalmak tisztázva (ld. teljes magyarázat a beszélgetésben):
- **SPA**: egyetlen/néhány trace vizuális elemzése, durva leakage esetén.
- **DPA**: statisztikai, sok ezer trace, bit-particionálásos átlagkülönbség.
- **CPA**: DPA finomítása, Hamming-súly modell + Pearson-korreláció, kevesebb
  trace kell hozzá - ez a ChipWhisperer alap AES-törő módszere.

## Eszközhelyzet

Jelenleg csak egy olcsó digitális szkóp + 16 csatornás logic analyzer van
(nincs labor, nincs ChipWhisperer, nincs decap-eszköz).

- **PicoGlitcher** (nyílt forrású, Raspberry Pi Pico-alapú, `findus` Python
  lib, ~30 EUR, Tindie-n rendelhető "Faulty Hardware"-től) - szóba került mint
  olcsó köztes lépés a glitching irányhoz, de **elvetve**, mert csak glitchel,
  DPA/CPA-t nem tud, és pár hónapon belül várható keret (pl. EKÖP) - akkor
  egyből egy teljes körű eszközbe érdemesebb fektetni, nem kettőbe egymás után.
- **ChipWhisperer-Husky** (~208 000 Ft) - glitchel ÉS DPA/CPA capture-t is
  tud, FPGA-alapú szinkronizált ADC. Ezt kellene beszerezni, ha nyílik keret.
  A **Nano** (~50 USD) elvetve, mert gyenge/jitteres glitch, és nincs rendes
  csatlakozója külső target (a saját STM32 node) csatlakoztatásához - csak a
  beépített demo targetre jó, tanulásra.
- Beszerzési csatorna: Mouser (van magyar oldal, `hu.mouser.com`, HU-ba
  szállít) vagy DigiKey - hivatalos forgalmazók, nem AliExpress (niche eszköz).

**Döntés (frissítve 2026-09-13):** az EKÖP-pályázat nem sikerült, tehát az
erre várt keret nem valósul meg - az eszközbeszerzés (Husky/PicoGlitcher)
finanszírozás nélkül marad, nyitott kérdés, nincs hozzá dátum. Lásd
`ROADMAP.md` STM32WL szakaszát: időközben egy tényleges STM32WL eszköz
elérhetővé vált, és egy egyszerűbb, ingyenes szoftveres debug-bypass már
önmagában is használható eredményt adott - ez enyhíti (de nem helyettesíti)
a DPA/CPA/glitching irány hiányát.

## Kapcsolódó publikált kutatás (irodalmi háttér)

**"Evaluation of Side-Channel Key-Recovery Attacks on LoRaWAN End-Device"**
(Springer, kb. 2020) - DOI: `10.1007/978-3-030-49443-8_4`

- Springer: https://link.springer.com/chapter/10.1007/978-3-030-49443-8_4
- ResearchGate (esetleg ingyenes preprint): https://www.researchgate.net/publication/342497465_Evaluation_of_Side-Channel_Key-Recovery_Attacks_on_LoRaWAN_End-Device

Eredményük: CPA, EM-leakage trace-ekkel, valódi LoRaWAN végponti eszközön.
260 trace -> teljes 16 bájtos payload-titkosító kulcs; 140 trace -> MIC-kulcs
12/16 bájtja. **Teljesen visszaállították az AppKey-t (v1.0) / NwkKey-t
(v1.1)** root kulcsot.

**TODO: teljes szöveg elolvasása + Zoteróba felvétel** a fenti DOI-val
("Add Item by Identifier" - automatikusan kitölti a metaadatokat). Ki kell
deríteni belőle: pontosan milyen chipen, milyen védelmi konfigurációval,
milyen mérési elrendezéssel (EM probe vs. áram-alapú) dolgoztak - ez alapján
finomítható, mi a tényleges rés a saját munkához képest.

Ugyanennek a kutatócsoportnak van egy korábbi, **szabadon elérhető** elődcikke
is: **"Experiment on Side-Channel Key-Recovery using a Real LPWA End-device"**
(2019, SciTePress) - ingyenes PDF: https://www.scitepress.org/Papers/2019/72595/72595.pdf

### A glitching technika NEM újdonság - csak a LoRaWAN-specifikus alkalmazás az

Utánanézve: a voltage/clock glitching + STM32 RDP bypass **önmagában már
alaposan publikált, generikus MCU-biztonsági téma**, nem LoRaWAN-specifikus:
- "Shaping the Glitch: Optimizing Voltage Fault Injection Attacks" (TCHES,
  peer-reviewed)
- chip.fail
- SEC Consult "SECGlitcher" blogsorozat
- Anvil Secure, SySS (STM32L05) blogposztok

Egyik sem LoRaWAN-kontextusban dolgozik - mind generikus "MCU biztonság"
keretben. **Tehát a nyers glitching/RDP-bypass technika saját magában nem
számít újdonságnak** - ez egy jól bejáratott, sokszor reprodukált módszer.

**Ami emiatt a valódi, megmaradó újdonság**: nem a technika, hanem **a
LoRaWAN-specifikus alkalmazás és a teljes lánc** - vagyis nem az, hogy
"megkerülöm az STM32 RDP-jét", hanem hogy **"megkerülöm egy valódi LoRaWAN
node RDP-jét, kinyerem belőle az AppKey-t, és ezzel ténylegesen
visszafejtem/hamisítom a forgalmat a saját ChirpStack hálózaton"** - ezt a
konkrét, LoRaWAN-re alkalmazott, végpontig menő kombinációt eddigi keresés
alapján még nem publikálta senki. Ez erősíti (nem gyengíti) az alábbi 3.
pontot ("teljes támadási lánc") - de egyértelművé teszi, hogy **a hangsúlyt
kizárólag a láncra és a LoRaWAN-kontextusra kell helyezni, nem a glitching
technika újdonságára**.

## Mivel lenne más/több a saját munka (a fenti cikkhez képest)

1. **Hardeningelt eszköz ellen**, nem feltételezett alapállapot ellen - ha a
   hivatkozott munka nem RDP2/PCROP/Secure-Boot-os eszközön dolgozott, a saját
   hozzáadott érték: a side-channel *akkor is* működik, ha minden hagyományos
   védelem be van kapcsolva.
2. **Multi-MCU összevetés** (ESP32/STM32/nRF52), nem egyetlen chip - illeszkedik
   a dolgozat már meglévő flash-dump fejezetének multi-MCU szerkezetéhez.
3. **Teljes támadási lánc**, nem izolált kriptó-demo: kulcs kinyerése -> a
   saját eszközökkel (`Application_Key_Breach`, session-kulcs levezetés,
   esetleg `Tools/OTA_Update` forgalom visszafejtése) gyakorlati kihasználás
   a saját ChirpStack/gateway testbeden.
4. **Glitching vs. side-channel összehasonlítás** ugyanazon a hardeningelt
   eszközön - melyik mikor/milyen feltétellel működik jobban.

**Fontos korrekció:** az "alacsony erőforrású, terepi támadó is meg tudja
csinálni" narratíva **nem tartható**, ha a drága (~208k Ft) Husky-t
használod - egy valódi terepi támadónak sem lenne ilyen. Emiatt ez a fejezet
külön, **"jól felszerelt/célzott ellenfél" fenyegetési modellt** igényel,
szemben a többi (SPI tap, firmware dump, AppKey brute-force) fejezettel, ahol
tényleg az "bárki egy logic analyzerrel" a reális attacker-profil. -> a
dolgozat Fenyegetési modell szakaszában két külön attacker-profilt érdemes
definiálni, és fejezetenként jelölni, melyik vonatkozik rá.

## Ha ez nem kerül be a szakdolgozatba: publikálási utak

Egyetemi affiliáció nélkül, magánszemélyként is publikálható infosec kutatás,
ez bevett gyakorlat a szakmában:

- **Saját blog + GitHub writeup** - leggyorsabb, azonnali út.
- **Konferencia CFP-k, affiliáció nem kell**: Hacktivity (Budapest),
  DEF CON/Black Hat (Demo Labs vagy talk track), hardwear.io (kifejezetten
  hardver/IoT biztonság - jó illeszkedés SPI-tap/glitching témákhoz),
  USENIX WOOT.
- **arXiv preprint** (cs.CR) - nem kell affiliáció, citálható/dátumozott.
- **Responsible disclosure / CVE** - ha konkrét, valós terméket (pl. egy
  konkrét ChirpStack-verziót vagy gateway firmware-t) érint.
- **Munkahelyen keresztül** (Silent Signal) - cégek gyakran publikálnak saját
  kutatóblogot/konferencia-előadást a munkatársak nevében.
- **Témavezetőkkel (Balázsné Dr. Kail Eszter, Bringye Zsolt) fenntartott
  kapcsolat, hivatalos hallgatói jogviszony nélkül is**: társszerzőségben az
  ő egyetemi affiliációjuk megmarad a publikáción -> ez nyitja meg az
  akadémiai folyóirat/konferencia utat, ami a tisztán magánszemélyes útnál
  nagyobb presztízsű. Emellett esetleg folytatódhat egyetemi
  eszközhozzáférés, vagy akár doktori témává nőhet a kutatás.
  **TDK-jogosultság hivatalos hallgatói jogviszony nélkül tisztázandó** a
  témavezetővel/TDK-tanáccsal - nem egyértelmű, van-e átmeneti szabály.

## Új fizikai támadási irányok - "mit nem próbált még senki" brainstorm

**Fontos: ezek KIFEJEZETTEN NEM kerülnek be a szakdolgozatba** - a felhasználó
explicit döntése, hogy ez már "sok lenne" a diplomamunkába. Ez az ág marad a
tiszta kutatási/post-graduation brainstorm szinten, ld. fenti publikálási
utak.

Számba vett irányok, miután tisztáztuk, hogy a nyers glitching/SCA technikák
önmagukban nem újdonságok (ld. fentebb):

1. **Akusztikus kriptoanalízis (acoustic side-channel)** - **ELVETVE**, a
   felhasználó nem érdeklődik iránta ("hallottam már máshonnan, de nem fogta
   meg a fantáziámat"). Ne hozzuk fel újra.
2. **Kontaktus nélküli EMFI (electromagnetic fault injection)** -
   érdekesebbnek találta, de tisztázandó volt az eszközigény. Kiderült: **az
   EMFI NEM a ChipWhisperer-Husky-tól függ**, ez egy külön eszközkategória:
   - **PicoEMP** (NewAE/Colin O'Flynn, nyílt forrású, Raspberry Pi Pico-alapú,
     ~133 USD, Tindie) - "uncalibrated feedback", belépő szintű.
   - **Faulty Cat** (ElectronicCats, PicoEMP/ChipSHOUTER remix, nyílt forrású,
     még olcsóbb kategória).
   - A "komoly", kalibrált verzió a **ChipSHOUTER** (NewAE) lenne, de ára
     valószínűleg meghaladja a PicoEMP/Faulty Cat szintet - pontos ár
     nincs megerősítve, utánanézendő ha ide jutunk.
3. **Valós telepítési körülmény hatásának vizsgálata** (a legkönnyebben
   kivitelezhető, szinte "ingyenes" irány): szinte minden publikált SCA/
   glitching munka csupasz fejlesztői panelen dolgozik - senki nem vizsgálja,
   mennyire működnek ugyanezek a technikák egy **valós tokozásban (IP65 doboz,
   kültéri szerelés), realisztikus távolságból**. Ez nem új technika, hanem
   egy új mérési dimenzió - a meglévő eszközökkel (szkóp, LA, majd
   glitcher/EMFI) is kivitelezhető, csak a target node-ot kell valódi
   tokozásba/kültéri szerelésbe tenni és úgy megismételni a méréseket.
   **Ez kombinálható a 2. ponttal** (EMFI + valós tokozás/deployment teszt),
   mint egy önálló, PicoEMP/Faulty Cat szintű (nem Husky-függő), külön
   beszerzési döntés - nincs ütközésben az EKÖP-re várakozó Husky-döntéssel.

## A "gap statement" a 2020-as cikkhez képest - végső összefogó irány

A 2020-as cikk ("Evaluation of Side-Channel Key-Recovery Attacks on LoRaWAN
End-Device") **kizárólag passzív** (EM-SCA/CPA, hallgatózás) - sosem avatkozik
be a chip működésébe. Ez adja a tiszta "kiegészítés, nem utánzás" keretet:

**2×2 mátrix, amit még senki nem futtatott végig ugyanazon a node-on:**

| | Passzív (hallgatózás) | Aktív (hiba okozása) |
|---|---|---|
| **Kontakt** | Power-SCA (sönt) | Voltage glitching |
| **Érintésmentes** | EM-SCA (ezt csinálta a 2020-as cikk) | EMFI |

Kiegészítő dimenziók, amik tovább erősítik a "gap"-et:
- **Hardeningelt eszköz** (RDP2+PCROP+Secure Boot) - feltéve, hogy a 2020-as
  cikk nem ezt tesztelte (ellenőrzendő a teljes szöveg elolvasásakor).
- **Valós telepítési körülmény** (zárt tokozás, realisztikus távolság) - a
  2020-as cikk feltehetően laborpadon dolgozott.
- **"Teljesen érintésmentes teljes lánc"**: EM-SCA (hallgatózás) + EMFI
  (aktív hiba) kombinálva, kizárólag érintésmentesen, végig egy
  kulcskinyerésig - erős biztonsági üzenet: a fizikai tamper-evidence
  intézkedések (zárt doboz, plombálás) nem elegendők, ha a támadó soha nem
  ér hozzá az eszközhöz.

**Előfeltétel a pontos gap statement megfogalmazásához**: a 2020-as cikk
teljes szövegének elolvasása (ld. fenti TODO) - csak abból derül ki, milyen
chipen, milyen védelmi szinttel, milyen setuppal/távolságból dolgoztak.

## Plusz irány: generikus megoldás/eszköz kifejlesztése (nem csak támadás)

Két lehetőség merült fel, melyik illik jobban a profilhoz:

**A) Generikus védekezés (countermeasure)** - szoftveres hardening library
(maszkolt/blindelt AES, redundáns glitch-detektáló ellenőrzések, canary
értékek a végrehajtási integritásra). Klasszikus attack+defense struktúra,
de mélyebb beágyazott kriptó-implementációs tudást igényel, és a
hatékonyságát is külön be kellene bizonyítani ugyanazokkal az SCA-eszközökkel.

**B) Generikus értékelő eszköz/módszertan (KIVÁLASZTVA - ez illik jobban a
felhasználó profiljához)** - egy nyílt, szabványosított teszt-keretrendszer/
toolkit, ami a fenti 2×2 mátrixot szisztematikusan végigfuttatja BÁRMILYEN
LoRaWAN végponti eszközön, és egyfajta "hardening score"/jelentést ad.
Illik a meglévő eszközépítési stílushoz (`SPI_Key_Extraction`,
`Application_Key_Breach`, `Firmware_Infect` - mind gyakorlati, újrafelhasználható
Python-eszközök). Jelenleg minden cikk ad-hoc módon teszteli a saját
eszközét - nincs közös, összehasonlítható módszertan/eszköz. Ez a "nyílt
eszközkészlet mint hozzájárulás" irány a felhasználó valódi versenyelőnye.

**Ez a végső, összefogó kutatási irány (post-thesis)**: a 2×2 mátrix
szisztematikus végigfuttatása egy hardeningelt LoRaWAN node-on + egy nyílt,
újrafelhasználható toolkit/módszertan, ami mindezt bárki más eszközén is
elvégezhetővé teszi.

## Nyitott teendők (ha valaha visszatérünk erre)

- [ ] Teljes cikk elolvasása (Springer/ResearchGate), Zoteróba felvétel DOI-val.
- [x] EKÖP-keret eldőlése: **nem sikerült** (2026-09-13) - eszközbeszerzés
      (Husky vagy más) egyelőre finanszírozás nélkül marad, nyitott kérdés.
- [ ] Fenyegetési modell szakasz kiegészítése: alacsony vs. magas erőforrású
      attacker-profil, fejezetenkénti hozzárendelés.
- [ ] Ha lesz eszköz: STM32 node ellen SPA -> glitching (RDP downgrade
      kísérlet) -> CPA, ugyanazon a hardveren, összehasonlítva.
- [ ] EMFI irány (PicoEMP/Faulty Cat) + valós tokozás/deployment-teszt -
      külön, Husky-döntéstől független beszerzés, ha érdekel még ez az irány.
