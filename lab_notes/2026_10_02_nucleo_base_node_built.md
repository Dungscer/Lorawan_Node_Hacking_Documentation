# NUCLEO base_node - st.com-mentes build sikeres (2026-10-02)

Folytatása a `2026_10_02_nucleo_swd_access_attempt.md`-nek (ott oldottam meg
a pyOCD-s flash/debug elérést); ez a build-oldal.

## Miért kellett saját build-rendszer

ST a `LoRaWAN_End_Node_LBM`-et csak STM32CubeIDE/MDK-ARM/EWARM projektként
adja - **nincs Makefile**. A CubeIDE letöltése a myST-login mögött nem
ment. Megoldás: a forrás GitHubról (`STM32CubeWL`, `--recursive`), a
fordító Debian apt-ból (`gcc-arm-none-eabi`), a build-rendszert pedig a
CubeIDE projektfájlokból generáltam.

## Lépések (reprodukálható)

```bash
sudo apt install gcc-arm-none-eabi
git clone --recursive --depth 1 https://github.com/STMicroelectronics/STM32CubeWL.git
# HEAD = f934cd2 (2026-07-01), pontosan amit a base_node/README rögzített
cd STM32CubeWL/Projects/NUCLEO-WL55JC/Applications/LoRaWAN/LoRaWAN_End_Node_LBM
python3 gen_makefile.py     # a repóban: base_node/LoRaWAN_End_Node_LBM/gen_makefile.py
make -j$(nproc)
```

## Mit csinál a generátor (`gen_makefile.py`)

A `.project` + `.cproject`-ből gépileg nyeri ki:
- **Források (119 db):** a `.project` explicit fájl-linkjei (HAL-subset,
  Utilities, Core, App, BSP, CMSIS system) + a két mappa-link
  (`Middlewares/LoRaWAN` = LoRa Basics Modem, `Middlewares/SubGHz_Phy/lorawan`)
  rekurzív `*.c`-i, a `.cproject` `excluding=` listáját tiszteletben tartva
  (kizárva: sx128x/sx127x/lr11xx/llcc68 rádiódriverek + opcionális LBM
  csomagok - geolocation, fragmented data block, clock sync, firmware
  management, multicast stb.). Marad az **sx126x** út (a STM32WL belső
  rádiója).
- **Include-ok (40 db):** a `STM32CubeIDE/Debug` build-mappához képesti
  relatív utak feloldva (ez volt a két buktató: a `../../Core/Inc` a
  `Debug/`-hoz relatív, nem a projekt gyökeréhez; a kizárások pedig a
  workspace-virtuális `Middlewares/LoRaWAN/...` néven vannak, nem a valós
  `Middlewares/Third_Party/LoRaWAN/...` úton).
- **Define-ok (7 db):** `CORE_CM4 DEBUG ENDNODE NUMBER_OF_STACKS=1
  STM32WL55xx SX126X USE_HAL_DRIVER`.
- Cortex-M4, soft-float, `-specs=nano.specs -specs=nosys.specs`, a vendorolt
  `STM32WL55JCIX_FLASH.ld` + `startup_stm32wl55jcix.s`.

## Eredmény

```
   text    data     bss     dec   filename
 120120    1208   20240  141568   build/LoRaWAN_End_Node_LBM.elf
```
`.bin` = 121 336 bájt. ~120 KB flash / ~21 KB RAM - belefér a
STM32WL55JCI-be (256 KB / 64 KB).

## Flashelés hardverre (2026-10-02) - tooling-tanulságok

Lokálisan (a klón `se-identity.h`-jában, **nem commitolva**) beégetve egy
lab-teszt DevEUI/JoinEUI/AppKey, rebuild, majd flash. Amit megtanultam:

- **A pyOCD flash ennél a targetnél CSENDBEN HIBÁZIK.** „programmed 121856
  bytes"-t jelez, de a flash utána `0xffffffff` marad (reset-halt + halted
  olvasással ellenőrizve). Nem használható STM32WL55 flashelésre.
- **A pyOCD mass-erase viszont MŰKÖDIK** (`pyocd erase --chip -M
  under-reset`) - és ez oldotta fel a patthelyzetet: a gyári alvó firmware
  törlése után a chip üresen a bootloaderben ül, nem alszik, így a debug
  kapcsolat stabil. **DE: a mass-erase visszaállította az option byte-okat
  gyári defaultra** (`FLASH_OPTR = 0x3FFFF0AA`).
- **A flashelés OpenOCD-vel ment, verify-jal:**
  ```
  openocd -f interface/stlink.cfg -f target/stm32wlx.cfg \
    -c "init" -c "halt" \
    -c "program build/LoRaWAN_End_Node_LBM.elf verify reset exit"
  ```
  -> `** Verified OK **`. A flash eleje a halted olvasásban is a mi
  vektorunk: `0x08000000=20008000` (SP), `0x08000004=08019e05` (reset) -
  bitre egyezik az ELF `.isr_vector`-ával.

**Nyitott (boot-konfiguráció):** reset-halt után a PC **`0x1fff2cc0`** =
rendszer-bootloader ROM, nem a firmware-em. Ok: a default
`FLASH_OPTR=0x3FFFF0AA`-ban `nSWBOOT0=1`, azaz BOOT0 a **PH3 lábról** jön,
és ezen a (klón/variáns) boardon a láb magas -> bootloader. A 09-19-i
jegyzetben a gyári firmware a flashből indult (reset-halt PC `0x0800014c`),
tehát a gyári option byte-ok valószínűleg `nSWBOOT0=0`-val szoftveres
flash-bootot kényszerítettek - ezt a mass-erase törölte.

## MEGOLDVA - hardveresen igazolt join (2026-10-02)

### 1. Boot-fix: `nBOOT0` AKTÍV-ALACSONY ezen a chipen
A boot-konfigot OpenOCD option_write-tal javítottam (RDP-t nem érintő
maszk, `0x0c000000`), de a polaritás a vártnak az **ellentéte** volt:
- `nSWBOOT0=0, nBOOT0=0` (`FLASH_OPTR=0x33fff0aa`) -> **még mindig
  bootloader** (reset-halt PC `0x1fff2cc0`).
- `nSWBOOT0=0, nBOOT0=1` (`FLASH_OPTR=0x3bfff0aa`) -> **flashből boot**
  (reset-halt PC `0x08019e04` = a reset-handlerem).

Tehát az „nBOOT0" aktív-alacsony: **`nBOOT0=1` -> BOOT0=0 -> main flash**.
A végleges parancs (halt után, külön session a latch-eléshez / tápciklus):
```
stm32wlx option_write 0 0x20 0x3bfff0aa 0x0c000000
stm32wlx option_load 0
```
Az OBL-reload ST-Linken néha „failed"-et jelez, de új session/tápciklus
után az OPTR a `0x3bfff0aa`-t mutatja és flashből indul.

### 2. DevEUI: a firmware a CHIP EUI-t használja, nem a se-identity-t
Kulcsfontosságú megfigyelés a gateway-frame capture-ből: a Join Request-ek
a `se-identity.h`-ban beállított DevEUI (`c4d2…`) helyett a chip gyári,
**ST-OUI-s egyedi DevEUI-jával** (`0080e115061bebe9`) mennek ki. Ok a
`lora_app.c`-ben (`SMTC_MODEM_EVENT_RESET` ág):
```c
GetUniqueId(user_dev_eui);            // <-- felülírja a se-identity DevEUI-t!
smtc_modem_set_deveui(stack_id, user_dev_eui);
smtc_modem_set_joineui(stack_id, user_join_eui);   // JoinEUI a se-identity-bol
smtc_modem_set_appkey (stack_id, user_gen_app_key);// AppKey  a se-identity-bol
smtc_modem_set_nwkkey (stack_id, user_app_key);
```
Tehát a **DevEUI mindig a chipé**, a JoinEUI/AppKey viszont a
`se-identity.h`-ból jön. **Következmény:** a ChirpStackben a device-ot a
chip EUI-val (`0080e115061bebe9`) kell regisztrálni, az AppKey az enyém
(`86b1deca…`). (Ha saját DevEUI kell, a `GetUniqueId` sort kell kivenni.)

### 3. ChirpStack + join
- Device regisztrálva: DevEUI `0080e115061bebe9`, JoinEUI `2670153ad5381183`,
  AppKey `86b1decab33df85114a67921139b98ba` (nwk_key+app_key), app
  `Application_01`, profil „OTAA" (EU868; a mac=1.0.0 profil interoperált az
  1.0.4 firmware join-jával - növekvő DevNonce, gond nélkül).
- Gateway `rak7268v2` (`ac1f09fffe1f4131`) online; a Join Request-ek
  868.1/868.3/868.5 MHz BW125 - szabályos EU868, erős RF.
- **Eredmény:** join sikeres, **DevAddr kiosztva** (pl. `0088ccc6`, majd
  újraindítás után `00342d61`), **fCntUp=48**, NFCntDown=3, last_seen aktív
  - a node folyamatosan uplinkel és downlinket is fogad. Kétirányú
  működés a valódi hálózaton igazolva.

### Megjegyzés: a kulcsok nincsenek commitolva
A DevEUI a chipé (nem titok), az AppKey csak a klón (untracked)
`se-identity.h`-jában és a ChirpStack adatbázisában él - a repóba nem került.
