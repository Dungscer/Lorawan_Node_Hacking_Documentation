# LoRaWAN honeypot - brainstorm (2026-07-22)

**Státusz:** ötletelési fázis, semmi nincs megvalósítva. Nem egyértelmű még,
hogy ez bekerül-e a szakdolgozatba - egyelőre csak egy megfogott ötlet,
priorizálva: **gateway-csali előbb, node-csali utána**.

## A téma

Nincs (a keresés szerint) **kifejezetten LoRaWAN-specifikus honeypot**
projekt vagy publikáció - sem GitHubon, sem akadémiai adatbázisokban. Van
általános IoT-honeypot kutatás (Conpot ipari vezérlőrendszerekhez, IoTPOT
Mirai-szerű malware-ekhez) és bőven LoRaWAN *sebezhetőség*-felmérés, de a
kettő metszete hiányzik. Ez egy valóban nyitott, alulkutatott terület lehet,
és jól illeszkedik a korábban azonosított "generikus eszköz/módszertan mint
hozzájárulás" irányhoz (ld. `../2026_07_22_DPA_DCA/brainstorm.md`).

## Prioritás: gateway-csali először, node-csali másodikként

### 1. Gateway-csali (első lépés)

Egy hamis gateway/network szerver végpont, ami vonzza:
- rogue node-okat / próbálkozó csatlakozásokat,
- a UDP packet forwarder elleni támadási kísérleteket (ez összekötné a
  kolléga gateway↔ChirpStack backhaul témáját is, ld.
  `../../.claude` memória `project_scope_gateway_deferred` - csak
  hivatkozásként/megfigyelésként, nem saját implementációként).
- Naplózza: honnan, milyen gyakorisággal, milyen tartalommal érkeznek
  csatlakozási/injektálási kísérletek.

### 2. Node-csali (második lépés, a gateway-csali után)

Egy hamis node, ami sebezhetőnek *tűnik* (pl. gyenge/default AppKey-t
"mutat", vagy sebezhető firmware-verziót jelez a Join Request/DevEUI
mintázatban), és naplózza, ha valaki megpróbálja kompromittálni:
- Join Request brute-force próbálkozások érzékelése,
- SPI-tap-szerű fizikai közelítés jelei (ha mérhető egyáltalán szoftverből),
- gyanús downlink/OTA-injektálási kísérletek.

## Miért illik ez a profilhoz

Ez egy konkrét, megírható Python/firmware-alapú eszköz, nem elméleti
koncepció - közvetlenül a saját testbeden (ChirpStack + node-ok + RAK7268
gateway) tesztelhető és demonstrálható, ugyanabban a gyakorlati
eszközépítő stílusban, mint a meglévő `Hacking/*` PoC-ok.

## Nyitott kérdések / következő lépések

- [ ] Irodalomkutatás mélyítése: van-e egyáltalán bármilyen közeli (nem
      LoRaWAN, de LPWAN/NB-IoT szintű) honeypot-publikáció, amiből tanulni
      lehetne architektúrát/naplózási módszertant.
- [ ] Eldönteni: a gateway-csali a valódi ChirpStack elé/mellé állna-e (pl.
      egy második, "csali" device profile/alkalmazás), vagy egy teljesen
      különálló, minimál LNS-imitáció legyen.
- [ ] Naplózási/riasztási formátum meghatározása (mihez kötné magát a
      dolgozat, ha ez bekerülne: pl. Graylog, amit az ESP32-es node-oknál
      is be kell kötni, ld. `ROADMAP.md`).
- [ ] Eldönteni, bekerül-e a szakdolgozatba, vagy ez is a
      `2026_07_22_DPA_DCA`-hoz hasonlóan post-thesis irány marad.
