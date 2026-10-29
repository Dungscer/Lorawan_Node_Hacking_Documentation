# Application_Key_Breach igazolás (erős vs gyenge kulcs), 2026-10-17

**Cél:** a `Hacking/Application_Key_Breach` README + mérési eredmények
véglegesítése (ROADMAP 1. sáv, határidő 2026-10-20). A tool egy elkapott
Join Request MIC-jéből offline próbál AppKey-t visszafejteni:
`MIC = AES-128-CMAC(AppKey, MHDR|AppEUI|DevEUI|DevNonce)[0:4]`. A lényeg a
**biztonsági határ** bemutatása: gyenge kulcs azonnal elesik, erős ellenáll.

## Erős kulcs - valódi elkapott frame, ellenáll
- Forrás: a Heltec V2 `base_node` **valódi** Join Requestje, amit a kettős napló
  futtatáskor fogtam el a ChirpStack device streamen (`frame_logger.py`,
  ld. `2026_10_17_v2_base_node_dual_log_bringup.md`): join_eui
  `d6e3…5502`, dev_eui `05ec…8fe5`, dev_nonce 49171, MIC bájtok
  `[97,198,238,70]`.
- A 23 bájtos on-air frame visszaépítve a mezőkből (MHDR|AppEUI(LE)|DevEUI(LE)|
  DevNonce(LE)|MIC). **Keresztellenőrzés:** a valódi AppKey-jel számolt CMAC MIC
  bit-pontosan egyezett az elkapott MIC-kel (`61c6ee46`) - ez igazolja a helyes
  mező/endian sorrendet.
- Tool futtatás (default hybrid): wordlist 20 jelölt -> nincs találat; sequential
  max 2 nem-nulla bájt, 7 807 080 jelölt -> **nincs találat, 31 s**. A valódi
  kulcs 16 nem-nulla bájtos random érték, így a sequential elérhetetlen -
  pontosan ahogy kell.

## Gyenge kulcs - szintetizált frame, elesik
- Join Request a `0102030405060708090a0b0c0d0e0f10` tutorial-kulccsal aláírva
  (ez benne van a `wordlist.py`-ban; ugyanaz, amit a `Weak_AppKey_OTAA` PoC
  használ), DevEUI `feedface00000001`, DevNonce 0x1234.
- Tool (`--mode wordlist`): **feltörve 0.03 s alatt**, AppKey helyesen
  visszanyerve.

## Megjegyzések
- A `appkeys.db` (tracked SQLite) a gyenge futtatáskor eltárolta a találatot;
  ezt **visszaállítottam** (`git checkout`), hogy a commit ne tartalmazzon
  teszt-melléktermék PoC-adatot.
- A valódi V2 AppKey **nincs** a repóban (sem itt, sem a README-ben) - csak offset/
  maszkolt hivatkozás, a valódi érték a gitignore-olt `Information.md`-ben.
- Ez az offline fele ugyanannak a láncnak, amit a `Hacking/Weak_AppKey_OTAA`
  élesben, végponttól-végpontig mutatott (RF-sniff -> crack -> session kulcsok),
  2026-10-05-én igazolva.
- Frame-építő script: `/tmp/build_frames.py` (nem repó).
