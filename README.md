# SkolViz
Rīgas 2026 skolu uzņemšanas plūsmas demo
# Teorētisks Rīgas 10. klašu uzņemšanas vietu sadales modelis
https://cresado-labx.github.io/SkolViz/skoluviz.html

## TLDR

Šis ir **interaktīvs vizualizācijas rīks**, kas simulē Rīgas 10. klašu uzņemšanas procesu. Modelis parāda, kā skolēni tiek sarindoti pēc kopvērtējuma, izsūta uzaicinājumi un kā skolēni var pieņemt/noraidīt uzaicinājumus 72 stundās, izraisot kaskādes ar nākamajiem uzaicinājumiem.

**Galvenās plūsmas:**
- 📊 **Punktu plūsma** – Aprēķina kopvērtējumus
- 📋 **Ranžēšanas plūsma** – Sakārto pieteikumus un nosaka robežas
- 📬 **Uzaicinājuma plūsma** – Izsūta uzaicinājumus
- ⏰ **72h lēmumu plūsma** – Procesa skolēnu lēmumus un izraisa kaskādes
- 🎨 **Vizualizācija** – Kanvā parāda skolēnus un pieteikumus

---

## Ātrs start

1. **Atvērt HTML failu** – Atveriet `skoluvize_patched_group_a.html` jebkurā mūsdienu pārlūkprogrammā
2. **Nokonfigurēt parametrus** (neobligāti):
   - Skolēnu skaits (noklusējums: 180)
   - Pieteikumi uz skolēnu (noklusējums: 3–6)
3. **Nospiest "Sākt"** – Ģenerē jaunu fiktīvu simulāciju
4. **Nospiest "Sarindot"** – Aprēķina kopvērtējumus un ranžē
5. **Nospiest "Izsūtīt uzaicinājumus"** – Izsūta pirmos uzaicinājumus
6. **Nospiest "Nākamās 72h"** – Simulē vienu 72 stundu ciklu
7. **Nospiest "Automātiski līdz stabilam"** – Darbina līdz end-state (var paņemt minūti)

### Ko redzēt

- **Kreisajā pusē** – Vadības poga un parametri
- **Vidū** – Animēta kanvā ar skolēniem (baseina vidū) un programmām (kartītes apkārt)
- **Labajā pusē** – Informācijas paneli: paskaidrojums, atlasītā skolēna detaļas, statistika, žurnāls

---

## Struktura

Kods ir sadalīts **8 galvenajos moduļos**:

### 1. Datu shēmas un ģenerators
**Fails:** `generateSimulation()` funkcija  
**Nolūks:** Izveido fiktīvu simulāciju ar:
- 5 skolām, katrā po 2 programmām (matemātikas + vispārējā)
- Konfigurējamu skolēnu skaitu (noklusējums 180)
- Nejaušus pieteikumus uz skolēniem (min 3, max 6 uz skolēnu)

**Izvades struktura:**
```javascript
{
  meta: { solis, cikls72h, seed, paskaidrojums },
  skolas: [],          // Skolu saraksts
  programmas: [],      // Programmu saraksts ar vietām
  skoleni: [],         // Skolēnu saraksts ar punktiem
  pieteikumi: [],      // Pieteikumu saraksts
  notikumi: []         // Žurnāls ar notikumiem
}
```

**Palīgfunkcijas:**
- `createSeededRandom(seed)` – Pseudo-nejaušu skaitļu ģenerators ar seeded reperatīvumu
- `weightedSampleWithoutReplacement()` – Atlasa skolas/programmas pēc svara
- `priorityNoise()` – Pievieno nejaušību skolēnu prioritātēm

---

### 2. Punktu plūsma
**Funkcija:** `scoreApplications(simulacija)`  
**Nolūks:** Aprēķina kopvērtējumu katram pieteikumam pēc formulas:

```
kopvērtējums = CE × 0.35 + iestājpārbaudījums × 0.45 + vidējāAtzīme × 10 × 0.20 - prioritātePenālis
```

**Procesa soļi:**
1. Ielāde skolēniem piesaistītie punkti
2. Aprēķina vērtējumu katrā programmā
3. Izfiltrē derīgos pieteikumus (vidējā atzīme ≥ 6.0)
4. Atjaunina `meta.solis` uz `"punkti"`

**Izejas efekts:** Katrs pieteikums iegūst `kopvertejums` un `derigs` karoga vērtības

---

### 3. Ranžēšanas plūsma
**Funkcija:** `rankApplicationsAndSetCutoffs(simulacija)`  
**Nolūks:** Ranžē pieteikumus katrā programmā un nosaka uzaicinājuma robežas

**Procesa soļi:**
1. Katrā programmā sortē derīgos, aktīvos pieteikumus pēc kopvērtējuma (dilstošs)
2. Piešķir `rangs` (1, 2, 3, ...)
3. Pirmie `programma.vietas` pieteikumi tiek marķēti kā `virsRobezas = true`
4. Pārējie kā `virsRobezas = false`
5. Nosaka `uzaicinatoRobeza` – pēdējā augstākā vērtējuma punkti, kuri vēl ir virs robežas

**Izejas stāvokļi:**
- `virs_robezas_bet_neuzaicinats` – Virs robežas, bet vēl nav uzaicinātie
- `gaida` – Zem robežas

---

### 4. Uzaicinājuma plūsma
**Funkcija:** `issueInvitations(simulacija)`  
**Nolūks:** Katram skolēnam izsūta tikai VIENU uzaicinājumu uz augstākās prioritātes programmu

**Procesa soļi:**
1. Grupē pieteikumus pēc skolēna ID
2. Katram skolēnam atrod augstākās prioritātes pieteikumu, kas ir `virsRobezas`
3. To marķē kā `uzaicinats`
4. Pārējos "virs robežas" pieteikumus marķē kā `virs_robezas_bet_neuzaicinats`

**Rezultāts:** Tikai viens aktīvs uzaicinājums uz skolēnu

---

### 5. 72h lēmumu plūsma
**Funkcija:** `process72hDecisions(simulacija, options)`  
**Nolūks:** Simulē skolēnu lēmumus par uzaicinājumiem

**Lēmumu varbūtības:**
- 58% – **Apstiprina** (`apstiprinats`) ✅
  - Atceļ visus zemākas prioritātes pieteikumus
- 28% – **Noraida** (`noraidits`) ❌
  - Pieteikums paliek disponējams nākamajam ciklam
- 14% – **Atsauc dalību** (`atsaukts`) ↩️
  - Atsauc VISUS skolēna pieteikumus

**Kaskāde:** Atbrīvotās vietas izraisa nākamo uzaicinājumu vilni

**Žurnāls:**
```
72h cikls #1: apstiprināti 58, noraidīti 29, atsaukti 10.
```

---

### 6. Kaskādes dzinējs (Orķestrācija)
**Funkcija:** `createCascadeEngine({...})`  
**Nolūks:** Orķestrē pilnu simulācijas plūsmu

**Galvenās metodes:**
- `runInitialRound()` – Izpilda punktu → ranžēšanu → uzaicinājumus (sākuma kārta)
- `runNext72hCycle()` – Procesa lēmumus, ranžē atkārtoti, izsūta jaunos uzaicinājumus
- `isStable()` – Pārbauda, vai simulācija ir sasniedzusi stabilu stāvokli (nav aktīvu uzaicinājumu)

**Stabilitātes kritēriji:**
- Nav neviena aktīva uzaicinājuma (`statuss === "uzaicinats"`)
- Uzkrāti 12 cikli (drošības limits)

---

### 7. Vizualizācijas slānis
**Funkcija:** `createVisualization(canvas, options)`  
**Nolūks:** Kanvā zīmē animētu simulācijas reprezentāciju

**Elementi uz kanvā:**

#### Skolēnu baseins (vidus)
- Apļveida area ar **skolēna punktiņiem**, kas rotē
- Punkta krāsa atkarīga no skolēna stāvokļa
- Klikšķis uz punkta – atlasa/neatlas skolēnu

#### Programmu kartes (četras malas)
- Taisnstūra kartītes ar programmas nosaukumu un vietu skaitu
- Ikreiz kartītes punkti (pieteikumi):
  - **Sakumā** – Nejauši izkaisīti
  - **Pēc ranžēšanas** – Sakārtoti rindās, ar dzeltenām robežu līnijām
- **Peles hover** – Rāda skolēna vārdu tooltip

#### Zaļas līnijas
- Izvelkas no atlasīta skolēna uz viņa pieteikumiem redzamajās programmās
- Palīdz vizualizēt saistības

**Krāsu shēma:**
- `#8b95a7` – Gaidītāji
- `#f59e0b` – Virs robežas, bet neuzaicinātie
- `#06b6d4` – Uzaicinātie
- `#4ade80` – Apstiprināts
- `#f97316` – Atsaukts pēc uzaicināšanas
- `#d946ef` – Atsaukts pirms uzaicināšanas
- `#475569` – Atcelts zemākas prioritātes dēļ

**Palīgfunkcijas:**
- `colorForStatus()` – Krāsa pēc stāvokļa
- `findNearestHit()` – Detektē klikšķi uz punkta
- `drawApplicationCircle()` – Zīmē pieteikuma apli
- `drawBoundaryBetweenApps()` – Zīmē dzeltenās robežas

---

### 8. UI un vadības panelis
**Objekts:** `AppController`  
**Nolūks:** Vada visu simulāciju un atjaunina ekrānu

#### Vadības pogas
- **Sākt** – Ģenerē jaunu simulāciju
- **Sarindot** – Izpilda ranžēšanu un robežu noteikšanu
- **Izsūtīt uzaicinājumus** – Izsūta uzaicinājumus
- **Nākamās 72h** – Procesa vienu 72h ciklu
- **Automātiski līdz stabilam** – Darbina 72h ciklus, līdz simulācija stabilizējas
- **Atiestatīt** – Dzēš visu un starts no sākuma

#### Parametri
- **Animācijas ātrums** – Multifikators (0.25× — 3×)
- **Skolēnu skaits** – 80–400
- **Pieteikumi uz skolēnu (min/max)** – 1–10

#### Redzamās programmas
- Checkbox saraksts – Ieslēgt/izslēgt programmas vizualizācijā

#### Labā puse (info paneli)
- **Kas pašlaik notiek?** – Paskaidrojums par darbības fāzi
- **Atlasītais skolēns** – Detalizēts pieteikumu saraksts (ja atlasīts)
- **Kopsavilkums** – Statistika (skolēni, pieteikumi, vietas, uzaicinātie, apstiprinātie, cikli)
- **Notikumu žurnāls** – Kronologisks notikumu saraksts (apgrieztā secībā)

#### Metodes
- `start()` – Ģenerē jaunu simulāciju
- `rank()` – Sarindo pieteikumus
- `invite()` – Izsūta uzaicinājumus
- `next72h()` – Nākamais 72h cikls
- `autoUntilStable()` – Automātiska simulācija
- `reset()` – Atiestatīšana
- `updateButtonStates()` – Ieslēdz/izslēdz pogas pēc stāvokļa
- `refresh()` – Atjaunina visu ekrānu
- `updateSelectedStudentPanel()` – Atjaunina atlasītā skolēna infopaneli
- `updateStats()` – Atjaunina statistiku

---

## Palīgfunkcijas

| Funkcija | Nolūks |
|----------|--------|
| `isActiveApplication(p)` | Pārbauda, vai pieteikums ir aktīvs (ne apstiprināts, ne atsaukts) |
| `cancelLowerPriorityApplications()` | Atceļ zemākas prioritātes pieteikumus, kad augstāka tiek apstiprinātā |
| `withdrawAllStudentApplications()` | Atsauc visus skolēna pieteikumus |
| `groupBy()` | Grupē masīvu pēc atslēgas funkcijas |
| `pseudoRandom()` | Deterministisks nejaušuma ģenerators teksta bāzē |
| `layoutProgramRects()` | Aprēķina programmu kartīču pozīcijas (4 malas) |
| `roundRect()` | Zīmē noapaļotus taisnstūrus |
| `wrapText()` | Laužas tekstu vairākās rindās kanvā |

---

## Datu plūsma

```
1. generateSimulation()
   ↓
2. scoreApplications()  [Aprēķina kopvērtējumus]
   ↓
3. rankApplicationsAndSetCutoffs()  [Ranžē un nosaka robežas]
   ↓
4. issueInvitations()  [Izsūta uzaicinājumus]
   ↓
5. Skolēnu lēmumi (72h)  [process72hDecisions()]
   ↓
6. ATKĀRTOT 3–5 (līdz stabilam)
```

---

## Stāvokļu diagramma

```
pieteikums.statuss:

gaida
  ↓
virs_robezas_bet_neuzaicinats  (ranžēšanā)
  ↓
uzaicinats  (izsūtīti uzaicinājumi)
  ├→ apstiprinats (72h lēmums)
  ├→ noraidits (72h lēmums)
  └→ atsaukts (72h lēmums — visi)

atcelts_zemakas_prioritates_del  (kaskāde, ja augstāka apstiprinātā)
```

---

## Svarīgumi konstantes

| Konstante | Vērtība | Nolūks |
|-----------|---------|--------|
| CE svars | 0.35 | Centralizētā iestāžu pārbaude |
| Iestājpārbaudījums svars | 0.45 | Augstākais svars |
| Vidējā atzīme svars | 0.20 | Vidus svars |
| Min vidējā atzīme | 6.0 | Derības slieksnis |
| Prioritātes penālis | 0.01 per prioritāte | Palielina augstāks prioritātes |
| Apstiprinājuma varbūtība | 58% | 72h lēmums |
| Noraidīšanas varbūtība | 28% | 72h lēmums |
| Atsaukšanas varbūtība | 14% | 72h lēmums |
| Max cikli | 12 | Drošības limits |

---

## Iespējamā paplašināšana

- ⚙️ Pielāgojams punktu aprēķins (svari, formulas)
- 📈 Detalizēti statistikas grafiki
- 💾 Simulāciju saglabāšana/nolāde
- 🌐 Tiešsaistes sadarbība
- 📱 Mobilais saderības režīms
- 🎬 Simulācijas ierakstīšana un atskaņošana

---

## Failu struktura

```
skolviz/
├── skoluvize_patched_group_a.html    # Galvenais fails – atvērt pārlūkprogrammā
├── patch.diff                         # Patch fails ar izmaiņām (informācijas nolūkam)
├── README.md                          # Šis dokuments
└── spec.md                            # Specifikācija (ja pieejama)
```

**HTML fails satur visu:**
- Stilus (CSS)
- Kodu (JavaScript) ar 8 moduļiem
- HTML marķējumu (vienkāršs 3-kolonnu izkārtojums)

---

## Pārlūkprogrammas prasības

✅ **Atbalstītas:**
- Chrome/Chromium 90+
- Firefox 88+
- Safari 14+
- Edge 90+

❌ **Neatbalstītas:**
- Internet Explorer (nav `structuredClone`, moderns CSS)
- Ļoti veci mobilās pārlūkprogrammas

---

## Vizualizācijas UI apskats

### Kreisais panelis (vadība)
```
┌─────────────────────────┐
│ Vadība                  │
├─────────────────────────┤
│ [Sākt]                  │
│ [Sarindot]              │
│ [Izsūtīt uzaicinājumus] │
│ [Nākamās 72h]           │
│ [Automātiski ...]       │
│ [Atiestatīt]            │
├─────────────────────────┤
│ Animācijas ātrums: [1×] │
│ Skolēnu skaits: [180]   │
│ Pieteikumi min: [3]     │
│ Pieteikumi max: [6]     │
├─────────────────────────┤
│ Skolas / programmas     │
│ ☑ Programma 1           │
│ ☑ Programma 2           │
│ ...                     │
├─────────────────────────┤
│ Leģenda                 │
│ ● gaidītāji             │
│ ● virs robežas          │
│ ● uzaicinātie           │
│ ...                     │
└─────────────────────────┘
```

### Vidējais panelis (kanvā)
```
                ┌──────────────────────────┐
    Programma A │                          │ Programma B
    ┌───────────┤                          ├───────────┐
    │           │    [Skolēnu baseins]     │           │
    │   ○ ○ ○   │      ⊙ ⊙ ⊙ ⊙ ⊙         │   ○ ○ ○   │
    │   ○ • ○   │      ⊙ → ⊙ ← ⊙         │   ○ ○ ○   │
    │   ○ ○ ○   │      ⊙ ⊙ ⊙ ⊙ ⊙         │   ○ ○ ○   │
    │           │                          │           │
    └───────────┤                          ├───────────┘
    Programma C │                          │ Programma D
                └──────────────────────────┘

Simboli:
⊙ = aktīvs skolēns baseins (pulsē)
→ = atlasīts skolēns (ar halo)
○ = pieteikums viena programmā
─ = saite starp skolēnu un pieteikumu
```

### Labais panelis (info)
```
┌──────────────────────────┐
│ Kas notiek?              │
├──────────────────────────┤
│ [Teksta paskaidrojums]   │
├──────────────────────────┤
│ Atlasītais skolēns       │
├──────────────────────────┤
│ [Pieteikumu saraksts]    │
├──────────────────────────┤
│ Kopsavilkums             │
│ 180 skolēni              │
│ 920 pieteikumi           │
│ 245 vietas kopā          │
│ 0 uzaicinātie            │
├──────────────────────────┤
│ Notikumu žurnāls         │
│ 3. Aprēķināti vērtējumi  │
│ 2. Pieteikumi sarindoti  │
│ 1. Ģenerēti dati         │
└──────────────────────────┘
```

---

## Interaktivitāte

| Darbība | Rezultāts |
|---------|-----------|
| Klikšķis uz skolēna baseinā | Atlasa/neatlas skolēnu (max 10) |
| Peles hover uz punkta | Parāda skolēna vārdu tooltip |
| Programmas checkbox | Rāda/slēpj programmu kartīti |
| Animācijas ātruma slider | Paātrina/palēnina kanvā |
| Parametru maiņa | Jāatkārto "Sākt" jaunu simulāciju |

---

## Svarīgumi jēdzieni

| Jēdziens | Skaidrojums |
|----------|-----------|
| **Kopvērtējums** | CE × 0.35 + iestājpārbaudījums × 0.45 + vidējā atzīme × 10 × 0.20 − prioritātes penālis |
| **Rangs** | Pieteikuma pozīcija programmā (1 = labākais, 2 = otrais labākais, ...) |
| **Virs/zem robežas** | `virsRobezas = true` nozīmē, ka vieta ir atvērta (rangs ≤ brīvas vietas) |
| **Uzaicinājuma robeža** | Kopvērtējums pēdējā pieteikuma, kas ir virs robežas |
| **Aktīvs pieteikums** | Nav `apstiprinats`, `noraidits`, `atsaukts` vai `atcelts` |
| **Kaskāde** | Kad skolēns apstiprina, zemākas prioritātes pieteikumi tiek nogriezti, atbrīvojot vietas citiem |
| **Stabilitāte** | Simulācija ir stabila, kad nav aktīvu uzaicinājumu vai sasniegtas 12 cikli |

---

## Jautājumi un troubleshooting

**P: Kanvā neko neredzam**  
A: Pārlādējiet lapu. Pārbaudiet pārlūkprogrammas konsoli (`F12`) par kļūdām.

**P: Simulācija izsalst vai ir ļoti lēna**  
A: Samaziniet skolēnu skaitu (noklusējums 180). Palieliniet animācijas ātrumu slider.

**P: Kāpēc dati vienmēr vienādi?**  
A: Kods izmanto fixed seed (12345) atkārtojamībai. Mainiet `seed` parametru, lai saņemtu citus datus.

**P: Kā mainīt aprēķina formulu?**  
A: Redakcija `scoreApplications()` funkciju kodam HTML failā (aptuveni 453. rinda).

---

## Liecības

⚠️ **Svarīgi:** Šis ir ilustratīvs modelis ar fiktīviem datiem. Tas **nav oficiāls** Rīgas skolu uzņemšanas aprēķins un nav paredzēts reālai izmantošanai pieņemšanas procesā.



