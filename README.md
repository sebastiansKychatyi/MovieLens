
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

## 3. ETL proces v Snowflake

### 3.1 Príprava a import dát
#### 1. Extract (Extrahovanie dát)
- Týmto príkazom hovoríme, že používame databázu GECKO_DB a schému MOVIELENS:
  ```sql
  USE DATABASE GECKO_DB;
  USE SCHEMA GECKO_DB.MOVIELENS;
  ```
  ```sql
  CREATE OR REPLACE STAGE age_group_staging;
  ```
  - Importovanie dat
  ```sql
  COPY INTO age_group
  FROM @movielens_stage/age_group.csv
  FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1);
  ```
  - Overenie spravnosti operacii
  ```sql
  SELECT * FROM age_group_staging;
  ```

# Analýza Databázy Filmov

## 3.2 Výsledky analýzy

### Top 10 filmov s najvyšším hodnotením

Nasledujúci SQL dotaz zobrazuje desať filmov s najvyšším priemerným hodnotením. Tento dotaz pomáha identifikovať najpopulárnejšie filmy v našej databáze na základe používateľských hodnotení.

```sql
SELECT 
    m.title,
    mar.avg_rating,
    mar.total_ratings
FROM movie_avg_ratings mar
JOIN movies m ON mar.movie_id = m.id
ORDER BY mar.avg_rating DESC
LIMIT 10;
```

### Top 10 žánrov podľa počtu filmov

Nasledujúci SQL dotaz ukazuje desať žánrov s najväčším počtom filmov v databáze. Tento prehľad pomáha pochopiť, ktoré žánry sú najpopulárnejšie.

```sql
SELECT 
    g.name AS genre_name,
    COUNT(gm.movie_id) AS total_movies
FROM genres g
JOIN genres_movies gm ON g.id = gm.genre_id
GROUP BY g.name
ORDER BY total_movies DESC
LIMIT 10;
```

### Top 10 užívateľov podľa počtu hodnotení

Tento dotaz identifikuje desať užívateľov, ktorí poskytli najviac hodnotení, čo môže byť užitočné na posúdenie úrovne užívateľskej aktivity a zapojenia do hodnotiaceho procesu.

```sql
SELECT 
    u.id AS user_id,
    COUNT(r.id) AS total_ratings
FROM ratings r
JOIN users u ON r.user_id = u.id
GROUP BY u.id
ORDER BY total_ratings DESC
LIMIT 10;
```

### Počet filmov vydaných každý rok

Nasledujúci dotaz poskytuje prehľad o počte filmov vydaných každý rok, čo umožňuje sledovať trendy vo filme produkcie a ich dynamiku cez čas.

```sql
SELECT 
    release_year,
    COUNT(*) AS total_movies
FROM movies
GROUP BY release_year
ORDER BY release_year ASC;
```

### 3.3 Vizualizácie
* Navrhnite vizualizácie, ktoré ukazujú:
- Priemerné hodnotenia filmov.
- Počet filmov podľa žánrov.
- Preferencie používateľov podľa vekových skupín.
## 4. Vizualizácia dát

V tejto sekcii projektu sa zameriavame na prezentáciu vizuálnych analýz, ktoré poskytujú hlbšie pochopenie údajov o filmoch a používateľoch z databázy MovieLens. Vizualizácie sú navrhnuté tak, aby odpovedali na kľúčové otázky a odhalili zaujímavé trendy a vzorce v dátach.

### 4.1 Priemerné hodnotenia filmov
Popis: Graf zobrazuje priemerné hodnotenia pre top hodnotené filmy.  
Účel: Identifikuje filmy s najvyššími hodnoteniam!
i, čo môže pomôcť pri odporúčaniach pre používateľov alebo pri analýze kvality filmových obsahov.
[graf_1](https://github.com/user-attachments/assets/2e77c44a-9503-40ea-a6d1-286d74b7c483)

### 4.2 Počet filmov podľa žánrov
Popis: Tento graf prezentuje rozdelenie filmov podľa žánrov.  
Účel: Poskytuje prehľad o popularite jednotlivých žánrov, čo môže byť užitočné pre filmové distribučné spoločnosti alebo marketérov.
![graf_2](https://github.com/user-attachments/assets/f77646f6-16b9-4a71-aca3-bf506b247a94)


### 4.3 Rozloženie hodnotení filmov
Popis: Zobrazuje distribúciu hodnotení medzi používateľmi.  
Účel: Analyzuje ako používatelia hodnotia filmy a môže odhaliť prípadnú prísťnosť alebo štedrosť v ich hodnoteniach.
![graf_3](https://github.com/user-attachments/assets/4e5b26f4-2f43-4f83-a14b-94a446c94dd0)

### 4.4 Počet hodnotení podľa vekových skupín
Popis: Graf ukazuje počet hodnotení podľa vekových skupín používateľov.  
Účel: Skúma, ako vekové rozdiely ovplyvňujú filmové preferencie a hodnotenie.
![graf_4](https://github.com/user-attachments/assets/4027f690-c91d-4261-94d3-9c77261d5751)


### 4.5 Historický vývoj počtu filmov
Popis: Tento graf ilustruje trend v počte vydaných filmov počas rôznych rokov.  
Účel: Umožňuje sledovať ako sa kinematografia vyvíjala v čase a reaguje na zmeny v kultúrnych a technologických trendoch.
![graf_5](https://github.com/user-attachments/assets/1dc051fd-356c-49e6-a5d3-20ac5771c776)


### Náhľady vizualizácií
![5_grafikov](https://github.com/user-attachments/assets/89689c63-84d7-4af7-8b0d-83b370f421a7)

