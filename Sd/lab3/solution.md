# Laborator 3: Desenează limita Dashboard-ului

## 0. Analiza cerinței (pe scurt)

| Ce cere laboratorul | Puncte importante |
| --- | --- |
| 3 diagrame de arhitectură (Context, Container, Component) | Fiecare nivel „deschide" un singur element din nivelul de deasupra. Component deschide **doar** Dashboard Application. |
| 3 diagrame de secvență (A, B, C) | Participanții sunt cei din Container (nume identice peste tot). Fiecare are `alt` pentru succes, respingere și defectare, plus ipoteze dedesubt. |
| Folosește țintele de volum din Laboratorul 2 | Se folosește modelul din Lecția 3 (1,000 Utilizatori concurenți, 234 RPS stabil, 310 RPS la deschiderea pieței). **Dacă Laboratorul 2 conține alte cifre, acestea se înlocuiesc în secțiunea 1.** |
| Cel mai mic set de containere | Nu se adaugă cache, coadă, replici sau serviciu de autentificare separat: nu există dovezi că volumul le cere. |
| Furnizorul poate da date invalide, poate limita cererile sau poate fi căzut | Cerința centrală: **citirile Utilizatorului nu apelează furnizorul**. Doar un proces separat de sincronizare vorbește cu el. |

### Decizii de proiectare luate (și motivul)

1. **Market Data Collector este container separat** de Dashboard Application. Motive: (a) volumul către furnizor nu depinde de numărul de Utilizatori (1 cerere la 60 s, nu 234 RPS), (b) o defectare a furnizorului nu rupe răsfoirea, (c) doar Collector deține credențialele furnizorului, deci nu ajung niciodată în browser.
2. **Dashboard Database** este singura stocare: Stocks, ultimul preț acceptat, istoric, Watchlist-uri, starea sincronizării. Este și punctul de adevăr pentru „datele acceptate".
3. **Autentificarea rămâne în Dashboard Application** (sesiune verificată de un component `Identity Guard`). Un serviciu separat nu e justificat la 1,000 de Utilizatori.
4. **Regulă de vechime:** un preț acceptat se afișează cu ora furnizorului și eticheta „întârziat" cât timp are cel mult **30 de minute** de la ora furnizorului. Peste limită: „indisponibil". Nu se afișează niciodată zero și un preț vechi nu e prezentat ca fiind curent.

---

## 1. Volum de lucru folosit (ipoteze)

Ipoteze preluate din modelul Dashboard al Lecției 3. Cifrele pentru scrieri sunt o ipoteză a proiectării.

| Flux | Calcul | Rezultat |
| --- | --- | --- |
| Overview | `1,000 x 0.70 / 30` | 23.33 RPS |
| Stock price | `1,000 x 0.20 x 1` | 200 RPS |
| Watchlist read | `1,000 x 0.60 / 60` | 10 RPS |
| **Citiri, regim stabil** | `23.33 + 200 + 10` | **233.33 -> 234 RPS** |
| **Citiri, deschiderea pieței** | `(233.33 + 30 + 18) x 1.10` | **309.47 -> 310 RPS** |
| Scrieri Watchlist (ipoteză) | `1,000 x 0.05 / 60` stabil, vârf aprox. 3x | aprox. 1 RPS stabil, 3 RPS vârf |
| Cereri către furnizor | 1 sincronizare / 60 s (+ cel mult o reîncercare) | aprox. 1-2 cereri/minut, **independent de Utilizatori** |

Observație: Stock price este aprox. 86% din citirile în regim stabil, deci este prima ipoteză de blocaj de măsurat. Nu e un blocaj dovedit.

Ținte de calitate folosite în diagrame (exemple, de ajustat cu Lab 2):

- Citire Stock price, în interiorul sistemului, la 310 RPS: p95 <= 300 ms.
- Disponibilitate răsfoire/citire: 99.9% în 30 de zile (buget: 43 min 12 s).
- Modificare Watchlist, observată de Client: p95 <= 1,500 ms.
- Consistență Watchlist: după succes confirmat, următoarea citire a aceluiași Utilizator arată modificarea.

---

## 2. Diagrama System Context

Arată doar limita sistemului, actorii și sistemele externe. Fără părți interne.

```mermaid
flowchart LR
    user["Utilizator autentificat<br/>Person"]
    dashboard["Stock Market Dashboard<br/>Software System"]
    provider["Market Data Provider<br/>External System"]

    user -->|"Răsfoiește și filtrează Stocks, citește prețuri și istoric, administrează Watchlist privat"| dashboard
    dashboard -->|"Cere periodic date de piață întârziate"| provider
    provider -->|"Returnează date, date invalide, limitare a cererilor sau nu răspunde"| dashboard
```

---

## 3. Diagrama Container

Diagrama deschide limita sistemului: trei containere, fiecare cu o responsabilitate.

```mermaid
flowchart LR
    user["Utilizator autentificat<br/>Person"]
    provider["Market Data Provider<br/>External System"]

    subgraph boundary["Stock Market Dashboard"]
        app["Dashboard Application<br/>Container: aplicație web<br/>Servește UI și API, verifică identitatea, citește date acceptate, administrează Watchlist"]
        collector["Market Data Collector<br/>Container: proces programat<br/>Preia, validează și acceptă date de piață, deține credențialele furnizorului"]
        db[("Dashboard Database<br/>Container: bază de date<br/>Stocks, prețuri acceptate, istoric, Watchlist-uri, stare sincronizare")]
    end

    user -->|"HTTPS: răsfoiește Stocks, citește prețuri, modifică Watchlist"| app
    app -->|"Citește Stocks, prețuri, istoric și Watchlist, scrie modificări Watchlist"| db
    collector -->|"HTTPS: cere date întârziate, folosind credențiale păstrate pe server"| provider
    collector -->|"Scrie datele acceptate și starea sincronizării într-o tranzacție"| db
```

**De ce acest set minim:**

- Dashboard Application nu apelează niciodată furnizorul: o citire a Utilizatorului nu cere o cerere nouă către furnizor.
- Collector și Dashboard Application comunică **doar prin Dashboard Database**, deci nu depind unul de altul la execuție.
- Nu se adaugă cache, coadă sau replici: 310 RPS de citire și aprox. 3 RPS de scriere nu justifică încă aceste mecanisme. Lipsesc măsurătorile care le-ar susține.

---

## 4. Diagrama Component (Dashboard Application)

Se deschide numai Dashboard Application. Componentele sunt responsabilități, nu clase sau funcții. Market Data Collector nu apare, fiind container lansat separat, iar furnizorul nu comunică direct cu aplicația.

```mermaid
flowchart LR
    user["Utilizator autentificat<br/>Person"]
    db[("Dashboard Database<br/>Container")]

    subgraph app["Dashboard Application"]
        handler["Request Handler<br/>Primește cereri HTTPS, rutează și formatează răspunsuri și erori"]
        guard["Identity Guard<br/>Verifică sesiunea și oferă identitatea Utilizatorului"]
        validator["Input Validator<br/>Verifică simbolul, filtrele și parametrii"]
        browse["Stock Browse Service<br/>Răsfoire și filtrare Stocks"]
        price["Price and History Service<br/>Preț și istoric, aplică regula de vechime de 30 min"]
        watch["Watchlist Service<br/>Autorizează proprietarul și modifică Watchlist-ul"]
        gateway["Data Gateway<br/>Singurul acces la Dashboard Database"]
    end

    user -->|"HTTPS"| handler
    handler -->|"Verifică sesiunea"| guard
    handler -->|"Validează intrarea"| validator
    handler -->|"Cereri de răsfoire"| browse
    handler -->|"Cereri de preț și istoric"| price
    handler -->|"Cereri Watchlist"| watch
    browse -->|"Citește Stocks"| gateway
    price -->|"Citește prețuri acceptate"| gateway
    watch -->|"Citește și scrie Watchlist"| gateway
    gateway -->|"Interogări și tranzacții"| db
```

---

## 5. Diagrame de secvență

### A. Sincronizează datele de piață

```mermaid
sequenceDiagram
    autonumber
    participant Collector as Market Data Collector
    participant Provider as Market Data Provider
    participant DB as Dashboard Database

    Note over Collector: Pornire: timer intern la fiecare 60 s
    Collector->>DB: Citește lista Stocks cunoscute și ultima oră acceptată
    Collector->>Provider: Cere prețuri întârziate (credențiale păstrate pe server)

    alt Timeout de 5 s
        Note over Collector,Provider: O singură reîncercare după 2 s
        Collector->>Collector: Dacă reîncercarea eșuează, marchează sincronizarea eșuată
    else Limitare a cererilor (429)
        Provider-->>Collector: 429 și Retry-After
        Collector->>Collector: Nu reîncearcă înainte de Retry-After, sare ciclul curent
    else Răspuns primit
        Provider-->>Collector: Lot de prețuri cu ora furnizorului
        Collector->>Collector: Validează schema, simbolul cunoscut, prețul pozitiv, ora nu mai veche decât cea acceptată

        alt Lot invalid sau peste 20% înregistrări invalide
            Collector->>Collector: Respinge tot lotul, înregistrează motivul, nu scrie nimic în DB
        else Lot acceptabil
            Collector->>DB: Tranzacție: actualizează ultimul preț acceptat, adaugă istoric, marchează sincronizare reușită

            alt Scriere eșuată
                DB-->>Collector: Eroare, rollback complet
                Collector->>Collector: Marchează sincronizarea eșuată
            else Scriere reușită
                DB-->>Collector: Commit confirmat
                Note over Collector,DB: Din acest moment datele acceptate sunt disponibile citirilor
            end
        end
    end

    Note over Collector,DB: La orice eșec, datele acceptate anterior rămân neschimbate
```

**Ipoteze:**

- Sincronizarea rulează la 60 s. Furnizorul oferă date întârziate (aprox. 15 min), deci o frecvență mai mare nu aduce date mai noi.
- Datele devin vizibile citirilor **numai după commit**. Un lot parțial scris nu este vizibil (tranzacție atomică).
- Înregistrările invalide sunt: simbol necunoscut, preț lipsă sau <= 0, schemă greșită, oră mai veche decât cea deja acceptată. Peste 20% invalide, tot lotul este respins (probabil defect la furnizor).
- La eșec (timeout, 429, lot invalid, DB), **ultimele date acceptate rămân** și continuă să fie servite cu eticheta de întârziere, până la limita de 30 min de la ora furnizorului. Apoi citirea răspunde „indisponibil".
- Cheia furnizorului este doar în mediul serverului Collector. Nu ajunge în browser, în Dashboard Application sau în răspunsuri.
- Eșecurile se înregistrează în log și în starea sincronizării, pentru alertare.

---

### B. Citește prețul unui Stock

```mermaid
sequenceDiagram
    autonumber
    participant User as Utilizator autentificat
    participant App as Dashboard Application
    participant DB as Dashboard Database

    User->>App: Cere prețul unui Stock (simbol, sesiune)
    App->>App: Verifică sesiunea (Identity Guard)

    alt Sesiune lipsă sau expirată
        App-->>User: 401 Autentificare necesară
    else Sesiune validă
        App->>App: Validează simbolul (Input Validator)

        alt Simbol invalid
            App-->>User: 400 Simbol invalid
        else Simbol valid
            App->>DB: Citește Stock-ul și ultimul preț acceptat

            alt Defectare a stocării sau timeout
                DB-->>App: Eroare
                App-->>User: 503 Temporar indisponibil (fără zero, fără preț presupus)
            else DB răspunde
                DB-->>App: Stock și ultimul preț acceptat, dacă există

                alt Stock inexistent
                    App-->>User: 404 Stock necunoscut
                else Preț lipsă
                    App-->>User: Preț indisponibil
                else Preț mai vechi de 30 min
                    App-->>User: Preț indisponibil, date prea vechi
                else Preț în limita de vechime
                    App-->>User: Preț, ora furnizorului și eticheta Întârziat
                end
            end
        end
    end

    Note over User,DB: Nicio cerere către Market Data Provider în acest flux
```

**Ipoteze:**

- **O citire a Utilizatorului NU cere o cerere nouă către furnizor.** Citește doar ultimul preț acceptat din Dashboard Database, scris de Market Data Collector.
- Limita maximă de vechime este 30 min față de ora furnizorului (aprox. 15 min întârziere normală + marjă pentru 1-2 sincronizări ratate).
- Un preț vechi peste limită nu se afișează ca preț curent, iar lipsa unui preț nu se afișează ca zero.
- Ținta: p95 <= 300 ms la 310 RPS. O defectare a stocării dă 503, nu un rezultat inventat.

---

### C. Modifică un Watchlist privat

Operația exemplu este adăugarea unui Stock în Watchlist-ul propriu.

```mermaid
sequenceDiagram
    autonumber
    participant User as Utilizator autentificat
    participant App as Dashboard Application
    participant DB as Dashboard Database

    User->>App: Adaugă un Stock în Watchlist (simbol, sesiune)
    App->>App: Verifică sesiunea (Identity Guard)

    alt Sesiune lipsă sau expirată
        App-->>User: 401 Autentificare necesară
    else Sesiune validă
        App->>App: Validează simbolul și cererea (Input Validator)

        alt Intrare invalidă sau Stock inexistent
            App-->>User: 400 sau 404, nicio modificare
        else Intrare validă
            App->>App: Autorizează: proprietarul Watchlist-ului este Utilizatorul din sesiune

            alt Încercare neautorizată (Watchlist al altui Utilizator)
                App-->>User: 403 Interzis, nicio scriere, evenimentul se înregistrează
            else Autorizat
                App->>DB: Scrie intrarea în Watchlist-ul Utilizatorului din sesiune (idempotent)

                alt Defectare a stocării
                    DB-->>App: Eroare, nicio modificare păstrată
                    App-->>User: 503 Nu s-a salvat, încearcă din nou
                else Scriere reușită
                    DB-->>App: Commit confirmat
                    Note over App,DB: PUNCT DE SUCCES: doar după commit confirmat
                    App-->>User: 200 Stock adăugat
                end
            end
        end
    end

    Note over User,DB: Următoarea citire a aceluiași Utilizator
    User->>App: Citește Watchlist-ul
    App->>DB: Citește Watchlist-ul Utilizatorului din sesiune
    DB-->>App: Lista, inclusiv Stock-ul adăugat
    App-->>User: Watchlist cu Stock-ul nou
```

**Ipoteze:**

- Identitatea proprietarului vine **numai din sesiune**, nu din parametrii cererii. Dacă cererea indică Watchlist-ul altui Utilizator, răspunsul este 403 și nu se scrie nimic.
- Verificările se fac în ordinea: identitate -> intrare -> permisiune -> scriere.
- Succesul se raportează **doar după commit confirmat**. La defectare, Utilizatorul primește eroare, nu succes, și nu rămâne o modificare parțială.
- Adăugarea este idempotentă: a adăuga același Stock a doua oară nu creează duplicat și nu dă eroare.
- **Consistență (read-your-writes):** după un succes confirmat, următoarea citire a aceluiași Utilizator arată Stock-ul adăugat. Se citește din aceeași bază, fără replică sau cache care ar putea întârzia.
- Watchlist-ul este privat: nicio citire nu returnează date ale altui Utilizator.
- Ștergerea din Watchlist urmează același flux, cu aceeași regulă: la citirea următoare, Stock-ul nu mai apare.

---
