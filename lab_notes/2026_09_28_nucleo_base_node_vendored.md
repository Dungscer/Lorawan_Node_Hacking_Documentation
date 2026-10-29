# NUCLEO-WL55JC1 base_node: ST hivatalos forrás bevezetve, korábbi sensor-mock terv visszavonva (2026-09-28)

**Kontextus:** az eszközt most nem tudtam csatlakoztatni, ezért a
`base_node` firmware-t a gyári forrás (`original_firmware/`) alapján
írtam meg. A korábban kijelölt irányt követtem: ST hivatalos
`LoRaWAN_End_Node_LBM` példaprojektjét veszem alapul, nem a gyári
AT-parancsos demo szerkezetét, és nem kézzel újraírt kódot.

## Mi történt

1. `github.com/STMicroelectronics/STM32CubeWL` elérhető volt (git clone ment)
   (a korábbi, 2026-09-24-i próbálkozáskor `st.com` nem volt elérhető,
   de a GitHub-repo igen - ez most is így volt). Sparse+shallow clone
   (`--depth 1`, csak a `Projects/NUCLEO-WL55JC/Applications/LoRaWAN/
   LoRaWAN_End_Node_LBM` alfa), commit `f934cd298bf82d45f96aac2f335c0bbd26d0b946`
   (2026-07-01).
2. Az App-szintű projekt (`Core/`, `LoRaWAN/App`, `LoRaWAN/Target`, `.ioc`,
   `STM32CubeIDE/`) bemásolva `Firmwares/Nodes/NUCLEO-WL55JC1/base_node/
   LoRaWAN_End_Node_LBM/`-be. A gyártói SDK-fák (`Drivers/`,
   `Middlewares/`, `Utilities/` - HAL, BSP, CMSIS, LoRa Basics Modem
   middleware) **szándékosan nincsenek bevezetve a repóba** - méretük és
   az, hogy változatlan vendor-kód, nem indokolja a duplikálást; a
   README leírja, hogyan kell a saját, teljes
   `STM32CubeWL` klónomba visszahelyezni ezt a mappát build előtt.

## A korábbi terv tévesnek bizonyult, visszavonva

A 2026-09-24-i README verzió szerint az egyetlen szükséges patch a
`SendTxData()`-ban lévő `EnvSensors_Read()` hívás mockolása lett volna,
mert "a plain NUCLEO-WL55JC1-en nincs szenzor shield, ez hibázna vagy
szemetet olvasna". **Ez a feltételezés téves volt** - a tényleges forrás
(`Core/Src/sys_sensors.c`, `Core/Inc/sys_conf.h`) ellenőrzése után
kiderült:

- `sys_conf.h`-ban `SENSOR_ENABLED` alapból `0`.
- `SENSOR_ENABLED == 0` esetén `EnvSensors_Read()` már eleve egy biztonságos
  fallback ágat futtat: a hőmérséklet a STM32WL saját belső MCU
  hőmérséklet-szenzoráról jön (`SYS_GetTemperatureLevel()`), a
  páratartalom/légnyomás pedig fix konstans (`HUMIDITY_DEFAULT_VAL` = 50.0,
  `PRESSURE_DEFAULT_VAL` = 1000.0). Nincs hiba, nincs szemét, nem kell
  hozzá shield.

Eredetileg megírtam egy counter-driven mock patch-et (hasonlóan a Heltec
`base_node`-okhoz), aztán a fenti felismerés után **visszavontam** - a
fájl így most bit-pontosan megegyezik ST forrásával (`diff -rq` ellenőrizve).
A felesleges kódot nem hagytam benne, miután kiderült, hogy nem kell.

## Állapot

- **Nincs hardveresen tesztelve** - az eszköz most nem volt csatlakoztatva.
  Ez tehát "implementálva, de nem hardveresen igazolva" állapot, nem "kész".
- `LoRaWAN/App/se-identity.h` ST gyári placeholder-eivel van a repóban
  (nulla DevEUI/JoinEUI, publikus FIPS-197 teszt-AppKey) - biztonságosan
  commit-olható; flash előtt a valós ChirpStack-értékekre kell cserélnem
  (valós kulcs soha ne kerüljön a repóba).
- A build STM32CubeIDE-t igényel; `st.com` erről a gépről nem
  elérhető (a korábbi megfigyeléssel egyezően), otthon kell
  letöltenem/telepítenem és kipróbálnom.

Részletek és pontos build-instrukció:
`Firmwares/Nodes/NUCLEO-WL55JC1/base_node/README.md`.
