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
## 3. Promisiunea și domeniul de aplicare ale produsului.
## 4. Cerințe funcționale.
## 5. C4 System Context view.