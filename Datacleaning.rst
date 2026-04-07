.. code:: ipython3

    import pandas as pd
    import numpy as np
    import re
    from dateutil import parser as dateparser

1. LOAD

.. code:: ipython3

    df = pd.read_csv(r"D:\Vidhya Learning\dataset\messy_IMDB_dataset.csv",
        encoding="latin1",
        sep=";",
        on_bad_lines="skip",
    )
    ## prints no of rows and columns in the dataset
    print(f"Loaded: {df.shape[0]} rows × {df.shape[1]} columns") 
    ## remove leading/trailing spaces
    


.. parsed-literal::

    Loaded: 101 rows × 12 columns
    

.. code:: ipython3

    df.columns




.. parsed-literal::

    Index(['IMBD title ID', 'Original titlÊ', 'Release year', 'Genrë¨', 'Duration',
           'Country', 'Content Rating', 'Director', 'Unnamed: 8', 'Income',
           ' Votes ', 'Score'],
          dtype='object')



2. RENAME GARBLED COLUMN NAMES

.. code:: ipython3

    df.columns = df.columns.str.strip()
    df = df.rename(columns={
        "IMBD title ID":  "title_id",
        "Original titlÊ": "title",
        "Release year":   "release_date",
        "Genrë¨":         "genre",
        "Duration":       "duration_min",
        "Country":        "country",
        "Content Rating": "content_rating",
        "Director":       "director",
        "Unnamed: 8":     "_drop",
        "Income":         "income_usd",
        "Votes":         "votes",
        "Score":          "score",
    })
    print(f"Columns after renaming: {df.columns.tolist()}")


.. parsed-literal::

    Columns after renaming: ['title_id', 'title', 'release_date', 'genre', 'duration_min', 'country', 'content_rating', 'director', '_drop', 'income_usd', 'votes', 'score']
    

3. DROP GHOST COLUMN (100% null)

.. code:: ipython3

    df = df.drop(columns=["_drop"], errors="ignore") #Drop ghost column (Unnamed: 8) which is 100% null
    

4. RELEASE DATE -> YEAR (kept as float64 until final cast in step 13)

.. code:: ipython3

    def extract_year(raw):
        if pd.isna(raw): #if value is missing, return NaN
            return np.nan
        raw = str(raw).strip()
        # Try to extract a 4-digit year directly first (fast path)
        match = re.search(r"\b(1[89]\d{2}|20\d{2})\b", raw) #- Looks for a 4-digit year in the range 1800–2099.
        if match:
            return float(match.group()) #- If regex finds a match → return the year as an integer.
        # Fallback: try dateutil parser for natural language dates
        try:
            return float(dateparser.parse(raw, fuzzy=True).year) #If regex fails,interpret the string as a normal date and extract year.
        except Exception:
            return np.nan
    df["release_year"] = df["release_date"].apply(extract_year)
    
    ## Drop the original release_date column as we now have release_year
    df = df.drop(columns=["release_date"])
    df.columns = df.columns.str.strip()          # remove leading/trailing spaces
    

5. SCORE -> float64 (Handles: “8,9f” “8..8” “8:8” “++8.7” “8.7.” “9,.”
   “08.9”)

.. code:: ipython3

    def clean_score(raw):
        if pd.isna(raw): #if value is missing, return NaN
            return np.nan
        s = str(raw).strip()
        
        s = re.sub(r"[^0-9.\-]", "", s) # Keep only digits, dots, minus(remove letters, commas, etc.)
        s = re.sub(r"\.{2,}", ".", s) # Remove duplicate dots (e.g. "8..8" → "8.8")
        s = s.rstrip(".")# Remove trailing dots (e.g. "8.7." → "8.7")
        try:
            val = float(s)
            return val if 1.0 <= val <= 10.0 else np.nan # Scores should be between 1 and 10 else return Nan
        except ValueError:
            return np.nan
    
    df["score"] = df["score"].apply(clean_score)
    

6. INCOME -> float64 Handles: “$ 4o8,035,783” leading $ commas OCR ‘o’
   vs ‘0’

.. code:: ipython3

    def clean_income(raw):
        if pd.isna(raw):
            return np.nan
        s = str(raw).strip()
        s = re.sub(r"[\$\s]", "", s) # Remove $ and spaces
        s = s.replace("o", "0").replace("O", "0") # Fix common OCR errors: letter 'o' instead of '0'
        s = s.replace(",", "") # Remove commas used as thousand separators
        try:
            val = float(s)
            return val if val > 0 else np.nan
        except ValueError:
            return np.nan
    
    df["income_usd"] = df["income_usd"].apply(clean_income)
    

7. DURATION -> float64 Handles: “Inf” “Nan” “-” ” ” “Not Applicable”
   “178c”

.. code:: ipython3

    
    def clean_duration(raw):
        if pd.isna(raw):
            return np.nan
        s = str(raw).strip()
        if s in ("", "-", "Nan", "Inf", "Not Applicable"):
            return np.nan
        
        s = re.sub(r"[^\d]", "", s) # Remove trailing non-numeric characters (e.g. "178c" → "178")
        try:
            val = float(s)
            
            return val if 60 <= val <= 300 else np.nan # Sanity check: movies are typically 60–300 min
        except ValueError:
            return np.nan
    
    df["duration_min"] = df["duration_min"].apply(clean_duration)

8. COUNTRY -> standardised name Handles: “US” “US.” “New Zesland” “New
   Zeland” “Italy1” “West Germany”

.. code:: ipython3

    
    COUNTRY_MAP = {
        "US":           "USA",
        "US.":          "USA",
        "New Zesland":  "New Zealand",
        "New Zeland":   "New Zealand",
        "Italy1":       "Italy",
        "West Germany": "Germany",
    }
    
    def clean_country(raw):
        if pd.isna(raw):
            return np.nan
        s = str(raw).strip().rstrip(".")
        return COUNTRY_MAP.get(s, s)
    
    df["country"] = df["country"].apply(clean_country)
    

9. VOTES -> float64 Handles European dot-thousands: “2.278.845” ->
   2278845.0

.. code:: ipython3

    
    def clean_votes(raw):
        if pd.isna(raw):
            return np.nan
        s = str(raw).strip()
        
        s = s.replace(".", "").replace(",", "") # European format: dots as thousands separators
        try:
            return float(s)
        except ValueError:
            return np.nan
    
    df["votes"] = df["votes"].apply(clean_votes)
    

10. STRIP WHITESPACE from all remaining string columns

.. code:: ipython3

    str_cols = df.select_dtypes(include="object").columns
    df[str_cols] = df[str_cols].apply(lambda c: c.str.strip())
    

11. Handle remaining missing values Drop rows where the primary key is
    missin

.. code:: ipython3

    #df.isnull(subset=["title_id", "title"]).sum()
    df = df.dropna(subset=["title_id", "title"])
    

12. FILL MISSING — content_rating Fill content_rating with ‘Unknown’ (24
    missing — too many to drop)

.. code:: ipython3

    df["content_rating"] = df["content_rating"].fillna("Unknown")

13. FILL + CAST numeric columns Numeric columns: fill with median
    (conservative choice) RULE: always fillna() WHILE the column is
    still float64, then round() and astype(“Int64”).

.. code:: ipython3

    
    df["score"] = df["score"].fillna(df["score"].median()).round(1)
    
    # Numeric columns: fill with median (conservative choice)
    for col in ["duration_min", "income_usd", "votes"]:
        df[col] = df[col].fillna(df[col].median())    # fill while float64
        df[col] = df[col].round(0).astype("Int64")    # now safe to cast
    
    # Release year: fill with median year
    df["release_year"] = (
        df["release_year"]
        .fillna(df["release_year"].median())
        .round(0)
        .astype("Int64")
    )
    


.. parsed-literal::

    c:\users\vsvid\appdata\local\programs\python\python39\lib\site-packages\numpy\lib\_nanfunctions_impl.py:1231: RuntimeWarning: Mean of empty slice
      return np.nanmean(a, axis, out=out, keepdims=keepdims)
    

14. SAVE

.. code:: ipython3

    
    df["duration_min"] = df["duration_min"].astype("Int64")
    df["votes"]        = df["votes"].astype("Int64")
    df["income_usd"]   = df["income_usd"].astype("Int64")
    df["score"]        = df["score"].round(1)
    
    df.columns = df.columns.str.title()  # ensure no leading/trailing spaces in column names
    df["Title"]= df["Title"].str.strip() # remove leading/trailing spaces from title column 
    

13. Save cleaned dataset

.. code:: ipython3

    output_path = "D:\\Vidhya Learning\\DecodeLabs\\imdb_cleaned.csv"
    df.to_csv(output_path, index=False)
    print(f"\nCleaned dataset saved to: {output_path}")


.. parsed-literal::

    
    Cleaned dataset saved to: D:\Vidhya Learning\DecodeLabs\imdb_cleaned.csv
    

14. Summary report

.. code:: ipython3

    print("\n===== CLEANING SUMMARY =====")
    print(f"Final shape : {df.shape[0]} rows × {df.shape[1]} columns")
    print("\nColumn dtypes:")
    print(df.dtypes)
    print("\nMissing values after cleaning:")
    print(df.isnull().sum())
    print("\nSample rows:")
    print(df.head(5).to_string())
    


.. parsed-literal::

    
    ===== CLEANING SUMMARY =====
    Final shape : 100 rows × 11 columns
    
    Column dtypes:
    Title_Id           object
    Title              object
    Genre              object
    Duration_Min        Int64
    Country            object
    Content_Rating     object
    Director           object
    Income_Usd          Int64
    Votes               Int64
    Score             float64
    Release_Year        Int64
    dtype: object
    
    Missing values after cleaning:
    Title_Id            0
    Title               0
    Genre               0
    Duration_Min      100
    Country             0
    Content_Rating      0
    Director            0
    Income_Usd          0
    Votes               0
    Score               0
    Release_Year        0
    dtype: int64
    
    Sample rows:
        Title_Id                     Title                 Genre  Duration_Min Country Content_Rating              Director  Income_Usd     Votes  Score  Release_Year
    0  tt0111161  The Shawshank Redemption                 Drama          <NA>     USA              R        Frank Darabont    28815245  22788450    9.3          1995
    1  tt0068646             The Godfather          Crime, Drama          <NA>     USA              R  Francis Ford Coppola   246120974  15726740    9.2          1972
    2  tt0468569           The Dark Knight  Action, Crime, Drama          <NA>     USA          PG-13     Christopher Nolan  1005455211  22416150    9.0          2008
    3  tt0071562    The Godfather: Part II          Crime, Drama          <NA>     USA              R  Francis Ford Coppola   408035783  10987140    9.0          1975
    4  tt0110912              Pulp Fiction          Crime, Drama          <NA>     USA              R     Quentin Tarantino   222831817  17801470    8.2          1994
    

