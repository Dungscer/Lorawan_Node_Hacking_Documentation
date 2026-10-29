# ChirpStack QuickJS sandbox-escape kutatás - CVE-2026-37630 kizárva, 2026-10-07

**Cél:** megvizsgálni (node-oldali, kizárólag payloaddal
kiváltható út más eszközök AppKey-jéhez), van-e ismert
sandbox-escape sebezhetőség a ChirpStack által használt JS-motorban
(QuickJS/QuickJS-NG, az `rquickjs` Rust binding-on keresztül), ami a
korábban (2026-07-27, `project_chirpstack_codec_attack_idea` memória)
deprioritizált irányt újra megnyithatná. **Tisztán irodalom-/forráskód-
kutatás, nincs hozzá hardveres előfeltétel.**

## Talált jelölt: CVE-2026-37630

- **QuickJS-NG 0.12.1**, `js_mapped_arguments_mark` függvény: a GC
  mark-fázisban nem-detached var-ref-eket objektumként kezel (nem GC-objektum),
  ami memóriakorrupcióhoz és - állítás szerint - kódfuttatáshoz vezethet.
  CVSS 7.3 (High), CWE-94. **Nincs szükség host binding-re, pőre JS elég
  hozzá** - ha igaz, ez pontosan a keresett "node payload -> szerver RCE" út
  lenne.

## Verzió-nyomozás - ez dőtötte el a kérdést

1. A ChirpStack **4.19.0** (a tesztkörnyezet verziója) forrása szerint
   (`Cargo.toml`, `v4.19.0` tag): `rquickjs = { version = "0.12", ... }`.
2. A tényleges **lezárt verzió** (`Cargo.lock`, `v4.19.0` tag, a build-hez
   ténylegesen használt verzió): **`rquickjs 0.12.0`** és
   **`rquickjs-sys 0.12.0`** - pontosan lekérdezve, nem feltételezve.
3. Az `rquickjs-sys` Rust crate **saját maga vendorol egy QuickJS-NG
   motort** git submodule-ként (`sys/quickjs`) - ennek verziószáma **nem**
   azonos az `rquickjs`/`rquickjs-sys` crate verziószámával, a kettő
   véletlenül hasonlít (0.12.x vs 0.12.1), de külön projektek külön
   verziózása.
4. Az `rquickjs-sys 0.12.0` submodule-referenciája a GitHub API szerint a
   `433941b99fb3c5e7f98b7ebd78727972bcf467ee` commit - ez a
   **quickjs-ng repo `v0.15.0` tag-je** (tag-commit üzenete: "Set version
   to 0.15.0", dátum: 2026-05-21).
5. A CVE-2026-37630 által érintett hiba (`js_mapped_arguments_mark` UAF)
   **javítva lett** a quickjs-ng upstream-ben a
   `3973106105…` commitban ("Fix UAF when handling non-detached var-refs
   in mapped arguments", 2026-03-12, ld.
   `quickjs-ng/quickjs#1400`) - ez **megelőzi** a v0.15.0 release dátumát
   (2026-05-21).

**Következtetés: a ChirpStack 4.19.0 ténylegesen beépített motorja
(quickjs-ng ~v0.15.0) már tartalmazza a javítást - a CVE-2026-37630 nem
alkalmazható erre a tesztkörnyezetre.** A verziószám-egyezés (rquickjs
"0.12.x" ↔ a sebezhető quickjs-ng "0.12.1") félrevezető volt, csak a
submodule-commit tényleges lekövetésével derült ki a valós kép.

## Mi NEM dőlt el ezzel

- Ez **egyetlen konkrét CVE-t zár ki**, nem jelenti azt, hogy a QuickJS-NG
  motor általában sandbox-escape-mentes. Más, esetleg még nem CVE-számozott
  hiba elvileg létezhet.
- Nem vizsgáltam még, hogy az `rquickjs`-en keresztül a ChirpStack maga
  köt-e be bármilyen host-funkciót a JS-kontextusba a dokumentált
  `bytes`/`fPort`/`variables` mezőkön túl (a `bindgen` és `array-buffer`
  feature-ök be vannak kapcsolva - ezek önmagukban nem jelentenek
  sebezhetőséget, de a pontos API-felület még nincs feltérképezve).

## Folytatás: a vendorolt motor ismert CVE/GHSA-listájának átfésülése

A `quickjs-ng/quickjs` GitHub Security Advisory listája **10 nyilvános
advisory-t** tartalmaz. A ténylegesen vendorolt verzióra (**v0.15.0**,
ld. fent) leszűrve:

| GHSA | Súlyosság | Érintett tartomány | v0.15.0 érintett? |
|---|---|---|---|
| GHSA-m53g-qx47-3h38 | **critical** - RCE `js_array_toReversed()` | `<0.14.1`, javítva `0.15.0`-ban | **nem** (pont a javító verzió) |
| GHSA-52gx-967m-hxp3 | **high** - UAF `js_array_buffer_transfer` külső ArrayBuffer-en | `<=0.15.1` (+ explicit `rquickjs<=0.12.1`, `rquickjs-sys<=0.12.1`) | **igen, a könyvtár szintjén** - ld. lent |
| GHSA-rxpx-wj25-jhqq | high - RegExp OOM OOB write | `>=0.15.1` | nem (a verzióm korábbi) |
| GHSA-r6vf-p8wr-4vg9 | high - UAF `set_array_length` | `<0.14.0` | nem |
| GHSA-9wv5-5cgr-grj6, GHSA-jp52-chwg-2p9p | high - 32-bites integer overflow-ok | `<=0.8.0` | nem |
| GHSA-3jf7-4qfx-xc2h | medium - `JSON.stringify` rope-pointer leak | `<0.16.0` | **igen** - ld. lent |
| GHSA-v647-xff9-jxf7, GHSA-crgj-9jv8-xfhq | low - OOM-indukált UAF-ok | `<0.16.0` | igen, de "low" + OOM-függő |
| GHSA-wm7w-r6qf-3c9w | medium - BigInt memory leak | `<=0.8.0` | nem |

### A legsúlyosabb jelölt: GHSA-52gx-967m-hxp3 (high, UAF -> arbitrary read)

Ez a hiba **explicit megnevezi az `rquickjs`/`rquickjs-sys` Rust
csomagokat** `<=0.12.1` verzióig sebezhetőként - a ChirpStack 4.19.0
pontosan ezt használja (`0.12.0`). A publikus PoC egy teljes,
bizonyított exploit-láncot mutat be: `ArrayBuffer.prototype.transfer()`
hívás egy **"külső" (Rust-oldalon, saját `free_func`-kal létrehozott)**
ArrayBuffer-en memóriakorrupciót (UAF) okoz, amiből tetszőleges
memóriacím-olvasás építhető - a szerzők ezzel ténylegesen kiolvasták a
host-folyamat környezeti változóit (`__environ`-láncon át).

**Miért NEM érhető el mégsem a ChirpStack kódek-útvonalán (forráskód-
ellenőrzéssel igazolva):** a hiba kizárólag **"külső" ArrayBuffer-eken**
váltható ki (amit a host, azaz a Rust-kód hoz létre `TypedArray::new`
vagy ezzel ekvivalens API-val, saját memóriafoglalóval). A ChirpStack
`codec/js/mod.rs` forrása szerint a payload-bájtok (`bytes`) egy sima
`Vec<u8>` -> `.into_js(&ctx)` konverzióval kerülnek be, és az `rquickjs`
`Vec<T>` -> `IntoJs` implementációja ezt **sima JS tömbbé** (számok
listája, `[10, 255, ...]`), **nem** TypedArray/ArrayBuffer-ré alakítja
(`core/src/value/convert/into.rs`, a `list:` ághoz generált makró).
A `chirpstack_input.bytes` objektumnak tehát nincs is `.transfer()`
metódusa. A vendorolt `buffer` JS-modul (Node.js `Buffer` polyfill) saját
maga is csak sima, motor-belső `new ArrayBuffer()`-t hoz létre a
sandboxolt JS-ből, ami **nem** "külső" típusú - a hibás ág védelmi
feltétele (`free_func != js_array_buffer_free`) erre nem áll fenn.
**Ellenőrzött, forráskód-alapú negatív eredmény**, nem feltételezés.

### A pontosan reprodukálható, de más jellegű jelölt: GHSA-3jf7-4qfx-xc2h (medium, info-leak)

Ez a hiba **pőre JavaScripttel, host-kötés nélkül** kiváltható: egy
`JSON.stringify(érték, null, hosszú_összefűzött_string)` hívás, ahol a
3. paraméter (`space`) egy ún. "rope" (lusta string-összefűzés belső
reprezentáció) - ilyenkor a motor a rope belső pointer-mezőjének első
bájtjait írja ki JSON-ként, egy valódi heap-címet szivárogtatva a
kimenetbe. **Ez tényleg reprodukálható lenne** egy kódek-szkripten
belülről, mert sima nyelvi feature-t használ, nincs szükség semmilyen
ChirpStack-specifikus bindinghez.

### A döntő korlát - threat model, nem technikai

**Ez a hiba (és az összes motor-szintű hiba, amit eddig találtam) csak
akkor váltható ki, ha a támadó irányíthatja a kódek JS *forráskódját*
(pl.\ rá tudja venni, hogy `JSON.stringify`-t hívjon egy saját maga
összerakott rope-paraméterrel) - nem elég, ha csak a `bytes` *bemenetet*
irányítja.** A ChirpStack valós modelljében a kódek szkriptjét az
üzemeltető írja/konfigurálja, bizalmi forrásból - egy kulcs nélküli,
csak rádión kommunikáló RF-outsider a node payload-ján keresztül
**nem tudja módosítani magát a kódeket**, csak annak bemeneti bájtjait.

Ez azt jelenti: a fenti motor-hibák kihasználásához az kéne, hogy **maga
az (operátor által írt) kódek-szkript** valamilyen szokatlan módon
közvetlenül a nyers bájtokból építsen fel veszélyes konstrukciót (pl.\
`JSON.stringify`-t hívjon a bájtokból képzett string paraméterrel, vagy
ArrayBuffer-t `.transfer()`-ezzen) - ez jóval szűkebb, "rosszul írt
kódek" forgatókönyv, analóg a korábban már lezárt injection-lánccal
(2026-10-05, stored XSS), nem egy általános "bármilyen ChirpStack
kódek ellen működik" eredmény.

## Összegzés, 2026-10-07

- **3 konkrét jelölt megvizsgálva, mindhárom korlátozott/kizárt a jelen
  node-only, trusted-kódek threat modellben:**
  1. CVE-2026-37630 (`js_mapped_arguments_mark`) - **a vendorolt motor-
     verzió már javítva** (verziónyomozással igazolva).
  2. GHSA-52gx-967m-hxp3 (`js_array_buffer_transfer` UAF) - **a motor-
     verzió sebezhető, de a ChirpStack bytes->JS konverziója nem hoz
     létre "külső" ArrayBuffer-t**, a dokumentált kódek-bemeneten
     keresztül nem érhető el (forráskód-alapú igazolás).
  3. GHSA-3jf7-4qfx-xc2h (`JSON.stringify` rope-leak) - **technikailag
     kiváltható pőre JS-sel**, de csak akkor, ha maga a (bizalmi)
     kódek-szkript küldi bele a nyers bájtokat egy ilyen konstrukcióba -
     a node önmagában, a payload-bájtjaival nem éri el.
- **A threat model maga a korlát, nem (csak) a konkrét hibák.** A
  "node-only, kulcs nélküli RF-outsider -> más eszközök AppKey-je"
  célkitűzés ezen az úton addig nem közelíthető meg, amíg a kódek
  szkriptjét bizalmi/operátori forrásnak tekintem - ami a valós
  ChirpStack-üzemeltetés modellje is.
- **Nyitva marad, ha folytatom:** megnézni, hogy valamelyik *gyakori,
  valós* kódek-minta (pl.\ amit a LoRaWAN-közösség gyakran másol/használ)
  véletlenül funnel-eli-e a nyers bájtokat egy ilyen veszélyes API-ba -
  ha igen, az "rosszul írt, de tipikus kódek" forgatókönyvként
  demonstrálható, hasonlóan a 2026-10-05-i injection-lánchoz.

---

## Folytatás: a rope-leak hiba helyi reprodukálása a valódi motoron (2026-10-07, második kör)

**Fontos: ez a lépés sem igényelt labor-/hálózati hozzáférést** - a
nyílt forráskódú quickjs-ng motort közvetlenül, forrásból fordítottam
le **pontosan azon a commit-on**, amit a ChirpStack 4.19.0 ténylegesen
vendorol (`rquickjs-sys 0.12.0` -> submodule commit
`433941b99fb3c5e7f98b7ebd78727972bcf467ee` = `v0.15.0` tag), majd
lefuttattam rajta a GHSA-3jf7-4qfx-xc2h (`JSON.stringify` rope-pointer
leak) publikus PoC-jának saját változatát.

**Eredmény: a hiba valóban reprodukálódott.** Negatív kontroll (rövid,
literál `space` string) helyesen literál ASCII-t adott vissza. Támadó
eset (20 000 karakteres, összefűzésből képzett "rope" string `space`-ként)
**nem literál, a motor belső `JS_TAG_STRING` tag-mintázatával (`0xFF...F9`)
egyező bájtokat** adott vissza a karakterek helyett - pontosan a hiba
leírása szerint. Teljes jegyzőkönyv:
`Firmwares/Nodes/Heltec_WiFi_LoRa_32_V4/chirpstack_js_engine/rope_leak_poc/local_repro/transcript.txt`.

**Ez tehát már nem csak verziószám alapú feltételezés, hanem ténylegesen
futtatott, megismételt bizonyíték** a pontos motorverzióra. Amit ez
*nem* dönt el: hogy a ChirpStack saját `rquickjs`-beágyazásán keresztül
(nem a sima `qjs` parancssori motoron) is ugyanígy viselkedik-e - ehhez
már kell a labor.

**Előkészítve az élő teszthez** (nincs hardveres előfeltétele a
*megírásnak*, csak a *futtatásnak*):
`Firmwares/Nodes/Heltec_WiFi_LoRa_32_V4/chirpstack_js_engine/rope_leak_poc/decode_rope_leak.js`
- kész `decodeUplink` teszt-kódek, amit a meglévő `ATTACK-jscodec`
device profile-ra kell beilleszteni, majd bármilyen uplinkkel kiváltani.
Pontos lépések: `rope_leak_poc/README.md`.

**A threat-model-korlát továbbra is érvényben van** (ld. fent): ez a
teszt-kódek szándékosan, kézzel van megírva úgy, hogy kiváltsa a hibát -
egy valódi, módosítatlan kódek ellen egy kulcs nélküli RF-outsider a
payload-bájtjaival önmagában ezt nem tudná elérni. A cél itt annak
megerősítése, hogy a motor hibája *egyáltalán* elérhető-e a ChirpStack
tényleges beágyazásán keresztül - nem egy általános, automatikus
támadás bármilyen kódek ellen.
