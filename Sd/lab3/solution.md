# Laboratorul 3: Desenează limita Dashboard-ului — Soluție

## 1. Diagrama System Context

**Explicație:** diagrama arată sistemul ca o singură cutie, cu cine îl folosește și cu ce sistem extern comunică. Investitorul citește și modifică datele, iar Furnizorul de date de piață trimite prețurile întârziate. Furnizorul poate răspunde cu date invalide, poate limita cererile sau poate să nu răspundă, de aceea relația este etichetată cu toate aceste rezultate. Părțile interne nu apar.

```mermaid
flowchart LR
    I["Investitor<br/>Person"]
    D["Personal Investment Dashboard<br/>Software System"]
    F["Furnizor de date de piață<br/>External System"]

    I -->|"Adaugă și editează poziții, administrează Watchlist, citește portofoliul, prețurile și istoricul"| D
    D -->|"Arată valoarea portofoliului, câștig/pierdere, prețuri cu ora furnizorului și starea datelor"| I
    D -->|"Cere periodic prețuri și date de piață întârziate"| F
    F -->|"Returnează date, date invalide, limitare a cererilor sau nu răspunde"| D
```

---

## 2. Diagrama Container

**Explicație:** limita sistemului este deschisă și apar trei containere, fiecare cu o singură responsabilitate. Dashboard Application servește Investitorul. Market Data Collector este singurul care vorbește cu furnizorul și păstrează credențialele acestuia. Dashboard Database ține datele acceptate și datele private. Cele două aplicații comunică doar prin baza de date, deci o citire a Investitorului nu cere niciodată o cerere nouă către furnizor, iar o defectare a furnizorului nu oprește răsfoirea.

```mermaid
flowchart LR
    I["Investitor<br/>Person"]
    F["Furnizor de date de piață<br/>External System"]

    subgraph boundary["Personal Investment Dashboard"]
        app["Dashboard Application<br/>Container: aplicație web<br/>Servește UI și API, verifică identitatea, citește date acceptate, administrează portofoliul și Watchlist-ul"]
        collector["Market Data Collector<br/>Container: proces programat<br/>Preia, validează și acceptă date de piață, deține credențialele furnizorului"]
        db[("Dashboard Database<br/>Container: bază de date<br/>Stocks, prețuri acceptate, istoric, poziții, Watchlist-uri, stare sincronizare")]
    end

    I -->|"HTTPS: răsfoiește Stocks, citește prețuri și portofoliu, modifică poziții și Watchlist"| app
    app -->|"Citește Stocks, prețuri, istoric, poziții și Watchlist, scrie modificări ale pozițiilor și Watchlist-ului"| db
    collector -->|"HTTPS: cere date întârziate, cu credențiale păstrate pe server"| F
    collector -->|"Scrie datele acceptate și starea sincronizării într-o tranzacție"| db
```

---

## 3. Diagrama Component (Dashboard Application)

**Explicație:** se deschide numai Dashboard Application și se arată responsabilitățile principale din interiorul ei: primirea cererilor, verificarea identității și a intrării, citirea Stocks și a prețurilor, administrarea portofoliului și a Watchlist-ului, accesul la date. Doar Investitorul și Dashboard Database comunică direct cu aceste componente. Market Data Collector și furnizorul nu apar, deoarece Collector este container lansat separat, iar furnizorul nu comunică direct cu aplicația.

```mermaid
flowchart LR
    I["Investitor<br/>Person"]
    db[("Dashboard Database<br/>Container")]

    subgraph app["Dashboard Application"]
        handler["Request Handler<br/>Primește cererile HTTPS, le rutează și formatează răspunsuri și erori"]
        guard["Identity Guard<br/>Verifică sesiunea și oferă identitatea Investitorului"]
        validator["Input Validator<br/>Verifică simbolul, filtrele, cantitatea și prețul de achiziție"]
        browse["Stock Browse Service<br/>Search și Filter pe Stocks acceptate"]
        price["Price and History Service<br/>Preț și istoric, stările curent, învechit, indisponibil după regula de 20 min"]
        portfolio["Portfolio Service<br/>Poziții, valoare, câștig/pierdere, exclude pozițiile cu preț indisponibil"]
        watch["Watchlist Service<br/>Autorizează proprietarul și modifică Watchlist-ul"]
        gateway["Data Gateway<br/>Singurul acces la Dashboard Database"]
    end

    I -->|"HTTPS"| handler
    handler -->|"Verifică sesiunea"| guard
    handler -->|"Validează intrarea"| validator
    handler -->|"Search și Filter"| browse
    handler -->|"Preț și istoric"| price
    handler -->|"Portofoliu"| portfolio
    handler -->|"Watchlist"| watch
    portfolio -->|"Cere prețul fiecărei poziții"| price
    browse -->|"Citește Stocks"| gateway
    price -->|"Citește prețuri acceptate și istoric"| gateway
    portfolio -->|"Citește și scrie poziții"| gateway
    watch -->|"Citește și scrie Watchlist"| gateway
    gateway -->|"Interogări și tranzacții"| db
```

---

## 4. Diagrame de secvență

### A. Sincronizează datele de piață

**Explicație:** fluxul arată cum ajung datele noi în sistem. Un timer pornește sincronizarea, Collector cere datele, le validează și le scrie în bază într-o singură tranzacție. Timeout-ul, limitarea cererilor, datele invalide și defectarea bazei sunt tratate fără a modifica datele acceptate anterior. Datele noi devin vizibile abia după commit.

```mermaid
sequenceDiagram
    autonumber
    participant Collector as Market Data Collector
    participant Furnizor as Furnizor de date de piață
    participant DB as Dashboard Database

    Note over Collector: Pornire: timer intern la fiecare 60 s
    Collector->>DB: Citește lista Stocks acceptate și ora furnizorului ultimului lot acceptat
    Collector->>Furnizor: Cere prețurile întârziate pentru toate Stocks acceptate (credențiale păstrate pe server)

    alt Timeout de 5 s
        Note over Collector,Furnizor: O singură reîncercare după 2 s
        Collector->>Collector: Dacă reîncercarea eșuează, marchează sincronizarea eșuată
    else Limitare a cererilor (429)
        Furnizor-->>Collector: 429 și Retry-After
        Collector->>Collector: Nu reîncearcă înainte de Retry-After, sare ciclul curent
    else Răspuns primit
        Furnizor-->>Collector: Lot de prețuri cu ora furnizorului
        Collector->>Collector: Validează schema, simbolul acceptat, prețul pozitiv, ora furnizorului nu mai veche decât cea acceptată

        alt Lot invalid sau peste 20% înregistrări invalide
            Collector->>Collector: Respinge tot lotul, înregistrează motivul, nu scrie nimic în DB
        else Lot acceptabil
            Note over Collector: Înregistrările invalide sunt excluse, acele Stocks păstrează prețul acceptat anterior
            Collector->>DB: Tranzacție: actualizează ultimul preț acceptat, ora furnizorului și ora sincronizării, marchează sincronizare reușită

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

- Sincronizarea rulează la 60 s pentru toate cele 5,700 Stocks, într-un singur lot. Furnizorul oferă date întârziate cu aproximativ 15 min, deci o frecvență mai mare nu aduce date mai noi.
- Datele devin vizibile citirilor **numai după commit**. Un lot parțial scris nu este vizibil.
- Sunt invalide: simbol necunoscut, preț lipsă sau mai mic sau egal cu zero, schemă greșită, oră a furnizorului mai veche decât cea deja acceptată. Peste 20% invalide, tot lotul este respins.
- **La eșec** (timeout, limitare, lot invalid, defectare a bazei), datele acceptate anterior rămân neschimbate. Ele se servesc mai departe cu eticheta potrivită vechimii (curent, învechit, apoi indisponibil după 20 min). Nu se înlocuiește nimic cu zero.
- Cheia furnizorului există doar în mediul Market Data Collector. Nu ajunge în browser, în Dashboard Application sau în răspunsuri.
- Eșecurile se înregistrează în log și în starea sincronizării.

---

### B. Citește prețul unui Stock

**Explicație:** fluxul arată ce se întâmplă când Investitorul cere un preț. Se verifică întâi identitatea și simbolul, apoi se citește ultimul preț acceptat din bază. Rezultatul afișat depinde de vechimea lui: curent, învechit sau indisponibil. Lipsa prețului sau o defectare a stocării dau un rezultat explicit, niciodată zero. Citirea nu apelează furnizorul.

```mermaid
sequenceDiagram
    autonumber
    participant Investitor as Investitor
    participant App as Dashboard Application
    participant DB as Dashboard Database

    Investitor->>App: Cere prețul unui Stock (simbol, sesiune)
    App->>App: Verifică sesiunea (Identity Guard)

    alt Sesiune lipsă sau expirată
        App-->>Investitor: 401 Autentificare necesară
    else Sesiune validă
        App->>App: Validează simbolul (Input Validator)

        alt Simbol invalid
            App-->>Investitor: 400 Simbol invalid
        else Simbol valid
            App->>DB: Citește Stock-ul și ultimul preț acceptat

            alt Defectare a stocării sau timeout
                DB-->>App: Eroare
                App-->>Investitor: 503 Temporar indisponibil (fără zero, fără preț presupus)
            else DB răspunde
                DB-->>App: Stock și ultimul preț acceptat, dacă există

                alt Stock inexistent în domeniul acceptat
                    App-->>Investitor: 404 Stock necunoscut
                else Preț lipsă sau Stock delistat
                    App-->>Investitor: Preț indisponibil
                else Vechime peste 20 min de la ora furnizorului
                    App-->>Investitor: Preț indisponibil, date prea vechi
                else Vechime între 16 și 20 min
                    App-->>Investitor: Preț cu ora furnizorului și starea Învechit
                else Vechime sub 16 min
                    App-->>Investitor: Preț cu ora furnizorului și eticheta Întârziat 15 min
                end
            end
        end
    end

    Note over Investitor,DB: Nicio cerere către Furnizorul de date de piață în acest flux
```

**Ipoteze:**

- **O citire a Investitorului NU cere o cerere nouă către furnizor.** Citește doar ultimul preț acceptat din Dashboard Database, scris de Market Data Collector.
- Un preț este acceptat pentru afișare cel mult 20 min de la ora furnizorului, altfel este indisponibil. Pragul de 16 min separă „curent" de „învechit" (15 min întârziere normală plus un ciclu de sincronizare de 60 s).
- Fiecare preț afișat poartă ora furnizorului. Lipsa unui preț nu se afișează ca zero, iar un preț vechi nu se prezintă ca fiind curent.
- Același tipar se aplică fiecărei poziții din portofoliu: o poziție cu preț indisponibil este arătată explicit și exclusă din total, iar restul portofoliului rămâne vizibil.
- O defectare a stocării dă 503, nu un rezultat inventat.

---

### C. Modifică un Watchlist privat

**Explicație:** fluxul arată cum se modifică în siguranță o listă privată. Se verifică în ordine identitatea, intrarea și permisiunea, abia apoi se scrie în bază. O încercare asupra listei altui Investitor este respinsă. Succesul se raportează doar după ce scrierea este confirmată, iar următoarea citire a aceluiași Investitor arată modificarea.

Operația exemplu: adăugarea unui Stock în Watchlist-ul propriu. Eliminarea urmează același flux.

```mermaid
sequenceDiagram
    autonumber
    participant Investitor as Investitor
    participant App as Dashboard Application
    participant DB as Dashboard Database

    Investitor->>App: Adaugă un Stock în Watchlist (simbol, sesiune)
    App->>App: Verifică sesiunea (Identity Guard)

    alt Sesiune lipsă sau expirată
        App-->>Investitor: 401 Autentificare necesară
    else Sesiune validă
        App->>App: Validează simbolul și cererea (Input Validator)

        alt Intrare invalidă sau Stock în afara domeniului acceptat
            App-->>Investitor: 400 sau 404, nicio modificare
        else Intrare validă
            App->>App: Autorizează: proprietarul Watchlist-ului este Investitorul din sesiune

            alt Încercare neautorizată (Watchlist al altui Investitor)
                App-->>Investitor: 403 Interzis, nicio scriere, evenimentul se înregistrează
            else Autorizat
                App->>DB: Scrie intrarea în Watchlist-ul Investitorului din sesiune (idempotent)

                alt Defectare a stocării
                    DB-->>App: Eroare, nicio modificare păstrată
                    App-->>Investitor: 503 Nu s-a salvat, se poate reîncerca
                else Scriere reușită
                    DB-->>App: Commit confirmat
                    Note over App,DB: PUNCT DE SUCCES: doar după commit confirmat
                    App-->>Investitor: 200 Stock adăugat
                end
            end
        end
    end

    Note over Investitor,DB: Următoarea citire a aceluiași Investitor, în aceeași sesiune
    Investitor->>App: Citește Watchlist-ul
    App->>DB: Citește Watchlist-ul Investitorului din sesiune
    DB-->>App: Lista, inclusiv Stock-ul adăugat
    App-->>Investitor: Watchlist cu Stock-ul nou, fiecare preț cu starea lui
```

**Ipoteze:**

- Identitatea proprietarului vine **numai din sesiune**, nu din parametrii cererii. Dacă cererea indică Watchlist-ul altui Investitor, răspunsul este 403 și nu se scrie nimic.
- Verificările se fac în ordinea: identitate, intrare, permisiune, scriere.
- Succesul se raportează **doar după commit confirmat**. La defectare, Investitorul primește eroare, nu succes, și nu rămâne o modificare parțială.
- Adăugarea este idempotentă: același Stock adăugat a doua oară nu creează duplicat și nu dă eroare.
- După un succes confirmat, următoarea citire a aceluiași Investitor, în aceeași sesiune, arată modificarea. Scrierea și citirea folosesc aceeași bază, fără replică sau cache care ar putea întârzia rezultatul.
- Watchlist-ul este privat: nicio citire nu returnează datele altui Investitor.

---
