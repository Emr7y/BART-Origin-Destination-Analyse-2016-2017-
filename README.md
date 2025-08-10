# 🚇 BART Origin–Destination Analyse (2016–2017)

Dieses Repo enthält eine kompakte, reproduzierbare Analyse der BART‑Fahrten 2016–2017. Wir verknüpfen die Trip‑Tabelle (Origin/Destination, Throughput, DateTime) mit der Stationstabelle (Abkürzung, Name, Koordinaten), normalisieren Codes und beheben historische Alias‑Abweichungen (z. B. WSPR → WARM).

Auf Basis der vom Datensatz gelieferten Zählgröße Throughput (Anzahl pro Zeitintervall, wie bereitgestellt) aggregieren wir:

Top‑Routen (gesamt oder durchschnittlich pro Tag),

Tageszeit‑Muster (Ø Trips pro Stunde),

Wochentags‑Muster (Ø Trips pro Wochentag).

Der Fokus liegt auf klaren, wenigen Visualisierungen und exportierbaren Tabellen – ideal für eine kurze Ergebnispräsentation ohne langatmige EDA im README (die ausführliche Exploration findet im Notebook statt).

## 🎯 Ziele

* **Top‑Routen** identifizieren (gesamt & Ø/Tag)
* **Tageszeit‑Profil** (Stunden) & **Wochentags‑Profil** (Ø/Tag)
* Stationen verknüpfen (Name + Koordinaten), inkl. **Alias‑Fix** (z. B. `WSPR` → `WARM`).

---

## 📦 Daten

* **Trips:** `df_1617` mit Spalten: `Origin`, `Destination`, `Throughput`, `DateTime`
* **Stationen:** `df_st` mit Spalten: `Abbreviation`, `Name`, `lat`, `lon`

> Falls Spaltennamen abweichen, bitte unten im „Setup“ die `rename`‑Zeile anpassen.

---

## ⚙️ Setup (kurz)

```bash
pip install pandas matplotlib folium  # folium optional
```

---

## 🧭 Minimaler Workflow (Notebook)

### 1) Stationen vorbereiten & Mergen

```python
# Stationstabelle säubern
df_st_fix = (
    df_st.rename(columns={"Abbreviation":"station_code","Name":"station_name"})
         [["station_code","station_name","lat","lon"]]
         .copy()
)
df_st_fix["station_code"] = df_st_fix["station_code"].str.strip().str.upper()

# Trips normalisieren
time_col = "DateTime"
df_1617[time_col] = pd.to_datetime(df_1617[time_col], errors="coerce")
df_1617["date_only"] = df_1617[time_col].dt.date
for c in ["Origin","Destination"]:
    df_1617[c] = df_1617[c].str.strip().str.upper()

# Alias (historische Codes)
alias = {"WSPR":"WARM"}  # Warm Springs/South Fremont
df_1617[["Origin","Destination"]] = df_1617[["Origin","Destination"]].replace(alias)

# Merge: Origin & Destination
DF = (
    df_1617.merge(df_st_fix, how="left", left_on="Origin", right_on="station_code")
            .rename(columns={"lat":"origin_lat","lon":"origin_lon","station_name":"origin_name"})
            .drop(columns=["station_code"])  
)
DF = (
    DF.merge(df_st_fix, how="left", left_on="Destination", right_on="station_code")
      .rename(columns={"lat":"dest_lat","lon":"dest_lon","station_name":"dest_name"})
      .drop(columns=["station_code"])  
)
```

### 2) Top‑15 Routen (gesamt)

```python
keys = [c for c in ("Origin","Destination","origin_name","dest_name") if c in DF.columns]
top15 = (DF.groupby(keys, dropna=False)["Throughput"].sum()
          .reset_index(name="trips").nlargest(15, "trips"))
```

### 3) Stundenprofil & Wochentagsprofil (Ø/Tag)

```python
DF["hour"] = DF[time_col].dt.hour
DF["dow"]  = DF[time_col].dt.dayofweek

hourly_avg = (DF.groupby(["date_only","hour"])["Throughput"].sum()
                .groupby("hour").mean())

dow_avg = (DF.groupby(["date_only","dow"])["Throughput"].sum()
             .groupby("dow").mean())
```

### 4) Simple Plots (kompakt)

```python
import matplotlib.pyplot as plt

# Top‑15 Balken (horizontal)
top15["route"] = top15["origin_name"] + " → " + top15["dest_name"]
plt.barh(top15["route"], top15["trips"]); plt.gca().invert_yaxis()
plt.title("Top 15 Routen – Trips gesamt"); plt.xlabel("Trips"); plt.tight_layout(); plt.show()

# Ø Trips pro Stunde
hourly_avg.plot(marker='o', title='Ø Trips pro Stunde'); plt.xlabel('Stunde'); plt.ylabel('Ø Trips'); plt.show()

# Ø Trips pro Wochentag
ax = dow_avg.plot(kind='bar', title='Ø Trips pro Wochentag'); ax.set_xlabel('Wochentag'); ax.set_ylabel('Ø Trips')
ax.set_xticklabels(['Mo','Di','Mi','Do','Fr','Sa','So']); plt.show()
```

---

## 📤 Exporte (für Abgabe/GitHub)

```python
DF.to_parquet('bart_trips_enriched.parquet', index=False)
top15.to_csv('bart_top15_total.csv', index=False)
hourly_avg.reset_index().to_csv('bart_hourly_avg.csv', index=False)
dow_avg.reset_index().to_csv('bart_weekday_avg.csv', index=False)
```

---

## 🖥️ (Optional) Streamlit‑App – Funktionsidee

* Sidebar: **Metrik** wählen (*Trips gesamt* vs. *Ø pro Tag*), **Top‑N** Slider
* Charts: **Top‑Routen** (Barh), **Ø/Std** (Line), **Ø/Wochentag** (Bar)
* Hinweise: Alias‑Mapping anzeigen (wenn angewendet)
* Export‑Buttons: Aggregationen als CSV herunterladen
* Optional: CSV‑Upload eigener Trips

> Wenn gewünscht, kann eine kompakte `app.py` (\~60–80 Zeilen) generiert werden.

---

## 📁 Repo‑Struktur (Vorschlag)

```
├── notebooks/
│   └── bart_analysis.ipynb
├── data/
│   ├── trips.csv                 # optional
│   └── stations.csv              # optional
├── outputs/
│   ├── bart_top15_total.csv
│   ├── bart_hourly_avg.csv
│   └── bart_weekday_avg.csv
├── app.py                        # optional (Streamlit)
└── README.md
```

---

## 📝 Lizenz

MIT (oder nach Bedarf anpassen)

---

**Made with ❤️ by Emr7y**

## 🔗 Quellen & Details

* **Modell(e):** Keine ML‑Modelle – deskriptive Analyse (Pandas‑GroupBy/Merge, Matplotlib)
* **Datenquelle:** [BART Ridership (Kaggle)](https://www.kaggle.com/datasets/saulfuh/bart-ridership)
