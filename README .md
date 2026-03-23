# Geboortedataanalyse 2019

Analyse van geboortecijfers voor het jaar 2019 op basis van dagelijkse CSV-bestanden per gemeente.

---

## Projectstructuur

```
Oefening01.01/
├── config.json
├── ReadFiles.ipynb
└── data/
    ├── geboortes/        ← bronbestanden (één CSV per dag, YYYY-MM-DD.csv)
    └── temp/             ← tussentijdse bestanden
```

---

## Configuratie

Het project wordt gestuurd via `config.json`. Dit bestand bevat:

| Sleutel | Beschrijving |
|---|---|
| `Directories.Root` | Rootmap van het project |
| `Directories.Data` | Submap met de data |
| `Directories.Input` | Map met de bronbestanden (`data/geboortes`) |
| `Directories.Temp` | Map voor tussentijdse bestanden |
| `Files.RestFile` | Naam van het tussentijds Excel-bestand |
| `Decisions.Wrong Date` | Functie die wordt aangeroepen bij een ongeldige datum in de bestandsnaam (`volgende_geldige_datum` of `laatste_geldige_datum`) |
| `Plot.kleuren` | Woordenboek met kleurcodes voor de grafieken |

### Beschikbare kleuren

```json
{
    "primair":    "#4C72B0",
    "secundair":  "#DD8452",
    "groen":      "#55A868",
    "rood":       "#C44E52",
    "paars":      "#8172B3",
    "bruin":      "#937860",
    "roze":       "#DA8BC3",
    "grijs":      "#8C8C8C",
    "geel":       "#CCB974",
    "lichtblauw": "#64B5CD",
    "marineblauw": "#00008B"
}
```

---

## Brondata

Elke CSV-bestand stelt één kalenderdag voor. De bestandsnaam is de effectieve geboortedatum (`YYYY-MM-DD.csv`).

### Kolommen per CSV

| Kolom | Type | Beschrijving |
|---|---|---|
| `gemeente` | str | Gemeente van geboorte |
| `naam` | str | Voornaam van het kind |
| `geslacht` | str | `Mannelijk` of `Vrouwelijk` |
| `verwachte datum` | datetime | Verwachte bevallingsdatum |

### Afgeleide kolommen (toegevoegd bij inlezen)

| Kolom | Beschrijving |
|---|---|
| `date` | Effectieve geboortedatum (afgeleid uit bestandsnaam) |
| `SourceFile` | Naam van het bronbestand |
| `dag van het jaar` | Dagnummer (1–365) |
| `dag van de week` | Weekdagnummer (0=maandag, 6=zondag) |

---

## Verwerking bij ongeldige datums

Bestandsnamen die geen geldige datum bevatten, worden verwerkt via de functie vermeld in `config["Decisions"]["Wrong Date"]`:

| Functie | Gedrag |
|---|---|
| `volgende_geldige_datum` | Gebruikt de eerste dag van de volgende maand |
| `laatste_geldige_datum` | Gebruikt de laatste geldige dag van die maand |
| *(andere/lege waarde)* | Bestand wordt overgeslagen (`continue`) |

---

## Centrale DataFrames

| DataFrame | Beschrijving |
|---|---|
| `df_births` | Alle rijen uit alle bronbestanden, gesorteerd op datum |
| `df_births_clean` | `df_births` zonder outlier-dagen (dag 1 en dag 182) |
| `df_wrong` | Verwijderde rijen met kolom `reden` |
| `geboortes_clean` | Aantal geboortes per dag + voortschrijdend weekgemiddelde |

---

## Analyses

### Stap 1 — Data inlezen
- `df_births` opgebouwd uit alle CSV-bestanden
- 116.850 rijen, periode 2019-01-01 t.e.m. 2019-12-31
- Dag van het jaar loopt van 1 t.e.m. 365
- Ongeldige datums worden afgehandeld via de config

### Stap 2 — EDA

**Vraag 1 — Initiële plot**
Lijnplot van het aantal geboortes per dag van het jaar met rode stippellijn voor het gemiddelde.

**Vraag 2.1 — Outliers zoeken**
Outliers gedefinieerd als datapunten die meer dan 70% van de weg tussen het gemiddelde en het maximum/minimum liggen.
- Bovenste outliers: dag 1 (534 geboortes) en dag 182 (923 geboortes)
- Onderste outliers: cluster rond dag 290–360

**Vraag 2.2 — Outlier remediation**
Dag 1 en dag 182 verwijderd uit `df_births` → opgeslagen in `df_wrong` met reden. `df_births_clean` bevat de gecorrigeerde dataset.

**Vraag 2.3 — 8 meest extreme dagen H2**
Absolute afwijking t.o.v. het gemiddelde van de tweede helft (dag 183–365). Top 8 geselecteerd via `nlargest`.

**Vraag 3.1 — Voortschrijdend weekgemiddelde**
Rolling gemiddelde met venster 7, `center=True`. Elke dag = gemiddelde van zichzelf + 3 voor + 3 na.

**Vraag 3.2 — Geboortes per weekdag**
Staafdiagram per weekdag. Zaterdag en zondag vertonen duidelijk lagere aantallen.

**Vraag 3.3 — Maandverschillen**
Gemiddeld aantal geboortes per dag per maand met 95% betrouwbaarheidsinterval. BI berekend via Monte Carlo simulatie op de t-verdeling (zonder scipy).

**Vraag 3.4 — Weekdag × seizoen**
Boxplots per seizoen (Winter/Lente/Zomer/Herfst) en weekdag. Toont of het weekdag-effect constant is doorheen het jaar.

### Stap 3 — Onderzoeksvragen

**Onderzoek 1 — Unisex namen**

| Vraag | Beschrijving |
|---|---|
| 1.1 | `df_name_gender`: één rij per naam met aantal jongens, meisjes en totaal |
| 1.2 | `df_real_unisex`: namen waarbij `x ≤ 1.5·y` én `y ≤ 1.5·x` |
| 1.3 | Percentage jongens/meisjes met een echte unisex naam |
| 1.4 | Gestapeld horizontaal staafdiagram: aandeel jongens vs meisjes per unisex naam |

**Onderzoek 2 — Accuraatheid verwachte bevallingsdatum**

| Vraag | Beschrijving |
|---|---|
| 2.1 | Effectieve vs. verwachte geboortes per dag van het jaar |
| 2.2 | Randeffect: afwijking aan begin/einde jaar door ontbrekende data buiten 2019 |
| 2.3 | Histogram van aantal dagen te vroeg geboren (mediaan + 90e percentiel) |
| 2.4 | Scatterplot effectief vs. verwacht per gemeente (top 8) |

**Onderzoek 3 — Namen vs. aantal geboortes**

| Vraag | Beschrijving |
|---|---|
| 3.1 | Cumulatief aantal unieke namen vs. cumulatief aantal geboortes — sublineair verband |

---

## Hulpfuncties

| Functie | Beschrijving |
|---|---|
| `is_geldige_datum(datum)` | Controleert of een string een geldige `YYYY-MM-DD` datum is |
| `volgende_geldige_datum(datum)` | Geeft de eerste dag van de volgende maand terug bij ongeldige datum |
| `laatste_geldige_datum(datum)` | Geeft de laatste geldige dag van die maand terug |
| `load_project_config(path)` | Laadt en valideert `config.json`, stopt met foutmelding bij problemen |
| `ci95(x)` | Berekent gemiddelde + 95% BI via Monte Carlo simulatie op de t-verdeling |

---

## Vereisten

```
pandas
numpy
matplotlib
```

> `scipy` wordt **niet** gebruikt. Het 95% betrouwbaarheidsinterval wordt berekend via `numpy.random.standard_t` (Monte Carlo).
