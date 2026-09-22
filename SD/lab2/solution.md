# Laboratorul 2: Cuantifică citirile Dashboard-ului — Soluție

## 1. Cerințe de calitate

### Latența citirilor

```text
Măsurare: de la primirea cererii de către Dashboard până la trimiterea
răspunsului (latență în interiorul sistemului). Nu include rețeaua sau
randarea în Browser.
Percentilă: p50 pentru „imediat", p95 pentru limita de 2 secunde din perioadele
aglomerate.
```

> La volumul de lucru țintă în regim stabil, latența citirii **Stock price**,
> măsurată de la primirea cererii de către Dashboard până la trimiterea
> prețului curent etichetat cu ora furnizorului, trebuie să respecte **p50 ≤
> 300 ms** și **p95 ≤ 2,000 ms**.
>
> La ținta de capacitate pentru deschiderea pieței, aceeași latență trebuie să
> respecte **p95 ≤ 2,000 ms**.
>
> O cerere care depășește 2,000 ms este o defectare a țintei de latență,
> indiferent dacă rezultatul final conținea un preț corect, un preț etichetat
> ca învechit sau „indisponibil". Un rezultat indisponibil rapid respectă
> ținta de latență, dar rămâne evaluat separat pentru disponibilitate.

**Ipoteză:** clientul nu a precizat o țintă p50; „prețurile trebuie să pară
imediate" este interpretat ca p50 ≤ 300 ms. Limita explicită de 2 secunde din
perioadele aglomerate este aplicată ca p95, conform recomandării lecției de a
folosi o percentilă pentru „majoritatea" citirilor.

Aceeași structură se aplică celorlalte citiri (Overview, Filter, Search,
History, Watchlist), cu o țintă mai relaxată pentru History (citire mai rar
folosită, nu are nevoie de „imediat"):

> La volumul de lucru țintă în regim stabil, latența citirilor **Overview,
> Filter, Search și Watchlist** trebuie să respecte p50 ≤ 300 ms și p95 ≤
> 1,500 ms. Latența citirii **History** trebuie să respecte p95 ≤ 2,500 ms.

### Disponibilitate bazată pe timpul de funcționare

**Definirea orelor de tranzacționare:** ședința regulată NYSE/Nasdaq, 09:30–16:00
America/New_York, luni–vineri, exclusiv sărbătorile bursiere. Presupunem 21 de
zile de tranzacționare într-un interval de 30 de zile calendaristice (≈ 30 ×
5/7, rotunjit).

```text
Ore de tranzacționare / 30 zile   = 21 zile x 6.5 h = 136.5 h = 491,400 s
Restul zilei / 30 zile            = 2,592,000 s - 491,400 s   = 2,100,600 s
(2,592,000 s = 30 x 24 x 60 x 60)
```

> În fiecare interval de 30 de zile, Dashboard-ul (citirile de piață) poate fi
> folosit cel puțin **99.9%** din timpul măsurat în **orele de tranzacționare**.
>
> În fiecare interval de 30 de zile, Dashboard-ul poate fi folosit cel puțin
> **99.5%** din timpul măsurat în **restul zilei** (inclusiv weekend-uri și
> sărbători bursiere).

**Justificare pentru ținte diferite:** în orele de tranzacționare, un preț
indisponibil sau vechi are efect direct asupra unei decizii de investiție
posibile chiar acum; o defectare este mai costisitoare pentru Utilizator. În
restul zilei, prețurile sunt oricum statice (piața e închisă), iar
Investitorul consultă de regulă performanța, nu ia decizii imediate — o țintă
cu o cifră 9 mai puțin este justificată prin costul de operare mai mic, nu
prin obișnuință.

```text
Buget indisponibilitate, ore de tranzacționare (30 zile):
  491,400 s x (1 - 0.999) = 491.4 s  ~= 8 min 11 s

Buget indisponibilitate, restul zilei (30 zile):
  2,100,600 s x (1 - 0.995) = 10,503 s ~= 2 h 55 min 3 s
```

### Consistență

**Preț de piață (actualitate):**

> Returnează ultimul preț de piață acceptat împreună cu ora furnizorului și o
> etichetă de întârziere, cât timp vechimea sa este ≤ **20 de minute** de la
> ora furnizorului. Peste 20 de minute, sau când niciun preț nu a fost primit
> vreodată pentru acel Stock, citirea Stock price returnează **indisponibil**.
> Nu arată zero și nu afirmă că un preț vechi este curent.

**Ipoteză:** limita de 20 de minute = întârzierea așteptată a furnizorului
(~15 minute, dată de client) + o marjă de 5 minute pentru sincronizare și
eventuale eșecuri temporare ale Furnizorului.

**Watchlist (citirea propriilor scrieri):**

> După ce Dashboard-ul confirmă adăugarea sau eliminarea unui simbol din
> Watchlist de către un Utilizator, următoarea citire a Watchlist-ului făcută
> de **același Utilizator, în aceeași sesiune**, trebuie să arate rezultatul
> modificării. Watchlist-ul este date proprii, cu volum mic de scriere —
> garanția aleasă este cea mai puternică (citirea propriilor scrieri,
> imediată), nu vechime limitată.

### Debit (RPS)

> La deschiderea pieței, pentru grupul de **30,000 Utilizatori** concurenți,
> Dashboard-ul finalizează cel puțin **10,296** de citiri acceptabile pe
> secundă (Overview, Filter, Stock price, History, Watchlist, Search), în timp
> ce țintele de latență, disponibilitate și consistență de mai sus continuă
> să fie respectate. Un rezultat indisponibil rapid sau un preț etichetat greșit
> nu este o citire acceptabilă pentru această țintă.

(Vezi tabelele de la secțiunile 2 și 3 pentru țintele echivalente la 300 și
3,000 de Utilizatori.)

---

## 2. Estimări RPS în regim stabil

```text
RPS = concurrent Users x participating share x actions per User / seconds
```

| Citire                    | 300 Utilizatori                    | 3,000 Utilizatori                   | 30,000 Utilizatori                    |
| -------------------------- | ----------------------------------- | ------------------------------------ | --------------------------------------- |
| Overview (70% / 30 s)       | `300 x 0.70 / 30` = 7               | `3,000 x 0.70 / 30` = 70             | `30,000 x 0.70 / 30` = 700              |
| Filter (50% x 3/60 s)       | `300 x 0.50 x 3/60` = 7.5           | `3,000 x 0.50 x 3/60` = 75           | `30,000 x 0.50 x 3/60` = 750            |
| Stock price (20% / 1 s)     | `300 x 0.20 / 1` = 60               | `3,000 x 0.20 / 1` = 600             | `30,000 x 0.20 / 1` = 6,000             |
| History (20% / 300 s)       | `300 x 0.20 / 300` = 0.2            | `3,000 x 0.20 / 300` = 2             | `30,000 x 0.20 / 300` = 20              |
| Watchlist (60% / 60 s)      | `300 x 0.60 / 60` = 3               | `3,000 x 0.60 / 60` = 30             | `30,000 x 0.60 / 60` = 300              |
| Search (10% x 3/60 s)       | `300 x 0.10 x 3/60` = 1.5           | `3,000 x 0.10 x 3/60` = 15           | `30,000 x 0.10 x 3/60` = 150            |
| **Total în regim stabil**  | `7+7.5+60+0.2+3+1.5` = **79.2**     | `70+75+600+2+30+15` = **792**        | `700+750+6,000+20+300+150` = **7,920**  |

Nu se rotunjește totalul în regim stabil — doar ținta finală de capacitate
(vezi secțiunea 3). Stock price generează ~76% din traficul stabil la toate
cele trei niveluri (60/79.2, 600/792, 6,000/7,920) — primul flux de investigat
pentru latență și debit, nu o dovadă de blocaj.

---

## 3. Estimări RPS la deschiderea pieței

Comportament suplimentar la deschidere (peste traficul stabil de mai sus, fără
suprapunere — traficul stabil deja include reîmprospătarea normală a Overview
și Watchlist; fluxul de deschidere este trafic în plus, pe același interval):

```text
Overview peak  = Users x 0.30 / 10
Watchlist peak = Users x 0.30 x 0.60 / 10
```

| Calcul la deschiderea pieței                     | 300 Utilizatori | 3,000 Utilizatori | 30,000 Utilizatori |
| ------------------------------------------------- | ---------------- | ------------------ | -------------------- |
| Trafic stabil de citire                            | 79.2              | 792                 | 7,920                 |
| Flux suplimentar Overview (`U x 0.30/10`)          | `300x0.3/10`=9    | `3,000x0.3/10`=90   | `30,000x0.3/10`=900   |
| Flux suplimentar Watchlist (`U x 0.30x0.60/10`)    | `300x0.18/10`=5.4 | `3,000x0.18/10`=54  | `30,000x0.18/10`=540  |
| **Subtotal la deschiderea pieței**                 | 93.6              | 936                 | 9,360                 |
| Marjă de capacitate de 10%                         | `93.6x1.10`=102.96| `936x1.10`=1,029.6  | `9,360x1.10`=10,296   |
| **Țintă rotunjită în sus la deschiderea pieței**   | **103**           | **1,030**           | **10,296**            |

---

## 4. Estimări de stocare

### Cercetează și definește un Stock

Un „Stock" (acțiune comună) reprezintă o cotă-parte în proprietatea unei
companii, cu drept de vot și pretenție reziduală asupra activelor — spre
deosebire de un ADR (certificat ce reprezintă acțiuni ale unei companii
non-americane, deținute de o bancă depozitară din SUA) sau un ETF (fond
tranzacționat la bursă, ce deține un coș de active)


**Decizii de domeniu de aplicare (prima versiune):**

- **Piețe acceptate:** doar acțiuni comune listate pe **NYSE** și **Nasdaq**
  (SUA). Alte burse și piețe OTC sunt excluse din prima versiune —
  reglementări, monede și fusuri orare diferite ar complica ținta unică de
  disponibilitate bazată pe orele de tranzacționare definită mai sus.
- **Tipuri de instrumente:** doar acțiuni comune (*common stock*), inclusiv
  ADR-uri tranzacționate ca acțiuni obișnuite pe NYSE/Nasdaq. **Excluse:**
  acțiuni preferențiale, ETF-uri și fonduri — motivul e coerent cu obiectivul
  exclus deja stabilit (fără consolidare pe mai multe clase de active în
  prima versiune).
- **Instrumente delistate/inactive:** excluse din domeniul de aplicare
  acceptat. O poziție dintr-un Watchlist al cărei simbol este delistat arată
  „indisponibil", nu date istorice înghețate.
- **Număr de Stocks acceptate:** aproximativ **3,450** listări pe Nasdaq și
  **2,200** pe NYSE, adică **~5,650** de acțiuni comune.
  Domeniul de aplicare al produsului este setat la **5,700 Stocks** (rotunjit
  în sus, cu o marjă pentru listări noi înainte de următoarea sincronizare a
  domeniului de aplicare).

### Definește datele sincronizate

| Nevoia produsului                          | Date stocate                                                                 |
| -------------------------------------------- | ------------------------------------------------------------------------------ |
| Search, Filter, identitatea Stock-ului       | simbol, nume companie, bursă, sector/industrie, monedă, stare (activ), dată listare |
| Overview, Stock detail (preț curent)         | ultimul preț acceptat, variație zilnică absolută și procentuală, ora furnizorului, ora sincronizării |
| Filter (interval capitalizare, prag variație)| capitalizare de piață, minim/maxim 52 de săptămâni, volum mediu               |
| Price history                                | punct zilnic OHLCV (open/high/low/close/volum) per Stock                       |

Frecvența de sincronizare presupusă: prețul curent și variația zilnică se
resincronizează la fiecare cerere de citire care depășește pragul de
actualitate de 20 de minute (vezi secțiunea 1); datele de referință (Filter,
identitate) se resincronizează o dată pe zi. Traficul de cereri către
Furnizor nu este estimat în această secțiune, conform cerinței laboratorului.

### Calculează stocarea

```text
raw storage = record count x average bytes per record
history record count = supported Stocks x history points per Stock per day x retained days
```

**Ipoteză de păstrare:** un punct zilnic OHLCV per Stock, păstrat **730 de
zile (2 ani)**, suficient pentru graficele de istoric ale primei versiuni fără
date intraday.

| Set de date              | Decizia despre produs și perioada de păstrare                          | Calculul numărului de înregistrări         | Octeți per înregistrare | Stocare brută            |
| ------------------------- | -------------------------------------------------------------------------- | --------------------------------------------- | -------------------------- | --------------------------- |
| Date de referință Stock   | 1 înregistrare per Stock acceptat, fără istoric de versiuni                | `5,700`                                       | ~150 B                     | `5,700x150` = 855,000 B     |
| Cele mai recente prețuri  | 1 înregistrare per Stock, suprascrisă la fiecare sincronizare               | `5,700`                                       | ~80 B                      | `5,700x80` = 456,000 B      |
| Price history             | 1 punct OHLCV/zi per Stock, păstrat 730 de zile                            | `5,700 x 1 x 730` = 4,161,000                 | ~70 B                      | `4,161,000x70` = 291,270,000 B |
| Alte date (Filter)        | capitalizare, min/max 52 săpt., volum mediu — 1 înregistrare per Stock     | `5,700`                                       | ~45 B                      | `5,700x45` = 256,500 B      |
| **Total**                 |                                                                              |                                                |                             | **292,837,500 B**           |

```text
292,837,500 B / 1,048,576 ~= 279.3 MiB  (~0.27 GiB)
```

Price history reprezintă ~99.4% din stocarea brută (291,270,000 /
292,837,500). Cu o fereastră de păstrare fixă de 730 de zile și purjarea
punctelor mai vechi, stocarea de istoric se stabilizează în jur de 280 MiB și
nu crește nelimitat — creșterea zilnică netă este de doar `5,700 x 70 B` =
399,000 B (~0.38 MiB) adăugați, echilibrați de purjarea punctelor mai vechi
de 730 de zile odată ce fereastra e plină.

Această estimare exclude indexurile, replicile, jurnalele, copiile de
siguranță și datele proprii ale Utilizatorului (conturi, Watchlist-uri) — nu
este un plan de cumpărare a stocării.

---

## 5. Găsește și analizează posibilele blocaje

| Calitate        | Posibil blocaj                                   | Dovezi din acest laborator                                                                                     | Efect posibil                                                                                   | Ce trebuie măsurat în continuare                                                    |
| ---------------- | --------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| Latență          | Calea de citire Stock price                        | Stock price este ~76% din traficul stabil la toate cele trei niveluri (secțiunea 2) și are cea mai strictă țintă de latență (p95 ≤ 2 s) | p95 al Stock price poate domina experiența Utilizatorului dacă fiecare citire repetă lucru costisitor | Un test de încărcare doar pe calea Stock price, la RPS-ul din secțiunea 3, măsurând p50/p95 reale |
| Consistență      | Sincronizarea prețului de la Furnizor              | Regula de actualitate (secțiunea 1) cere „indisponibil" peste 20 de minute; nu există încă dovezi despre cât de des Furnizorul depășește întârzierea de 15 minute promisă | Prețuri afișate ca „învechite"/„indisponibile" mai des decât se așteaptă Investitorul, sau — mai grav — un preț vechi afișat ca fiind curent | Măsurarea reală a distribuției vechimii răspunsurilor Furnizorului pe o perioadă reprezentativă |
| Debit            | Calea de citire Stock price la deschiderea pieței  | Stock price rămâne cel mai mare flux absolut chiar și după adăugarea fluxurilor de deschidere (secțiunea 3); ținta de 10,296 RPS la 30,000 Utilizatori nu este încă un rezultat măsurat | Sistemul poate porni cereri fără să le finalizeze în limitele de calitate la ținta de capacitate      | Un test de încărcare cu rată fixă de sosire (de tipul constant-arrival-rate) la 10,296 RPS |
| Disponibilitate  | Dependența de Furnizorul de date de piață          | Toate citirile care afișează preț (Overview, Stock price, History, Filter) depind de un răspuns valid al Furnizorului; Furnizorul e un sistem extern, în afara limitei Dashboard-ului (System Context, Laboratorul 1) | O întrerupere sau o limitare de rată la Furnizor poate consuma rapid bugetul de 8 min 11 s din orele de tranzacționare | Rata de eșec/timeout a apelurilor către Furnizor, separat de disponibilitatea internă a Dashboard-ului |

---
