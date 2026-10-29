# JS-codec injection chain - a nyelő lezárva (stored XSS), 2026-10-05

## Mi ez - és mi NEM (hitelesség a tézishez)
- **Nem** ChirpStack-0-day/CVE. A DoS-ág a 4.19.0-ban mitigált (2026-10-04),
  a sandbox-escape alátámasztatlan. Marad az **injection-osztály**.
- Ez egy **sérülékenység-osztály működő PoC-ja**: *rosszindulatú/kompromittált
  LoRaWAN-node a payload-bájtjaival injectál az operátori toolingba, ha a
  device-profile codec sanitizálatlanul továbbítja, és egy downstream fogyasztó
  sanitizálatlanul rendeli.* Két láncszemet (a hibás codec + a hibás dashboard)
  magam írtam - a severity az **alkalmazásrétegi sanitizáláson** múlik, nem a
  ChirpStack/QuickJS magján.
- Hardveren igazolt alap (2026-10-04): a codec **verbatim passthrough** - a
  támadó tetszőleges bájtjai változatlanul érnek `object.raw`-ba MQTT-n. Emiatt
  a payload-string szabadon választható -> a sink-PoC ezt használja ki.

## Setup
- `sink_poc/dashboard.html` - szándékosan sebezhető „fleet monitor": minden
  uplinkre a dekódolt `object.raw`-t írja a DOM-ba. A sebezhető sor jelölve:
  `cell.innerHTML = raw;  // <-- SINK`.
- `sink_poc/operator_open.mjs` - jsdom-harness: betölti a dashboardot
  (`runScripts:'dangerously'`, origin `https://fleet.monitor.local`), és a
  ChirpStack által publikált **pontos** `event/up` üzeneteket adja neki.
- jsdom 25.0.1, node v24.11.0. Telepítés: `npm install` (jsdom, gitignore-olt).

## Eredmény (ma, ezen a gépen - `node operator_open.mjs`, exit 0)
1. **Kontroll (ALL_ZERO, len 8):** `object.raw` = 8× `\u0000` -> üres render,
   0 injektált elem. (Ez megy most a wire-ön.)
2. **L1 - markup injection (HARDVEREN MEGFIGYELT string):** `object.raw` =
   `'";</script><img src=x>${7*7}--` -> az `<img>` **élő DOM-elemmé** válik az
   operátor-oldalon (`querySelector('img')` megtalálja), nem szövegként.
3. **L2 - script-végrehajtás (weaponizált, szoftveres demó):** `object.raw` =
   `<img src=x onerror="…">` -> a támadó JS **lefut az operátor-originben**
   (`window.__xss_fired=true`, proof: „JS ran in operator origin
   https://fleet.monitor.local").

## jsdom-korlát
A jsdom **nem tölti be** az `<img>`-et (a ResourceLoader `fetch`-je meg sem
hívódik - leteszteltem), így a `src=x` sikertelen betöltése **nem vált ki**
automatikus `onerror`-t, mint valós böngészőben. Ezért a harness: (a) igazolja,
hogy az injektált `onerror` **élő event-handlerré** regisztrálódott az
operátor-originben, majd (b) kibocsátja **ugyanazt az `error` eseményt**, amit
valós böngészőben a sikertelen képbetöltés automatikusan dob. A **handler**
támadó-vezérelt; a **trigger** valós operátor-böngészőben automatikus, itt csak
a jsdom-korlát miatt szintetizált. L2-t tehát „szoftveres sink-demóként" címkézem,
nem „hardveren igazolt on-wire végrehajtásként".

## Mitigáció (igazolva)
`cell.textContent = raw` -> egyik szint sem jön át: nincs injektált elem, a
payload szó szerinti szövegként renderelődik (külön leteszteltem). Mélységi
védelem: a codec-kimenetet is megbízhatatlanként kell kezelni szerveroldalon.

## Állapot a teszt után
- Új anyag csak a `sink_poc/` alatt (PoC) + ez a lab note. A `node_modules/` és
  `package-lock.json` gitignore-olt a PoC-mappában.
- A testbed ChirpStack itt **érintetlen** (ez tisztán szoftveres
  sink-demó a már rögzített MQTT-stringen). A `ATTACK-jscodec` profil + V4
  regisztráció a 2026-10-04 állapot szerint bent maradt (visszavonható).
- Titok nincs a forrásban (a DevEUI a PoC-ban `0114f46d-demo` placeholder).
