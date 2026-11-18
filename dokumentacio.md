# BB84 kvantumkulcsszétosztó (QKD) szimulátor – dokumentáció

**Tárgy:** Kvantuminformatikai alkalmazások
**Házi feladat:** 4. feladat – BB84-szimulátor készítése
**Félév:** 2026 ősz

---

## 1. Bevezető – mi a QKD és miért jó?

A kvantumkulcs-szétosztás (Quantum Key Distribution, QKD) célja, hogy két fél, a
hagyományos elnevezéssel Alice és Bob, egy közös titkos kulcsot tudjon létrehozni úgy,
hogy közben egy esetleges lehallgató (Eve) jelenléte **észrevehető** legyen. A klasszikus
kulcscserénél (pl. Diffie–Hellman) a biztonság matematikai nehézségen alapul, amit egy
elég erős (akár kvantum-) számítógép elvileg feltörhet. A QKD ezzel szemben a fizika
törvényeire épít: egy ismeretlen kvantumállapotot nem lehet tökéletesen lemásolni
(no-cloning tétel), és a mérés megváltoztatja az állapotot. Emiatt a lehallgatás
nyomot hagy, amit a felek ki tudnak mutatni.

A legismertebb QKD-protokoll a **BB84**, amit Charles Bennett és Gilles Brassard írt le
1984-ben. Ez a feladat egy olyan szimulátort valósít meg, amin lépésről lépésre
végig lehet követni a protokoll működését, és be lehet kapcsolni egy lehallgatót is,
hogy látható legyen a hatása.

## 2. A BB84 protokoll elméleti háttere

### 2.1 A felhasznált bázisok

A BB84 két, egymáshoz képest 45°-ban elforgatott bázist használ egy qubit kódolására:

- **rektilineáris bázis (jelölés: +, ez a Z-bázis):** a 0 bitet a |0⟩, az 1 bitet az |1⟩
  állapot kódolja;
- **diagonális bázis (jelölés: ×, ez az X-bázis):** a 0 bitet a |+⟩, az 1 bitet a |−⟩
  állapot kódolja.

A két bázis "nem összeférő": ha egy qubitet az egyik bázisban készítünk elő, de a másik
bázisban mérünk, akkor a mérés eredménye teljesen véletlenszerű (50–50%), és a mérés
el is rontja az eredeti állapotot.

### 2.2 A protokoll lépései

1. **Alice** minden qubithez sorsol egy véletlen bitet (0/1) és egy véletlen bázist (+/×),
   és a bitet a választott bázisban kódolva küldi el.
2. A qubitek átmennek a **kvantumcsatornán**.
3. **Bob** minden beérkező qubithez szintén véletlen bázist választ, és abban mér.
   Ha a bázisa megegyezik azzal, amiben Alice küldött, biztosan a helyes bitet kapja;
   ha nem, akkor 50% eséllyel hibás értéket mér.
4. **Bázis-egyeztetés (sifting):** Alice és Bob egy nyilvános (de nem titkos) csatornán
   összeveti, melyik pozíciónál milyen bázist használtak. Csak azokat a pozíciókat
   tartják meg, ahol a bázis egyezett. Az itt maradt bitek alkotják a *sifted* kulcsot.
   Átlagosan a qubitek fele marad meg, mert feleannyiszor esik egybe a két véletlen bázis.
5. **Hibaarány-becslés (QBER):** a sifted kulcs egy részét feláldozzák: ezeket a biteket
   nyilvánosan összehasonlítják, és megnézik, hány százalékban térnek el. Ez a QBER
   (Quantum Bit Error Rate). Ideális, lehallgatásmentes és zajmentes esetben a QBER 0.
6. Ha a QBER egy küszöb (a programban 11%) felett van, a felek **eldobják** a kulcsot,
   mert lehallgatásra gyanakodnak. Egyébként a megmaradt bitek lesznek a közös titok.

### 2.3 Miért buktatja le a lehallgatót a QBER?

A legegyszerűbb támadás az **intercept-resend**: Eve elkapja a qubitet, véletlen bázisban
megméri, és a mért eredményt a saját bázisában küldi tovább Bobnak. A gond az, hogy Eve
nem tudja, Alice melyik bázist használta, így az esetek felében rossz bázisban mér.
Amikor rossz bázist választ, elrontja az állapotot, és onnantól Bob mérése már akkor is
hibázhat, ha Bob történetesen a "jó" bázist használta.

Egy kis számolás: azokon a pozíciókon, ahol Alice és Bob bázisa egyezik (tehát bekerül a
sifted kulcsba), Eve az esetek felében szintén ebben a bázisban mér, ekkor nincs hiba.
Az esetek másik felében Eve rossz bázisban mér, és ilyenkor Bob mérése 50% eséllyel
hibás. Így a sifted kulcson a hibaarány körülbelül **0,5 × 0,5 = 0,25**, vagyis **25%**.
Ekkora QBER egyértelműen jóval a küszöb felett van, tehát a lehallgatás kiderül.

## 3. A program felépítése és a felhasznált technológiák

A szimulátor egyetlen, önálló **HTML-fájl** (`bb84_szimulator.html`), amely
HTML + CSS + JavaScript (vanilla JS) felhasználásával készült. Nem használ külső
könyvtárat és nem igényel internetkapcsolatot, így **telepítés nélkül, dupla kattintással**
megnyitható bármelyik modern böngészőben (Chrome, Edge, Firefox) egy átlagos Windows
gépen. Ez teljesíti a kiírás "standalone futtatható" követelményét.

A kód három logikai részre tagolódik:

- **Modell:** a protokoll tiszta logikája – véletlen bit/bázis generálása (`veletlenBit`,
  `veletlenBazis`), a mérés szabálya (`mer`), Eve lehallgatása (`eveLehallgat`), Bob
  mérése (`bobMer`), a bázis-egyeztetés (`sifting`), a QBER kiszámítása (`qberSzamol`)
  és a végső kulcs előállítása (`vegsoKulcs`).
- **Nézet:** a HTML és CSS adja a vezérlőpanelt, az átviteli táblázatot, a QBER-sávot és
  a kulcs megjelenítését. A két bázist és a szereplőket színkód különbözteti meg
  (+ zöld, × kék, Eve sorai pirosak).
- **Vezérlő:** egy egyszerű állapotgép lépteti a protokollt a fázisokon keresztül
  (Alice → küldés → Eve → Bob → sifting → QBER → kulcs), és minden lépéshez magyarázó
  szöveget mutat.

### 3.1 Felhasznált könyvtárak

A program **nem használ semmilyen külső könyvtárat vagy keretrendszert**, kizárólag a
böngésző beépített képességeire támaszkodik (DOM, szabványos JavaScript). Emiatt nincs
semmilyen függőség, amit telepíteni kellene.

## 4. Használati útmutató

1. Nyisd meg a `bb84_szimulator.html` fájlt egy böngészőben (dupla kattintás).
2. A **Beállítások** panelen:
   - add meg a **qubitek számát** (alapértelmezés 12, 2 és 40 között);
   - válaszd ki a **bemenet módját**:
     - *Véletlen* – a program sorsolja Alice bitjeit és bázisait;
     - *Kézi megadás* – megjelenik két mező, ahova beírhatod Alice bitjeit
       (pl. `011010`) és bázisait (pl. `+x+x+x`);
   - jelöld be az **„Eve lehallgat”** kapcsolót, ha szeretnéd bekapcsolni a támadót.
3. Kattints a **„Protokoll indítása”** gombra.
4. A **„Következő lépés”** gombbal haladj végig a protokoll fázisain. Minden fázisnál
   egy rövid magyarázat jelenik meg, és az **Átvitel részletei** táblázat fokozatosan
   töltődik fel (Alice sorai, majd Eve sorai, majd Bob sorai, végül az egyezések).
   A **„Vissza”** gombbal visszaléphetsz, a **„Teljes lefuttatás”** gombbal rögtön a
   végeredményre ugorhatsz.
5. A **QBER** panelen látszik a mért hibaarány és a döntés (elfogadva / eldobva), a
   **Végső közös kulcs** panelen pedig a kész kulcs.

> Megjegyzés: a kézi módnál csak Alice adatait adhatjuk meg, Bob (és Eve) továbbra is
> véletlen bázist választ – a protokoll ettől működik hűen, hiszen a valóságban sem tudja
> Bob előre, mit küldött Alice.

## 5. Egy példafuttatás elemzése

**Eve nélkül (12 qubit):** a bázis-egyeztetés után átlagosan 6 bit marad meg. Mivel nincs
lehallgató és a csatorna zajmentes, Alice és Bob megtartott bitjei teljesen megegyeznek,
a QBER 0%, a döntés "Biztonságos – kulcs elfogadva". A két fél ugyanazt a kulcsot kapja.

**Eve bekapcsolva (ugyanúgy 12 qubit):** a táblázatban megjelennek Eve (piros) sorai is.
A sifted kulcson most már lesznek eltérő bitek, ezeket a program az "Egyezés" sorban
≠ jellel és piros háttérrel jelöli. A QBER tipikusan 20–30% körül alakul (elméleti
várható érték 25%), a döntés "Lehallgatás gyanúja – kulcs eldobva". Így a szimulátor
jól mutatja, hogy a lehallgatás kimutatható.

Nagyobb qubitszámnál (pl. 40) a QBER stabilabban áll be a 25% közelébe, mert kevésbé
számít a véletlen ingadozás. Kis qubitszámnál néha szerencséje van Eve-nek, és a QBER
alacsonyabb – ez is jól mutatja, miért kell a valóságban sok qubittel dolgozni.

## 6. Összegzés

A program a BB84 protokoll teljes menetét megvalósítja és láthatóvá teszi: Alice bit- és
bázisválasztását (véletlen vagy kézi módon), a kvantumcsatornát, az intercept-resend
lehallgatót, Bob mérését, a bázis-egyeztetést, a QBER-számítást és a végső kulcsot. A
protokoll lépésről lépésre léptethető, a lehallgató hatása (a megnövekedett QBER és az
eltérő bitek formájában) jól érzékelhető. A megoldás egyetlen, függőség nélküli
HTML-fájl, ami telepítés nélkül fut egy átlagos böngészős gépen.
