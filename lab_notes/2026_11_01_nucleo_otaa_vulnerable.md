# NUCLEO otaa_vulnerable - gyenge-kulcs join igazolva (2026-11-01)

Megírtam a NUCLEO `otaa_vulnerable` tier-t: a `otaa_basic` LoRa-Basics-Modem
vevője (így az aláíratlan OTA-út öröklött) a szándékos gyengeségekkel - all-zero/
alapértelmezett AppKey a `se-identity.h`-ban (ez maga a sebezhetőség, nem titok,
ezért a forrásban is nullák), `DEBUGGER_ENABLED=1` (nyitott SWD), `OTA_SECURE_MODE=0`
(aláíratlan OTA), titkosítatlan Cayenne-payload. Az option byte-okhoz (RDP) nem
nyúltam - egy téves option-byte-írás az egyetlen, ami valóban téglásíthatná a nem
cserélhető boardot; az RDP0 amúgy is az alap fejlesztői állapot.

## Gyenge-kulcs join hardveren
A firmware DevEUI-ja a chip `GetUniqueId`-ja, vagyis ugyanaz, mint a többi NUCLEO
firmware-é. Hogy a triviális kulcsú joint igazoljam, a meglévő ChirpStack-eszköz
kulcsait (`nwk_key`+`app_key`) ideiglenesen all-zero-ra állítottam (gRPC
`UpdateKeys`), ráflasheltem a vulnerable firmware-t, és a node **joinolt az
all-zero AppKey-jel** (friss DevAddr `01895c7a`, `last_seen` néhány mp). Vagyis egy
alapértelmezett/kitalálható kulcsú node-nál a csatlakozás és a session-kulcsok
levezetése triviális - ez a gyenge-kulcs forgatókönyv a NUCLEO-n.

Utána a ChirpStack-kulcsokat visszaállítottam a valós értékre, és a boardra
visszatöltöttem a valós kulcsú működő firmware-t (secure+aktiváció), ami újra
joinol - a baseline helyreállítva, a kulcs-nullázás szigorúan ideiglenes
teszt-lépés volt.

A valós kulcsot és a konkrét bájtokat nem írom ide (a gitignore-olt
`Information.md`-ben vannak); a forrásba égetett all-zero a sebezhetőség maga, nem
titok.
