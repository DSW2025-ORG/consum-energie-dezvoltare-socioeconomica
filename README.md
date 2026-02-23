# Flux de date energetice (EIA & WDI)

## Introducere

Acest proiect are ca scop construirea unui flux de date reproductibil pentru analiza relației dintre indicatori energetici și indicatori socio-economici, folosind date publice provenite din surse oficiale. Datele energetice sunt colectate din **Energy Information Administration (EIA API v2)**, iar ulterior vor fi combinate cu date socio-economice din **World Development Indicators (WDI)**.

Proiectul este structurat pe etape clare: colectarea datelor, curățarea și standardizarea acestora, integrarea surselor și analiza finală. Prezentul raport documentează fiecare etapă, explicând metodele utilizate și deciziile tehnice relevante.

---

## Pasul 1 – Colectarea datelor energetice (EIA)
#### Responsabil: Cătălina Minciună

### Scopul etapei

Această etapă are ca obiectiv colectarea și structurarea datelor energetice provenite exclusiv din **Energy Information Administration (EIA)**. Datele obținute aici vor reprezenta baza de referință temporală pentru etapele următoare ale proiectului, inclusiv pentru colectarea și integrarea datelor socio-economice din WDI.

### Sursa datelor

Datele energetice sunt obținute din **EIA API v2**, endpoint-ul `international/data`, care oferă serii temporale anuale pentru indicatori energetici la nivel de țară.

- **API**: EIA API v2  
- **Frecvență**: anuală  
- **Perioadă solicitată**: 2019–2024  
- **Autentificare**: cheie API stocată local într-un fișier `credentials.py` (neinclus în repository)

În practică, disponibilitatea datelor diferă între indicatori și țări, iar nu toate seriile conțin valori pentru toți anii solicitați.

### Țări analizate

Analiza se concentrează pe un set de țări din Europa Centrală și de Est, identificate prin cod ISO3 / `countryRegionId`:

- România (ROU)  
- Bulgaria (BGR)  
- Ungaria (HUN)  
- Polonia (POL)  
- Cehia (CZE)  
- Slovacia (SVK)

### Indicatori energetici selectați

Pentru fiecare țară sunt colectați următorii indicatori EIA, identificați prin `productId`:

- **44** – Primary energy consumption  
- **47** – Energy intensity  
- **4008** – CO₂ emissions  

Acești indicatori acoperă consumul energetic, eficiența energetică și impactul asupra mediului și sunt relevanți pentru analize comparative între țări și în timp.

### Metodologia de colectare

Pentru a evita duplicarea codului și pentru a asigura consistența interogărilor către API, colectarea datelor este realizată printr-o **funcție reutilizabilă**, care:

- trimite cereri către EIA API pentru fiecare `productId`  
- aplică filtrele de bază (țări, frecvență, perioadă)  
- normalizează răspunsul JSON într-un tabel pandas  
- păstrează doar coloanele relevante pentru etapele următoare  

Datele sunt descărcate separat pentru fiecare indicator și apoi concatenate într-un singur dataset brut (`eia_raw`), fără a elimina inițial seriile paralele sau unitățile multiple.

### Structura inițială a datelor

Unitatea de analiză utilizată este **țară × an × indicator**. Dataset-ul este organizat în format *long (tidy data)*, cu următoarele câmpuri:

- `countryRegionId` – țara  
- `year` – anul  
- `productId` – indicatorul energetic  
- `value` – valoarea raportată  
- `unit` – unitatea de măsură  
- `activityId` – identificator de serie (metadată EIA)

Acest format facilitează curățarea ulterioară și integrarea cu alte surse de date.

### Dificultăți întâmpinate în etapa EIA

În procesul de colectare a datelor din EIA au fost identificate mai multe dificultăți importante:

- **Lipsa completitudinii temporale**: deși perioada solicitată a fost 2019–2024, pentru majoritatea indicatorilor datele sunt disponibile doar pentru anii **2020–2023**. Anii 2019 și 2024 lipsesc sau sunt incompleți pentru multe țări.
- **Serii și unități multiple**: același indicator energetic este raportat în mai multe unități de măsură (ex. MTOE, TJ, QBtu) și prin serii diferite (`activityId`), ceea ce face ca datele să conțină observații multiple pentru aceeași combinație țară–an–indicator.
- **Necesitatea normalizării**: aceste diferențe de unități și serii au făcut ca procesul de normalizare și selecție a unei singure observații relevante să fie confuz și să necesite o etapă separată de curățare și standardizare.

Aceste probleme sunt abordate explicit în etapa următoare a proiectului, dedicată curățării și agregării datelor EIA.

### Rezultatul etapei de colectare

La finalul acestei etape, datele energetice sunt salvate într-un singur fișier CSV, în format long, care păstrează toate seriile disponibile pentru transparență:

- `eia_data.csv`

Acest fișier va fi utilizat ca intrare pentru etapa de **curățare și standardizare a datelor EIA**, precum și ca referință temporală pentru colectarea datelor WDI în pasul următor.

## Pasul 2 – Colectarea datelor socio-economice (WDI)
#### Responsabil: Nadejda Galamaga (draft inițial – colectare WDI); Valeria Ianițchi (integrarea finală în pipeline și armonizarea cu EIA)

### Scopul etapei

Această etapă are ca obiectiv colectarea indicatorilor socio-economici din **World Development Indicators (WDI)**, necesari pentru analiza relației dintre dezvoltarea economică, consumul energetic și tranziția energetică. Datele WDI sunt pregătite într-un format compatibil cu datele energetice colectate anterior din EIA, pentru a permite integrarea ulterioară.

### Sursa datelor

Datele sunt obținute prin **World Bank API (WDI)**, care oferă serii temporale anuale pentru un număr mare de indicatori socio-economici.

- **API**: World Bank API (WDI)  
- **Frecvență**: anuală  
- **Format**: JSON  
- **Perioadă analizată**: 2020–2023  

Perioada analizată a fost aleasă în funcție de **disponibilitatea datelor din EIA**, pentru a asigura consistența temporală între cele două surse.

### Țări analizate

Sunt utilizate aceleași țări ca în etapa EIA, pentru a permite integrarea directă a dataset-urilor:

- România (ROU)  
- Bulgaria (BGR)  
- Ungaria (HUN)  
- Polonia (POL)  
- Cehia (CZE)  
- Slovacia (SVK)

### Indicatori WDI selectați

Au fost selectați indicatori socio-economici relevanți pentru analiza tranziției energetice:

- **PIB per capita** – `NY.GDP.PCAP.CD`  
- **Consum de energie per capita** – `EG.USE.PCAP.KG.OE`  
- **Ponderea energiei regenerabile** – `EG.FEC.RNEW.ZS`  
- **Acces la electricitate** – `EG.ELC.ACCS.ZS`  

Acești indicatori oferă informații despre nivelul de dezvoltare economică, consumul energetic, accesul la energie și progresul către surse regenerabile.

### Metodologia de colectare

Pentru fiecare indicator WDI, datele sunt descărcate separat prin API-ul World Bank. Procesul de colectare include:

- interogarea API-ului pentru fiecare indicator selectat  
- descărcarea datelor în format JSON  
- stocarea temporară a răspunsurilor brute  
- filtrarea observațiilor pentru țările și anii de interes  

Pentru a evita cod duplicat, descărcarea este realizată printr-o funcție dedicată care returnează răspunsul JSON al API-ului.

### Normalizarea datelor WDI

Datele JSON sunt transformate într-un tabel pandas în format **long (tidy data)**, unde fiecare rând reprezintă o observație unică definită prin:

- `countryRegionId` – țara  
- `year` – anul  
- `indicator` – indicatorul socio-economic  
- `value` – valoarea indicatorului  

În această etapă:
- câmpurile sunt curățate și redenumite pentru consistență cu datele EIA  
- valorile sunt convertite la tip numeric  
- datele sunt concatenate într-un singur dataset (`wdi_long`)  

### Verificări rapide și calitatea datelor

Pentru a valida corectitudinea dataset-ului WDI, au fost realizate verificări de bază:

- dimensiunea dataset-ului  
- anii disponibili  
- țările incluse  
- existența duplicatelor pe cheia (țară, an, indicator)  
- identificarea valorilor lipsă  

Nu au fost identificate duplicate pe cheia principală. Pentru unii indicatori (ex. ponderea energiei regenerabile) există valori lipsă pentru anumite țări sau ani, care sunt păstrate ca `NaN` în această etapă.

### Observații și limitări

- Disponibilitatea datelor WDI diferă între indicatori, ceea ce conduce la valori lipsă pentru anumite combinații țară–an–indicator.
- Perioada de analiză a fost limitată la **2020–2023**, pentru a fi compatibilă cu datele disponibile din EIA.
- Nu se aplică în această etapă completarea valorilor lipsă sau agregări suplimentare.

Aceste aspecte vor fi tratate în etapa următoare a proiectului, dedicată curățării, agregării și integrării finale a datelor.

### Rezultatul etapei de colectare WDI

La finalul acestei etape, datele WDI sunt salvate într-un fișier CSV, în format long, pregătit pentru integrarea cu datele energetice EIA:

- `wdi_data.csv`

## Pasul 3 – Curățarea, alinierea și combinarea datelor (EIA + WDI)
#### Responsabil: Cătălina Minciună

### Scopul etapei

Scopul acestei etape este curățarea datelor colectate din sursele EIA și WDI, alinierea acestora pe aceeași unitate de analiză și combinarea într-un singur dataset final, coerent și pregătit pentru analiza exploratorie și statistică.

### Unitatea de analiză

Unitatea de analiză utilizată este **țară × an** (`countryRegionId`, `year`).  
Această alegere este justificată de natura anuală a indicatorilor din ambele surse și permite comparabilitatea directă între indicatorii energetici și cei socio-economici.

După alinierea surselor, dataset-ul final conține un panel echilibrat pentru șase țări și perioada **2020–2023**.

### Date de intrare

În această etapă sunt utilizate două seturi de date rezultate din pașii anteriori:

- **Date EIA** (`01_eia_data_student1.csv`) – indicatori energetici în format long  
- **Date WDI** (`02_wdi_data_student2.csv`) – indicatori socio-economici în format long  

Ambele seturi folosesc aceleași chei principale: `countryRegionId` și `year`.

### Verificări inițiale

Înainte de curățare și transformare, au fost realizate verificări preliminare pentru ambele dataset-uri:

- verificarea tipurilor de date (în special pentru variabila `year`);
- verificarea setului de țări și a intervalului de ani disponibili;
- identificarea duplicatelor pe cheile principale:
  - (`countryRegionId`, `year`, `productId`) pentru EIA;
  - (`countryRegionId`, `year`) pentru WDI.

Aceste verificări au confirmat existența unui număr mare de duplicate în datele EIA, cauzate de raportarea acelorași indicatori în mai multe unități și serii.

### Curățarea datelor EIA

Datele EIA pot conține mai multe observații pentru aceeași combinație (**țară, an, indicator**), deoarece:

- același indicator este raportat în **unități diferite** (ex. MTOE, QBtu, TJ);
- există **serii diferite**, identificate prin `activityId`.

Pentru a obține un dataset coerent, a fost selectată **o singură serie standard pentru fiecare indicator EIA**, prin alegerea explicită a unei unități și a unui `activityId` reprezentativ.  
După această selecție, fiecare combinație (țară × an × indicator) este reprezentată de o singură observație, eliminând complet duplicatele.

### Transformarea datelor EIA în format wide

După curățare, datele EIA au fost transformate din format *long* în format *wide*, folosind pivotarea pe:

- index: `countryRegionId`, `year`
- coloane: `productId`
- valori: `value`

Coloanele rezultate au fost redenumite pentru claritate (ex. `eia_primary_energy_44`, `eia_energy_intensity_47`, `eia_co2_emissions_4008`).

### Curățarea și transformarea datelor WDI

Datele WDI au fost inițial colectate în format long și au fost transformate în format wide, cu o coloană distinctă pentru fiecare indicator socio-economic.

Pentru indicatorul **ponderea energiei regenerabile**, au fost identificate valori lipsă pentru anumite țări și ani. Acestea au fost completate prin **interpolare liniară la nivel de țară**, o metodă adecvată pentru serii temporale scurte și continue.

### Alinierea și verificarea cheilor

După transformarea ambelor seturi de date în format wide, a fost verificată consistența cheilor (`countryRegionId`, `year`) între EIA și WDI.  
Nu au fost identificate combinații prezente într-o sursă și absente în cealaltă, ceea ce confirmă o aliniere corectă a celor două dataset-uri.

### Combinarea seturilor de date EIA și WDI

Seturile de date EIA și WDI au fost combinate printr-un **join de tip inner** pe cheile comune (`countryRegionId`, `year`).  
Această abordare asigură păstrarea doar a observațiilor pentru care există date complete în ambele surse, evitând introducerea valorilor lipsă artificiale.

### Dificultăți întâmpinate în etapa de curățare

În această etapă au fost identificate mai multe dificultăți tipice procesului de curățare a datelor:

- **Număr mare de duplicate în datele EIA**, cauzate de existența mai multor unități și serii pentru același indicator;
- **Confuzie inițială în alegerea seriei „corecte”**, fiind necesară explorarea unităților și a valorilor pentru a selecta o variantă reprezentativă;
- **Verificarea repetată a cheilor**, pentru a evita pierderea observațiilor în timpul pivotării și al îmbinării dataset-urilor;
- **Gestionarea valorilor lipsă din WDI**, care a necesitat o decizie metodologică (interpolare) justificată de natura seriilor.

Aceste dificultăți au evidențiat importanța testării pas cu pas a codului și a verificărilor intermediare înainte de obținerea dataset-ului final.

### Dataset-ul final

Dataset-ul final rezultat conține:

- cheile: `countryRegionId`, `year`;
- indicatori energetici EIA;
- indicatori socio-economici WDI.

Acesta reprezintă un **panel curat**, fără duplicate și fără valori lipsă, pregătit pentru analiza exploratorie a datelor și pentru etapele de feature engineering și analiză statistică.

Dataset-ul final a fost salvat în format CSV:
- `cleaning_aggregation.csv`


## Pasul 4 – Analiza datelor exploratorie (EDA)
### Responsabil: Nadejda Galamaga

### Scopul etapei

Scopul acestei etape este investigarea dataset-ului final obținut după curățare și combinarea datelor EIA și WDI. Analiza exploratorie permite înțelegerea distribuției indicatorilor, identificarea unor tipare sau anomalii, verificarea corelațiilor și fundamentarea etapelor de modelare ulterioare.

### Metodologia EDA

### Pentru explorarea dataset-ului s-au aplicat următoarele tehnici:

#### 1. Statistici descriptive

- Calcularea valorilor minime, maxime, medii, mediane și deviației standard pentru fiecare indicator.
- Identificarea valorilor lipsă rămase și a distribuției lor pe țară și an.

### 2. Vizualizări grafice

- Serie temporală pentru fiecare indicator: grafice liniare (line plots) pentru evoluția consumului energetic, intensității energetice, emisiilor de CO₂, PIB per capita, consum energetic per capita și ponderea energiei regenerabile.
- Compararea țărilor: grafice de tip barplot și boxplot pentru a vizualiza diferențele între țări pentru anumiți indicatori.
- Corelații între indicatori: matrice de corelație și heatmap pentru a observa relațiile între indicatorii EIA și cei socio-economici.
- Distribuții individuale: histograme și grafice de densitate pentru a evalua variabilitatea și forma distribuției fiecărui indicator.

### 3. Analiza relațiilor între indicatori

- Verificarea corelației dintre consumul energetic și PIB per capita, precum și între ponderea energiei regenerabile și consumul de energie per capita.
- Observarea tendințelor comune între țări și identificarea eventualelor valori outlier.

### 4. Interactivitate și consistență
- Graficele au fost realizate folosind biblioteci Python (matplotlib și seaborn), cu titluri explicite, axe etichetate și scheme de culori consistente pentru fiecare indicator și țară.
- Valorile lipsă au fost marcate explicit pentru a evita interpretări eronate.

## Observații principale din EDA
- Există diferențe semnificative între țări în ceea ce privește consumul energetic și PIB per capita, reflectând niveluri diferite de dezvoltare economică.
- Corelațiile dintre consumul de energie per capita și PIB per capita sunt pozitive, indicând o legătură între dezvoltarea economică și consumul energetic.
- Ponderea energiei regenerabile a fost mai ridicată în țările cu politici energetice mai avansate, însă valorile lipsă au necesitat completări prin interpolare.
- Distribuțiile anumitor indicatori sunt ușor asimetrice, ceea ce sugerează că mediana poate fi un indicator mai robust decât media pentru aceste cazuri.

## Dataset utilizat

Analiza exploratorie a fost realizată pe dataset-ul final curat, rezultat din combinarea datelor EIA și WDI, salvat în:
- `cleaning_aggregation.csv`

## Concluzii intermediare

EDA a oferit o înțelegere clară asupra structurii dataset-ului, a relațiilor dintre indicatori și a diferențelor între țări. Rezultatele acestei analize au fundamentat deciziile pentru eventuale transformări suplimentare, selecția variabilelor pentru modelare și interpretarea corectă a valorilor în etapele de analiză statistică sau vizualizare finală.

## Pasul 5. Feature Engineering și Analiză Statistică
### Responsabil: Valeria Ianitchi

### 5.1 Scopul etapei

Etapa de feature engineering a avut ca scop transformarea indicatorilor economici și energetici brut în variabile comparabile între țări și ani, care să permită analiza relației dintre consumul de energie, dezvoltarea economică și impactul asupra mediului. Această etapă a fost necesară pentru a reduce diferențele de scară dintre variabile și pentru a surprinde diferențele structurale dintre economii.

### 5.2 Indicatori de intensitate și variabile per capita

Pentru a analiza eficiența utilizării energiei și impactul asupra mediului, au fost construiți indicatori de intensitate, precum intensitatea emisiilor de CO₂ raportată la consumul de energie per capita. Acești indicatori permit evaluarea gradului de poluare asociat utilizării energiei, independent de dimensiunea economiei sau a populației. În paralel, utilizarea variabilelor exprimate per capita a facilitat comparații corecte între țări cu niveluri diferite de dezvoltare economică.

### 5.3 Transformări logaritmice

Variabilele economice și energetice relevante (PIB per capita, consum de energie per capita și emisii de CO₂) au fost supuse transformărilor logaritmice, acolo unde a fost relevant. Această transformare reduce asimetria distribuțiilor, diminuează influența valorilor extreme și permite interpretarea coeficienților estimați în modelele OLS ca elasticități, oferind o interpretare economică mai intuitivă a relațiilor analizate.

### 5.4 Gruparea țărilor după nivelul de dezvoltare

Pentru a permite comparații între economii aflate la niveluri diferite de dezvoltare, țările au fost clasificate în trei grupuri în funcție de PIB-ul per capita: nivel scăzut, mediu și ridicat. Această grupare a stat la baza aplicării testelor statistice de tip ANOVA și a permis evaluarea diferențelor dintre grupuri în ceea ce privește consumul de energie și intensitatea emisiilor de CO₂.

### 5.5 Structura sistemului energetic

Analiza a inclus și variabile care descriu structura sistemului energetic, precum ponderea energiei din surse regenerabile și accesul la electricitate. Acești indicatori reflectă infrastructura energetică, politicile publice și gradul de tranziție energetică și au fost utilizați atât în analize descriptive, cât și ca variabile explicative în modelele econometrice.

### 5.6 Metode statistice aplicate

Pe baza feature-urilor construite au fost aplicate corelații Spearman, regresii OLS simple și multiple, precum și teste ANOVA pentru comparații între grupuri de țări. Aceste metode au permis evaluarea relațiilor dintre variabilele cheie și identificarea diferențelor structurale dintre economii.

### 5.7 Limitări

Rezultatele analizei trebuie interpretate în contextul mai multor limitări. În primul rând, dimensiunea redusă a eșantionului, determinată atât de numărul limitat de țări analizate, cât și de perioada scurtă de timp (2020–2023), restrânge puterea statistică a modelelor utilizate și capacitatea de generalizare a concluziilor. În plus, disponibilitatea incompletă a datelor pentru anumiți ani și indicatori a impus excluderea unor observații, ceea ce a condus la un eșantion și mai restrâns pentru anumite analize.

### 5.8 Rezultat
În ansamblu, etapa de feature engineering și analizele statistice aplicate au permis formularea unor răspunsuri clare la întrebările de cercetare ale proiectului. Transformările realizate și indicatorii construiți au facilitat evaluarea relației dintre consumul de energie și nivelul de dezvoltare economică, identificarea diferențelor structurale dintre țări în ceea ce privește eficiența energetică și intensitatea emisiilor, precum și analiza rolului energiei regenerabile și al accesului la electricitate.

Prin utilizarea corelațiilor, regresiilor OLS și testelor ANOVA, analiza a evidențiat atât relații robuste, cât și limitele acestora în contextul eșantionului analizat, oferind o perspectivă coerentă asupra dinamicii energie–dezvoltare–mediu în statele din Europa de Est membre ale Uniunii Europene. Astfel, rezultatele obținute susțin concluziile finale ale proiectului și confirmă relevanța metodologică a etapelor de feature engineering și analiză statistică.
