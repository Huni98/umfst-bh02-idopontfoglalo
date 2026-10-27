# A rakoncátlan időpontfoglaló – hibariport

Repó: https://github.com/<felhasznalo>/umfst-bh02-idopontfoglalo
Készítette: <név>, <csoport>

## 1. hiba
- **Hiba leírása és reprodukálása:** Ugyanarra az időpontra (pl. 09:00) többször is lehet foglalni, a „már foglalt” üzenet sosem jelenik meg. Kézi tesztnél, ha kétszer küldjük be a 09:00-t, mindkettő bekerül a listába.
- **Gyökérok (sor + magyarázat):** Az `app.js` 12-14. sorában a `foglalt(ido)` függvény a `foglalasok.indexOf(ido)` metódust használja. Mivel a `foglalasok` tömb objektumokat tartalmaz (`{ nev, ido }`), a stringre való keresés `indexOf`-fal mindig sikertelen (-1), így a feltétel sosem teljesül.
- **Javítás (commit link):** https://github.com/Huni98/umfst-bh02-idopontfoglalo/commit/ee3b0cd0c9f6efd337d828db179629a0675ff5da
  - Előtte: `return foglalasok.indexOf(ido) !== -1;`
  - Utána: `return foglalasok.some(function(f) { return f.ido === ido; });`
- **Hogyan tesztelted (kézi tesztlépések, konzolos ellenőrzés vagy egységteszt):** Kézi teszttel böngészőben. Felvettem egy 09:00-es időpontot, majd megpróbáltam újra felvenni. A felület kiírta, hogy "Ez az időpont már foglalt.", és a második foglalás nem került be.
- **Tanulság / érintett alapelv:** Típusismeret és megfelelő adatstruktúra-bejárás. Objektumokat tartalmazó tömbökben iterációs metódusokkal (pl. `some`, `find`, `findIndex`) kell keresni, nem használható az egyszerű értékekre kitalált `indexOf`.

## 2. hiba
- **Hiba leírása és reprodukálása:** A rendszer a napi 8 fős limit ellenére a 9. beteget is engedi rögzíteni, a "Ma már nincs szabad hely" üzenet csak a 10. próbálkozásnál jelenik meg. Kézi teszteléssel 8 foglalás leadása után a 9. is sikeresen megjelenik a listában.
- **Gyökérok (sor + magyarázat):** Az `app.js` 42. sorában a feltételvizsgálat hibás: `foglalasok.length > MAX_NAPI`. Ha a tömb hossza már 8, a `8 > 8` hamis, így a kód továbbmegy és beleteszi a 9. elemet. Off-by-one (határérték) hiba történt.
- **Javítás (commit link):** https://github.com/Huni98/umfst-bh02-idopontfoglalo/commit/58d56206b2fad4e3a50a4e4e778c741e3e5e2605
  - Előtte: `if (foglalasok.length > MAX_NAPI)`
  - Utána: `if (foglalasok.length >= MAX_NAPI)`
- **Hogyan tesztelted (kézi tesztlépések, konzolos ellenőrzés vagy egységteszt):** Kézi teszteléssel felvettem 8 időpontot. Amikor a 9.-et próbáltam hozzáadni, a felület kiírta a hibaüzenetet, és a foglalás nem rögzült.
- **Tanulság / érintett alapelv:** Határértékek pontos kezelése és relációs jelek helyes használata. Limitek vizsgálatakor mindig át kell gondolni, hogy szigorú egyenlőtlenség (`>`) vagy megengedő (`>=`) szükséges-e.

## 3. hiba
- **Hiba leírása és reprodukálása:** A felvett foglalások nem időrendben jelennek meg a felületen. Kézi teszteléssel: ha felveszünk egy 10:00-es, majd egy 08:30-as időpontot, a 08:30-as helytelenül a 10:00-es után marad a listában.
- **Gyökérok (sor + magyarázat):** Az `app.js` 17. sorában a `sort` összehasonlító függvényének visszatérési értéke egy boolean kifejezés (`a.ido > b.ido`). A JavaScript `sort()` metódusa negatív vagy pozitív számokat (esetleg nullát) vár a helyes sorrend megállapításához, logikai értékekkel a rendezés kiszámíthatatlan lesz.
- **Javítás (commit link):** https://github.com/Huni98/umfst-bh02-idopontfoglalo/commit/b8805d6c96c2b9bee9471f0ba0b10eb6417b35d8
  - Előtte: `foglalasok.sort(function (a, b) { return a.ido > b.ido; });`
  - Utána: `foglalasok.sort(function (a, b) { return a.ido.localeCompare(b.ido); });`
- **Hogyan tesztelted (kézi tesztlépések, konzolos ellenőrzés vagy egységteszt):** Kézi teszteléssel böngészőben. Összevissza (nem kronológiai sorrendben) adtam meg időpontokat, és a lista minden sikeres rögzítés után helyes, növekvő időrendi sorrendbe rendeződött.
- **Tanulság / érintett alapelv:** A JavaScript `sort()` függvényének megfelelő használata. Egyedi összehasonlító függvény írásakor számot kell visszaadni, stringek lexikografikus és időrendi rendezésére pedig az `a.localeCompare(b)` egy tiszta és szabványos megoldás.

## 4. hiba
- **Hiba leírása és reprodukálása:** XSS (Cross-Site Scripting) sebezhetőség. Ha a név mezőbe kártékony HTML/JS kódot írunk (pl. `<img src=x onerror="document.body.style.background='red'">`), az lefut az oldal betöltésekor és módosítja a DOM-ot (a példában piros lesz a háttér).
- **Gyökérok (sor + magyarázat):** Az `app.js` 25. sorában az `innerHTML += ...` szerkezettel a felhasználói bemenet (`f.nev`) közvetlenül, tisztítás és escape-elés nélkül kerül a DOM-ba, így a böngésző végrehajtható kódként értelmezi.
- **Javítás (commit link):** https://github.com/Huni98/umfst-bh02-idopontfoglalo/commit/22e13b5fa5fb138b3d91a31b8d49811aa123c26d
  - Előtte: `lista.innerHTML += '<div class="sor"><span>' + f.ido + ' – ' + f.nev + '</span>' + ...`
  - Utána: 
    ```javascript
    var tisztaNev = f.nev.replace(/</g, "&lt;").replace(/>/g, "&gt;");
    lista.innerHTML += '<div class="sor"><span>' + f.ido + ' – ' + tisztaNev + '</span>' + ...
    ```
- **Hogyan tesztelted (kézi tesztlépések, konzolos ellenőrzés vagy egységteszt):** Kézi teszteléssel megadtam a fenti `<img>` taget betegnévként. A javítás után az oldal nem váltott pirosra, a beadott kód pedig egyszerű, ártalmatlan szövegként (stringként) jelent meg a listában.
- **Tanulság / érintett alapelv:** Biztonság (Security). Felhasználótól származó adatban sosem bízunk. `innerHTML` használatakor az adatot kötelező "escape"-elni (kiküszöbölni a vezérlőkaraktereket), vagy biztonságosabb DOM API-kat (pl. `textContent`, `innerText`) kell használni.

## 5. hiba
- **Hiba leírása és reprodukálása:** A "Törlés" gombra nem lehet Tab-bal navigálni, illetve billentyűzettel (Enter vagy Szóköz megnyomásával) nem lehet törölni az elemeket. Csak egérrel működik.
- **Gyökérok (sor + magyarázat):** Az `app.js` 26. sorában a törlés gomb egy `<span>` elemként (`<span class="torol">`) van leképezve. A `<span>` nem interaktív elem, így nem kerül bele a Tab-sorrendbe, és natívan nem figyeli a billentyűzetes aktivációt.
- **Javítás (commit link):** https://github.com/Huni98/umfst-bh02-idopontfoglalo/commit/efe611b04337435f304653f4af351c03e6ccb32d
  - Előtte: `'<span class="torol" data-i="' + i + '">Törlés</span></div>';`
  - Utána: `'<button type="button" class="torol" data-i="' + i + '">Törlés</button></div>';`
- **Hogyan tesztelted (kézi tesztlépések, konzolos ellenőrzés vagy egységteszt):** Kézi teszteléssel billentyűzetet használva (Tab-bejárás). Az új foglalás után Tab-bal elértem a Törlés gombot (látható fókusz), majd Entert nyomva az elem sikeresen törlődött.
- **Tanulság / érintett alapelv:** Szemantika előbb / Akadálymentesítés (a11y). Interaktív, kattintható vezérlőkhöz mindig natívan interaktív elemeket (`<button>`, `<a>`) kell használni a `<div>` vagy `<span>` helyett, hogy a billentyűzetes és képernyőolvasós felhasználók is kezelni tudják azokat.

## 6. hiba
- **Hiba leírása és reprodukálása:** Teljesítményprobléma. Sok (50+) foglalás esetén a lista renderelése nagyon belassul, a felület "megfagy" egy pillanatra minden új elem hozzáadásakor.
- **Gyökérok (sor + magyarázat):** Az `app.js` 25. sorában az `innerHTML +=` művelet egy `forEach` cikluson belül történik. Ez azt eredményezi, hogy a böngésző minden iterációban újra és újra elemzi és felépíti a DOM fát (O(N^2) komplexitás). A közvetlen DOM-manipuláció cikluson belül az egyik legnagyobb teljesítménygyilkos hiba.
- **Javítás (commit link):** https://github.com/Huni98/umfst-bh02-idopontfoglalo/commit/0c70f83acef11f7d2ebfc164c713ae669f338844
  - Előtte: `lista.innerHTML += ...` (a cikluson belül folyamatosan frissítve a DOM-ot)
  - Utána: Egy `var htmlStr = '';` változóba gyűjtöm a HTML szöveget a ciklusban, majd a ciklus után egyetlen utasítással (`lista.innerHTML = htmlStr;`) módosítom a DOM-ot.
- **Hogyan tesztelted (kézi tesztlépések, konzolos ellenőrzés vagy egységteszt):** DevTools konzolból feltöltöttem a tömböt 100 elemmel (`for(let i=0; i<100; i++) foglalasok.push({...})`), majd meghívtam a `render()` függvényt. A módosítás után a megjelenítés azonnali volt, a Performance panelen a "Long Task" (hosszú feladat) megszűnt.
- **Tanulság / érintett alapelv:** Teljesítményoptimalizálás (Performance) és DOM manipuláció. A DOM olvasása és írása drága folyamat. Ciklusban sosem módosítjuk a DOM-ot közvetlenül; az adatokat memóriában építjük fel (stringként vagy DocumentFragment-ként), és csak a legvégén, egy lépésben adjuk át a böngészőnek.

## Összegzés
- **Mit tanultam:** A stabil JavaScript alkalmazások titka az adatszerkezetek (objektumok tömbje) megfelelő bejárása, a beépített metódusok helyes használata (`some`, `localeCompare`), valamint a biztonsági rések (XSS) és a teljesítménybeli szűk keresztmetszetek (cikluson belüli DOM módosítás) elkerülése. Ezen felül dinamikus HTML generálásnál is figyelni kell a szemantikára (interaktív gombok).
- **Melyik alapelv sérült a leggyakrabban:** A JavaScript „Best Practices” (legjobb gyakorlatok) és a biztonságos kódolás elve. A hibák jelentős része (XSS sebezhetőség, hibás rendezési logika, nem hatékony DOM-kezelés) abból fakadt, hogy a kód nem kellő gondossággal kezelte a felhasználói adatokat és a böngésző API-kat. 
