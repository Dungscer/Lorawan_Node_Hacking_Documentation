# (B) NUCLEO OTA újratervezés - külső SPI flash + másodlagos bootloader

**Dátum:** 2026-10-02. **Státusz:** DESIGN (hardver nélkül eddig jutottam;
a megvalósításhoz SPI NOR flash chip kell - ld. "Hardver" lent).

## Miért kell újratervezni

Az `otaa_basic`/`otaa_secure` OTA-vevő a teljes image-et **belső flashbe**
stage-eli, majd A/B cserével aktiválná. Ez a STM32WL55JCI-n **nem működik**
(mért adatok, 2026-10-02):

- Linker FLASH régió: `0x08000000`–`0x0803E000` (248 KB); NVM kontextus
  `0x0803F000`+.
- Futó image: `base_node` 119 KB, `otaa_basic` **123 KB**.
- Szabad flash: ~129 KB - **két teljes image nem fér be** 256 KB-ba.
- A chip **egybankos** (nincs `DBANK`), tehát nincs hardveres bank-swap
  aktiváláshoz (ld. `otaa_basic/README.md`).

## Választott architektúra: külső SPI flash staging + mini-bootloader

```
  [LoRaWAN downlink: manifest + chunk-ok]  (meglévő Tools/OTA_Update framing)
             │
             ▼
  App (otaa_basic/secure)  ── a chunkokat KÜLSŐ SPI flashbe írja (staging)
             │  teljes image megérkezett + SHA-256 (+ secure: Ed25519) OK
             ▼
  OTA-state lap (belső flash, 1 db 2 KB)  ← "pending image ready @ ext addr, ver=N"
             │  reset
             ▼
  Mini-bootloader (belső flash 0x08000000, ~8–16 KB)
      - boot-kor olvassa az OTA-state lapot
      - ha pending: KÜLSŐ flash -> BELSŐ app-régió másolás, állapot törlése
      - (secure: a másolás előtt újra-ellenőrzi az aláírást/hasht)
      - ugrás az app-ra
```

### Belső flash-térkép (javaslat)

| Régió       | Cím          | Méret   | Tartalom                                                 |
|-------------|--------------|---------|----------------------------------------------------------|
| Bootloader  | `0x08000000` | 16 KB   | mini-bootloader (saját, kis image)                       |
| App         | `0x08004000` | ~216 KB | `otaa_*` alkalmazás (a linker ORIGIN-t erre kell tolni!) |
| OTA state   | `0x0803D800` | 2 KB    | pending-flag, verzió, ext-flash cím, hash                |
| NVM (modem) | `0x0803F000` | 4 KB    | LBM kontextus (változatlan)                              |

A **teljes staging a külső flashen** van, így a belső flash méret-fala
megszűnik. A külső chip (pl. 4–8 MB) bőven elég több image-verzióhoz +
rollbackhez.

### Wire protokoll

Változatlan: a `Tools/OTA_Update/framing.py` v3 manifest+chunk formátuma
(verzió, total_bytes, SHA-256, Ed25519 aláírás; per-chunk CRC8). A `basic`
nem kényszeríti az aláírást/anti-rollbacket, a `secure` igen - ugyanaz a
protokoll, ld. `otaa_basic/README.md`.

## Hardver ( amit be kell kötni - majd együtt)

- **SPI NOR flash chip**, pl. **Winbond W25Q32/64** (DIP-8 vagy modul).
- **SPI periféria:** a belső rádió a dedikált SUBGHZSPI-t használja (nem
  elérhető kívül), tehát **SPI1 vagy SPI2** egy szabad lábkiosztáson. A
  pontos szabad GPIO-kat a NUCLEO-WL55JC sematikából kell megerősíteni
  (CLAUDE.md #5 - nem tippelek lábat); jelöltek: SPI1 (SCK/MISO/MOSI) +
  egy GPIO CS. A logic analyzert ezekre a SPI-vonalakra lehet kötni a
  bringup hibakereséséhez.
- Bekötés után: SPI flash driver (JEDEC ID olvasás -> sector erase -> page
  program -> read), majd az `ota_update.c` staging-rétegét átirányítani a
  belső `FLASH_IF_Write` helyett a SPI driverre.

## Alternatíva: ST SBSFU / Secure Engine

ST kész secure-boot + firmware-update kerete (dual-slot, aktiválás,
anti-rollback, aláírás) **pontosan ezt oldja meg** - de **st.com-login
mögött** van (ugyanaz a fal, mint a CubeIDE, ld.
`project_stm32wl_toolchain` memória). Ha a letöltés menne, ez gyorsabb út
lenne, mint saját bootloader. Nyitott: megpróbálja-e a felhasználó letölteni.

## Megvalósítási sorrend (javaslat)

1. SPI flash chip bekötése + driver bringup (JEDEC ID, erase/program/read) -
   **logic analyzer itt segít**.
2. `ota_update.c` staging átirányítása a SPI flashre; a `#error` címek
   helyett a külső flash offsetjei.
3. Mini-bootloader (belső `0x08000000`, 16 KB) + a linker ORIGIN app-ra
   tolása (`0x08004000`).
4. End-to-end OTA teszt: `Tools/OTA_Update/main.py`-ból push -> staging ->
   reset -> bootloader copy -> új app boot. (Ez már az `otaa_basic` OTA
   valódi igazolása.)
5. `otaa_secure`: aláírás-ellenőrzés + anti-rollback bekapcsolva - ez már
   az interaktív "támadás + patch" fázis (külön, a felhasználóval).

## Mi van kész most

- `otaa_basic` **buildel + joinol** hardveren (provizórikus belső staging-
  címekkel, az OTA adat-út tudottan nem működik - ez a fenti terv tárgya).
- **1. lépés (SPI flash bringup) KÉSZ, 2026-10-23, hardveresen igazolva.** A
  megérkezett chip **Winbond W25Q64** (8 MiB, JEDEC ID `EF 40 17`), bekötve a
  NUCLEO Arduino-fejlécére (PA5/6/7 = SPI1 AF5, PA4 = CS). Csupasz-regiszteres
  SPI1 driver (`base_node/.../Core/Src/w25q_bringup.c`), a JEDEC ID + SR1
  helyesen beolvasva a VCP-n. Részletek + build/flash buktatók:
  `Documentation/lab_notes/2026_10_23_nucleo_w25q_spi_bringup.md`.
- **2. lépés (erase/program/read) KÉSZ, 2026-10-23, hardveresen igazolva:**
  sector erase (0x20) -> blank-check -> page program (0x02) -> read-back verify
  PASS a W25Q64-en (self-teszt `0x001000`-nél). A teljes flash írás/olvasás út
  megy.
- **Következő: 3. lépés** - az `ota_update.c` staging átirányítása a W25Q-ra +
  mini-bootloader; majd 4. lépés end-to-end OTA teszt (határidő 2026-10-25).
