# Tankelinjer för examensprojektet

Projektet ämnar att öka min nuvarande kunskap gällande statistisk modellering, samt att få möjlighet att produktionssätta en prediktionsmodell i en Shiny-app. 
Inspiration hämtades från en tidigare multipel regressionsansats som skattade effekten av olika egenskaper på försäljningspriset för villor i Malmö – ett arbete som väckte intresset för att utforska området ytterligare.

Shmueli (2010) belyser en central distinktion inom statistisk modellering – skillnaden mellan att förklara och att predicera. 
Ett förklarande syfte syftar till att testa kausala hypoteser och skatta effektstorleken av olika variabler på ett utfall, medan ett prediktivt syfte istället fokuserar på att minimera prediktionsfelet för nya observationer. 
Författaren poängterar att dessa två mål ofta sammanblandas, trots att de ställer fundamentalt olika krav på modellbygget – exempelvis gällande variabelval, modellkomplexitet och hur bias-varians-avvägningen hanteras. 
Det föregående projektet hade ett förklarande syfte, där regressionsmodellen användes för att skatta hur olika egenskaper påverkade slutpriset för villor i Malmö. 
I detta projekt är syftet däremot renodlat prediktivt – målet är inte att tolka enskilda variablers effekt, utan att erhålla en så träffsäker uppskattning som möjligt av försäljningspriset utifrån aktiva annonser på Hemnet.

1. Data
   



## Dataanalys och features

**Dina befintliga variabler ser bra ut**, men det finns några saker att tänka på:

- `listing_price` och `price_development` är **dataleakage-risk** – listing price är känd vid annonsering, men price_development (slutpris minus listpris) kan du inte ha som prediktor eftersom den implicerar slutpriset. Ta bort den.
- `secondary_area_missing` är en bra imputation-flagga, fortsätt med den.
- `sale_date`, `month`, `year` – överväg att behålla dessa för säsongsmönster.

## Externa variabler att lägga till

**Makroekonomiska (störst påverkan):**
- **Bolåneränta** (Riksbankens styrränta eller genomsnittlig bolåneränta vid säljtillfället) – enskilt starkaste makrovariabel för svenska huspriser
- **KPI/inflation** vid säljtillfället
- **HOX-index** eller liknande prisindex för Malmö/Skåne som laggad variabel

**Geografiska (troligen stor förbättring):**
- Avstånd till Malmö C (eller närmaste tågstation) – beräkna från lat/lon
- Avstånd till hav/strand
- Genomsnittligt kvadratmeterpris per neighborhood (aggregerat från din träningsdata)
- Skolbetyg/skola i närheten via Skolverkets öppna data

**Grannskapsstatistik (SCB öppen data):**
- Medianinkomst per postnummer/DeSO-område
- Befolkningstäthet
- Andel bostadsrätter vs villor i området

## Modellval i R

Med ~2200 obs och den här typen av skev fördelning (huspriser) rekommenderar jag:

```r
# Starta med dessa, jämför:
library(tidymodels)

# 1. XGBoost – hanterar icke-lineariteter, saknade värden, interaktioner bra
# 2. Random Forest (ranger) – robust baseline
# 3. Linjär regression med log(house_price) – tolkbar, bra benchmark

# Feature engineering
recipe <- recipe(house_price ~ ., data = train) %>%
  step_log(house_price, skip = TRUE) %>%   # log-transformera target
  step_mutate(
    age = year - year_built,
    sqm_price_listing = listing_price / living_area,
    dist_to_center = geosphere::distHaversine(...)
  ) %>%
  step_date(sale_date, features = c("month", "quarter")) %>%
  step_impute_median(all_numeric_predictors())
```

**Log-transformera house_price** – huspriser är höger-skeva och modeller presterar tydligt bättre på log-skala. Kom ihåg att exponera prediktionerna.

## Validering – viktigt för tidsseriedata

Använd **tidsbaserad cross-validation**, inte slumpmässig split:

```r
library(rsample)
# Träna på t.ex. 2023-2024, validera på jan-mar 2025
splits <- time_series_split(data, date_var = sale_date, 
                             assess = "3 months")
```
Annars överskattar du modellens prestanda pga att grannskapseffekter "läcker" mellan train/test.

## Shiny-appen i produktion

Tänk på detta flödet:

```
Hemnet-annons (input) → feature engineering → model predict → UI
```

- Spara modellen med `readr::write_rds()` eller `bundle`-paketet
- Räkna ut geografiska features (avstånd etc.) **i realtid** i appen
- Hämta aktuell bolåneränta via API (t.ex. Riksbankens API) eller uppdatera manuellt
- Visa ett **konfidensintervall**, inte bara punktestimat – `quantreg`-forest eller XGBoost med quantile-loss gör detta bra

## Vanliga fallgropar

- **Neighborhood som faktor** – har du tillräckligt med obs per neighborhood? Slå ihop sällsynta neighborhoods eller använd target encoding
- **Multikollinearitet** – `living_area`, `number_of_rooms` och `plot_area` är korrelerade
- **Modellen åldras** – huspriser förändras snabbt med ränteläget, bygg in en pipeline för att omträna med nya data
