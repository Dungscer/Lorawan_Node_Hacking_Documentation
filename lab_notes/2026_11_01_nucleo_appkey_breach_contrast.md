# NUCLEO Application-Key-Breach: vulnerable vs secure kontraszt (2026-11-01)

A `Hacking/Application_Key_Breach` PoC-ot lefuttattam a NUCLEO `otaa_vulnerable`
(gyenge all-zero kulcs) és a secure node (valós random kulcs) ellen. Az attack:
egy elfogott Join Request MIC-jéből offline AppKey-visszafejtés (egy AES-CMAC
jelöltenként), hálózati interakció nélkül.

## Capture (SDR nélkül)
Az RTL-SDR ebben a menetben nem volt elérhető, ezért a ChirpStack device-frame
stream (`InternalService.StreamDeviceFrames`) volt a capture-pont - a README is
engedi ("ChirpStack event log"). A stream a Join Request mezőit dekódolva adja
(join_eui, dev_eui, dev_nonce, mic); ebből rekonstruáltam a 23 bájtos on-air
frame-et (MHDR | AppEUI_LE | DevEUI_LE | DevNonce_LE | MIC). A MIC a node valódi,
sugárzott MIC-je - a breach ebből fejti vissza a kulcsot, nem kapja meg.

## Eredmény
- **Vulnerable (all-zero kulcs):** DevNonce 200, frame
  `000101010101010101e9eb1b0615e18000c8007a2b2cd3`. A breach a wordlist-fázisban
  0.01 s alatt megtalálta: `AppKey = 00000000000000000000000000000000`. Innen a
  session-kulcsok (NwkSKey/AppSKey) levezethetők, a forgalom visszafejthető.
- **Secure (valós random kulcs):** DevNonce 201, frame
  `00831138d53a157026e9eb1b0615e18000c9002f312edd`. A breach 20 szótár-jelölt után
  "AppKey not found" - egy random 128-bites kulcs brute-force-olhatatlan.

Ugyanaz a támadás, azonos eszközön/platformon: a gyenge-kulcs node teljesen
kompromittálódik egy elfogott joinból, a secure node nem. Ez a hardening hatásának
közvetlen, kontrasztos bizonyítéka a NUCLEO-n.

## Teszt-higiénia
A gyenge-kulcs joinhoz a meglévő ChirpStack-eszköz kulcsait ideiglenesen all-zero-ra
állítottam, utána visszaállítottam a valós értékre; a board a valós kulcsú working
firmware-en (secure+aktiváció) joinol. A breach kimenetében szereplő kulcs all-zero
(nem titok, maga a sebezhetőség); a valós kulcs nincs leírva (gitignore-olt
`Information.md`).
