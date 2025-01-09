
# ETL Proces pre databázu MovieLens

## 1. Úvod a popis zdrojových dát

### 1.1 Téma projektu
Tento projekt sa zameriava na analýzu údajov z databázy MovieLens. Dáta zahŕňajú informácie o používateľoch, filmoch, žánroch a používateľských hodnoteniach. Hlavným cieľom projektu je vytvoriť ETL proces v Snowflake na prípravu dát pre analýzu a budovanie modelu, ktorý umožňuje:
- Určiť populárne filmy a žánre.
- Analyzovať preferencie používateľov.
- Skúmať správanie používateľov na základe hodnotení.

---

### 1.2 Typ dát
Databáza obsahuje štruktúrované dáta vrátane číselných, textových a časových polí. Formát dát je MySQL dump a CSV súbory, ktoré sú spracované a nahrané do Snowflake.

---

### 1.3 Popis jednotlivých tabuliek
#### age_group
- Popis: Kategórie vekových skupín používateľov.
- Stĺpce:
  - id: Unikátny identifikátor vekovej skupiny.
  - name: Názov vekovej skupiny.

#### occupations
- Popis: Zoznam povolaní používateľov.
- Stĺpce:
  - id: Unikátny identifikátor povolania.
  - name: Názov povolania.

#### users
- Popis: Informácie o používateľoch.
- Stĺpce:
  - id: Unikátny identifikátor používateľa.
  - age: Vek používateľa.
  - gender: Pohlavie používateľa.
  - zip_code: PSČ používateľa.
  - occupation_id: Identifikátor povolania.
  - age_group_id: Identifikátor vekovej skupiny.

#### ratings
- Popis: Hodnotenia filmov používateľmi.
- Stĺpce:
  - id: Unikátny identifikátor hodnotenia.
  - user_id: Identifikátor používateľa.
  - movie_id: Identifikátor filmu.
  - rating: Hodnotenie (1–5).
  - rated_at: Dátum a čas hodnotenia.

#### movies
- Popis: Informácie o filmoch.
- Stĺpce:
  - id: Unikátny identifikátor filmu.
  - title: Názov filmu.
  - release_year: Rok vydania filmu.

#### genres
- Popis: Žánre filmov.
- Stĺpce:
  - id: Unikátny identifikátor žánru.
  - name: Názov žánru.

#### genres_movies
- Popis: Prepojenie medzi filmami a ich žánrami.
- Stĺpce:
  - id: Unikátny identifikátor záznamu.
  - movie_id: Identifikátor filmu.
  - genre_id: Identifikátor žánru.

---

### 1.4 ERD diagram
Nasledujúci diagram znázorňuje vzťahy medzi tabuľkami:
![ERD Diagram](https://github.com/user-attachments/assets/ac1a3229-be95-470e-b670-55aceeaaeffb)

---

## 2. Návrh dimenzionálneho modelu

### 2.1 Popis dimenzionálneho modelu
Pre projekt bol navrhnutý multi-dimenzionálny model typu hviezda. Tento model umožňuje efektívnu analýzu hodnotení filmov používateľmi. Model obsahuje faktovú tabuľku fact_ratings a niekoľko dimenzionálnych tabuliek.

---

### 2.2 Hlavné tabuľky
#### fact_ratings (Faktová tabuľka)
- Obsahuje kľúčové metriky a prepojenie na dimenzie.
- Stĺpce:
  - rating_id: Unikátny identifikátor hodnotenia.
  - rating: Hodnotenie (1–5).
  - rated_at: Dátum a čas hodnotenia.
  - user_id: Identifikátor používateľa.
  - movie_id: Identifikátor filmu.
  - dim_time_time_id: Identifikátor času.

#### Dimenzionálne tabuľky:
- `dim_users`: Obsahuje informácie o používateľoch.
- `dim_movies`: Obsahuje údaje o filmoch.
- `dim_genres`: Obsahuje zoznam žánrov filmov.
- `dim_time`: Obsahuje informácie o čase (dátum, mesiac, rok).

---

### 2.3 ERD diagram dimenzionálneho modelu
![Dimenzionálny model ERD](https://github.com/user-attachments/assets/cfcf6cb7-98c7-4b2e-87dc-cb67daa4f8d6)

---
### 3. ETL proces v Snowflake

ETL proces (Extract, Transform, Load) je kľúčovou súčasťou dátovej integrácie, pričom sa zameriava na extrakciu dát zo zdrojových systémov, ich transformáciu podľa potreby a následné načítanie do cieľovej databázy. V tomto prípade bol proces implementovaný v platforme Snowflake s cieľom pripraviť zdrojové dáta z vrstvy staging na použitie v dimenzionálnom modeli. Tento model umožňuje analýzu a vizualizáciu dát.

#### 3.1 Extrahovanie dát (Extract)

Proces začal nahrávaním zdrojových dát vo formáte `.csv` do interného stage nazvaného `my_stage`. Na vytvorenie stage sa použil nasledujúci SQL príkaz:

```sql
CREATE OR REPLACE STAGE my_stage 
FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"');
```

Po vytvorení stage sa obsah každého `.csv` súboru načítal do staging tabuliek. Napríklad pre tabuľku `ratings_staging` bol postup nasledujúci:

1. **Vytvorenie tabuľky**:

```sql
CREATE OR REPLACE TABLE ratings_staging (
    id INT,
    userId INT,
    movieId INT,
    rating FLOAT,
    timestamp STRING
);
```

2. **Nahranie dát**:

Dáta boli importované príkazom:

```sql
COPY INTO ratings_staging
FROM @my_stage/ratings.csv
FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1);
```

3. **Overenie správnosti operácií**:

Správnosť nahraných dát sa overovala dvoma spôsobmi:
- SQL dotazom, napríklad:  
  ```sql
  SELECT * FROM ratings_staging;
  ```
- Použitím Snowflake rozhrania: kliknutím na **Data > Databases > Tables > Tabuľka > Data Preview**, kde je možné prezerať obsah tabuľky. Navyše, v časti **Copy History** je zobrazený stav importovania súboru spolu s informáciou o úspešnosti operácie.

---

#### 3.2 Transformácia dát (Transform)

Transformácia zahŕňala prispôsobenie dát pre potreby dimenzionálneho modelovania. Nižšie sú uvedené príklady transformácií:

1. **Unifikácia údajov v dimenzionálnych tabuľkách**:

Príklad pre vytvorenie tabuľky dimenzie používateľov `DIM_USERS`:

```sql
INSERT INTO DIM_USERS (USER_ID, AGE_GROUP_ID, GENDER, OCCUPATION_ID)
SELECT 
    us.USER_ID,                  
    ag.ID AS AGE_GROUP_ID,       
    us.GENDER,                   
    os.ID AS OCCUPATION_ID       
FROM USERS_STAGING us
LEFT JOIN AGE_GROUP_STAGING ag
    ON us.AGE >= ag.ID AND us.AGE < ag.ID + 6  
LEFT JOIN OCCUPATIONS_STAGING os
    ON us.OCCUPATION_ID = os.ID; 
```

2. **Načítanie faktov do faktovej tabuľky `FACT_RATINGS`**:

```sql
INSERT INTO FACT_RATINGS (RATING_ID, RATING, RATED_AT, USER_ID, MOVIE_ID, DIM_TIME_TIME_ID)
SELECT 
    rs.RATING_ID,                 
    rs.RATING,                    
    rs.RATED_AT,                  
    rs.USER_ID,                   
    rs.MOVIE_ID,                  
    dt.TIME_ID                    
FROM RATINGS_STAGING rs
LEFT JOIN DIM_TIME dt
    ON dt.FULL_DATE = DATE(rs.RATED_AT)  
LEFT JOIN DIM_USERS du
    ON rs.USER_ID = du.USER_ID           
LEFT JOIN DIM_MOVIES dm
    ON rs.MOVIE_ID = dm.MOVIE_ID;
```

---

#### 3.3 Načítanie dát (Load)

Po transformáciách boli dáta načítané do cieľových tabuliek. Pre optimalizáciu procesu boli staging tabuľky po dokončení vymazané, aby sa uvoľnili zdroje. Na to sa použili nasledujúce príkazy:

```sql
DROP TABLE IF EXISTS AGE_GROUP_STAGING;
DROP TABLE IF EXISTS BRIDGE_GENRES_MOVIES;
DROP TABLE IF EXISTS GENRES_STAGING;
DROP TABLE IF EXISTS MOVIES_GENRES_STAGING;
DROP TABLE IF EXISTS MOVIES_STAGING;
DROP TABLE IF EXISTS OCCUPATIONS_STAGING;
DROP TABLE IF EXISTS RATINGS_RAW_STAGING;
DROP TABLE IF EXISTS RATINGS_STAGING;
DROP TABLE IF EXISTS USERS_RAW_STAGING;
DROP TABLE IF EXISTS USERS_STAGING;
```

---

Tento proces efektívne pripravil dáta na analýzu a následné vizualizácie, čím umožnil hlbšie porozumenie dát a podnikovým rozhodovacím procesom. Snowflake svojou flexibilitou a jednoduchosťou výrazne uľahčil celý postup.


