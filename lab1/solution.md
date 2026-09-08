## 1. Cercetarea produsului

```text
Cum ajută produsele existente un Utilizator să urmărească informațiile despre
piață și ce părți aparțin primei versiuni a acestui Dashboard?
```
| Produs            | Utilizatorul probabil și obiectivul său | Model reutilizabil           |
| ----------------- | --------------------------------------- | ---------------------------- |
|Google Finance     | Investitor individual, retail, care vrea să monitorizeze gratuit valoarea portofoliului și prețurile, fără cont de brokeraj și fără să tranzacționeze din aplicație.                                        |Separă clar watchlist (simboluri urmărite, fără deținere) de portofoliu (poziții deținute, introduse manual), fiecare cu preț curent și variație zilnică. Datele sunt „ca atare", fără sfaturi de investiții.|
|TradingView        |	Trader activ care urmărește simultan multe simboluri și vrea să fie notificat automat când prețul atinge un anumit nivel, fără să verifice manual fiecare grafic.                                         |Alerte atașate unui simbol sau unui watchlist întreg, care rulează pe server și notifică independent de sesiunea utilizatorului.                              |

Decizie de domeniu de aplicare influențată de cercetare: Dashboard-ul preia din Google Finance separarea watchlist/portofoliu ca model central al primei versiuni. Modelul de alerte din TradingView este util, dar prea complex pentru MVP (necesită job-uri de fundal și notificări) — devine un obiectiv exclus explicit, nu o presupunere ascunsă.

## 2. Părți interesate și actori.
| Parte interesată | Motivație | Influență | Motiv |
|---|---|---|---|
| Investitor (Utilizator) | Ridicată | Scăzută | Depinde direct de produs pentru a-și vedea portofoliul, dar nu decide roadmap-ul |
| Furnizor de date de piață | Scăzută | Ridicată | Nu îi pasă de succesul Dashboard-ului, dar poate opri accesul la API sau schimba limitele |
| Proprietarul produsului | Ridicată | Ridicată | Decide domeniul de aplicare, prioritățile și bugetul |
| Autoritatea de reglementare a pieței financiare | Scăzută | Ridicată | Nu folosește produsul, dar poate impune reguli despre afișarea datelor financiare |

| Motivație | Influență scăzută | Influență ridicată |
|---|---|---|
| Ridicată | Investitor → Consultă | Proprietarul produsului → Gestionează îndeaproape |
| Scăzută | — | Furnizor de date de piață, Autoritatea de reglementare → Menține satisfacția |

| Candidat | Parte interesată? | Actor direct? | Motiv |
|---|---|---|---|
| Investitor | Da | Da | Adaugă poziții, citește prețuri și valoarea portofoliului |
| Furnizor de date de piață | Da | Da (sistem extern) | Furnizează prețuri curente la cerere |
| Proprietarul produsului | Da | Nu | Ia decizii despre produs, dar nu interacționează cu fluxul |
| Autoritatea de reglementare | Da | Nu | Poate impune limite, dar nu folosește direct Dashboard-ul |
## 3. Promisiunea și domeniul de aplicare ale produsului

**Promisiunea produsului:**

> Personal Investment Dashboard ajută un Investitor individual să își vadă portofoliul de investiții într-un singur loc, cu prețuri curente și performanță, astfel încât să poată urmări starea investițiilor sale fără să treacă printre mai multe surse de date.

**Obiective (5):**

1. Arată Investitorului valoarea curentă a portofoliului și câștigul/pierderea totală.
2. Permite Investitorului să adauge o poziție deținută (simbol, cantitate, preț mediu de achiziție).
3. Arată prețul curent și variația zilnică pentru fiecare poziție din portofoliu.
4. Permite Investitorului să creeze un watchlist cu simboluri de interes, separat de portofoliul deținut.
5. Arată Investitorului clar când prețul unei poziții este învechit sau indisponibil.

**Obiective excluse (3):**

1. Nu execută tranzacții de cumpărare/vânzare — nu se conectează la niciun brokeraj pentru trading.
2. Nu oferă recomandări de investiții, semnale de trading sau sfaturi financiare personalizate.
3. Nu acceptă alerte automate de preț, mai multe monede/conturi sau consolidare fiscală în prima versiune.

**Constrângeri:**
- Prima versiune afișează un singur portofoliu per Investitor.
- Pozițiile se introduc manual — nu există import automat din brokeraj.

**Ipoteze:**
- Furnizorul de date de piață returnează un rezultat utilizabil la cerere.
- Un preț introdus manual de Investitor (cantitate, preț de achiziție) este suficient de corect pentru calculul valorii portofoliului.

---

## 4. Cerințe funcționale

### DASH-1
**Obiectivul actorului:** Investitorul trebuie să poată adăuga o poziție deținută în portofoliu.

**Povestea de Utilizator:**
> Ca Investitor, vreau să adaug o poziție (simbol, cantitate, preț mediu de achiziție), astfel încât să văd valoarea ei în portofoliul meu.

**Definiția de Făcut:**
- Poziția apare în lista portofoliului cu simbolul, cantitatea și prețul de achiziție introduse.
- Dacă simbolul introdus nu este recunoscut, sistemul arată o eroare clară și nu adaugă poziția.
- Poziția nu este vizibilă pentru alt Investitor.

### DASH-2
**Obiectivul actorului:** Investitorul trebuie să afle cât valorează portofoliul său acum.

**Povestea de Utilizator:**
> Ca Investitor, vreau să văd valoarea curentă și câștigul/pierderea totală a portofoliului meu, astfel încât să știu cum performează investițiile mele.

**Definiția de Făcut:**
- Valoarea portofoliului se calculează din prețul curent al fiecărei poziții deținute.
- Dacă prețul curent lipsește pentru o poziție, aceasta este arătată explicit ca „preț indisponibil" și exclusă din total — nu tratată ca zero.
- Câștigul/pierderea se arată atât ca valoare absolută, cât și procentuală.

### DASH-3
**Obiectivul actorului:** Investitorul trebuie să poată urmări simboluri fără să le dețină.

**Povestea de Utilizator:**
> Ca Investitor, vreau să adaug simboluri într-un watchlist separat de portofoliul deținut, astfel încât să urmăresc acțiuni fără să le dețin.

**Definiția de Făcut:**
- Un simbol din watchlist arată prețul curent și variația zilnică.
- Un simbol poate fi în watchlist fără să fie și în portofoliu.
- Investitorul poate elimina un simbol din watchlist.

### DASH-4
**Obiectivul actorului:** Investitorul trebuie să știe când datele de preț nu sunt de încredere.

**Povestea de Utilizator:**
> Ca Investitor, vreau să văd când prețul unei poziții este învechit, astfel încât să nu iau decizii pe baza unor date greșite.

**Definiția de Făcut:**
- Fiecare preț afișat arată ora ultimei actualizări.
- Dacă Furnizorul de date de piață nu răspunde, sistemul arată explicit „date indisponibile" pentru pozițiile afectate — nu ultimul preț cunoscut ca fiind curent.
- Restul portofoliului rămâne vizibil chiar dacă o singură poziție are date indisponibile.

### DASH-5
**Obiectivul actorului:** Investitorul trebuie să poată corecta o poziție introdusă greșit.

**Povestea de Utilizator:**
> Ca Investitor, vreau să șterg sau să editez o poziție existentă, astfel încât portofoliul meu să reflecte corect ce dețin.

**Definiția de Făcut:**
- Editarea cantității sau prețului de achiziție recalculează imediat valoarea și câștigul/pierderea poziției.
- Ștergerea unei poziții o elimină din calculul valorii totale a portofoliului.
- O poziție nu poate avea cantitate negativă; sistemul respinge o astfel de editare cu un mesaj clar.

---

## 5. C4 System Context view

**Sistemul de interes:** Personal Investment Dashboard

**Actori umani direcți:**
- Investitor — adaugă/editează poziții și simboluri de watchlist; citește valoarea portofoliului și prețurile curente.

**Sisteme externe conectate direct:**
- Furnizor de date de piață — furnizează prețuri curente și variații zilnice la cererea Dashboard-ului.

**Descrierea relațiilor:**

| De la                      | Către                      | Scop                                                                                                   |
| ---------------------------- | ---------------------------- | --------------------------------------------------------------------------------------------------------- |
| Investitor                    | Dashboard                     | Adaugă/editează poziții deținute și simboluri de watchlist                                                |
| Dashboard                     | Investitor                    | Arată valoarea portofoliului, câștig/pierdere, prețuri curente și starea datelor (curent/învechit/indisponibil) |
| Dashboard                     | Furnizor de date de piață     | Solicită prețul curent și variația zilnică pentru simbolurile din portofoliu și watchlist                 |
| Furnizor de date de piață     | Dashboard                     | Returnează prețuri și date de piață, sau nu răspunde (caz de defectare)                                    |

**Limita sistemului:**

| În interiorul limitei Dashboard                                                                | În afara limitei Dashboard                                                        |
| -------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| Arată portofoliul, watchlist-ul și valoarea calculată                                              | Investitorul furnizează cantitatea și prețul de achiziție                            |
| Interpretează rezultatul furnizorului de date și previne un „succes fals" când prețul lipsește      | Furnizorul de date de piață deține prețurile sursă și decide disponibilitatea lor    |
| Arată clar Investitorului starea datelor (curent/învechit/indisponibil)                            | —                                                                                     |

---