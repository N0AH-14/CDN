# 🐼 Pandas — Comprehensive Notes
> A complete reference for creating, handling, accessing, and operating on DataFrames and Series.

---

## Table of Contents
1. [What is Pandas?](#1-what-is-pandas)
2. [Core Data Structures](#2-core-data-structures)
3. [Creating DataFrames](#3-creating-dataframes)
4. [Reading & Writing Data (I/O)](#4-reading--writing-data-io)
5. [Inspecting DataFrames](#5-inspecting-dataframes)
6. [Accessing Data (Indexing & Selection)](#6-accessing-data-indexing--selection)
7. [Filtering & Boolean Masking](#7-filtering--boolean-masking)
8. [Modifying DataFrames](#8-modifying-dataframes)
9. [Handling Missing Data](#9-handling-missing-data)
10. [Sorting & Ranking](#10-sorting--ranking)
11. [Statistical Operations & Aggregations](#11-statistical-operations--aggregations)
12. [GroupBy Operations](#12-groupby-operations)
13. [Merging, Joining & Concatenating](#13-merging-joining--concatenating)
14. [Reshaping DataFrames (Pivot, Melt, Stack)](#14-reshaping-dataframes-pivot-melt-stack)
15. [String Operations](#15-string-operations)
16. [DateTime Operations](#16-datetime-operations)
17. [Apply, Map & Vectorized Operations](#17-apply-map--vectorized-operations)
18. [MultiIndex](#18-multiindex)
19. [Performance Tips](#19-performance-tips)
20. [Cheat Sheet Quick Reference](#20-cheat-sheet-quick-reference)

---

## 1. What is Pandas?

Pandas is a fast, powerful, and flexible open-source data analysis and manipulation library built on top of NumPy.

```python
import pandas as pd
import numpy as np
```

**Two primary objects:**
- `Series` — a 1D labeled array
- `DataFrame` — a 2D labeled table (like a spreadsheet or SQL table)

---

## 2. Core Data Structures

### Series
A one-dimensional array with an index.

```python
s = pd.Series([10, 20, 30, 40])
# 0    10
# 1    20
# 2    30
# 3    40

s = pd.Series([10, 20, 30], index=['a', 'b', 'c'])
# a    10
# b    20
# c    30

s['b']        # → 20
s[['a', 'c']] # → Series with a=10, c=30
s.values      # → numpy array: [10, 20, 30]
s.index       # → Index(['a', 'b', 'c'])
s.dtype       # → int64
s.name        # → None (can assign: s.name = 'scores')
```

### DataFrame
A two-dimensional table with labeled rows (index) and columns.

```python
df = pd.DataFrame({
    'name': ['Alice', 'Bob', 'Carol'],
    'age':  [25, 30, 27],
    'score': [88.5, 92.0, 79.5]
})

df.index    # → RangeIndex(start=0, stop=3, step=1)
df.columns  # → Index(['name', 'age', 'score'])
df.values   # → 2D numpy array
df.shape    # → (3, 3)  — (rows, cols)
df.dtypes   # → column-wise dtypes
```

---

## 3. Creating DataFrames

### From a Dictionary of Lists
```python
df = pd.DataFrame({
    'city':  ['NYC', 'LA', 'Chicago'],
    'pop':   [8.3, 4.0, 2.7],
    'state': ['NY', 'CA', 'IL']
})
```

### From a List of Dictionaries
```python
rows = [
    {'name': 'Alice', 'age': 25},
    {'name': 'Bob',   'age': 30},
]
df = pd.DataFrame(rows)
```

### From a List of Lists
```python
data = [['Alice', 25], ['Bob', 30], ['Carol', 27]]
df = pd.DataFrame(data, columns=['name', 'age'])
```

### From a NumPy Array
```python
arr = np.random.randn(4, 3)
df = pd.DataFrame(arr, columns=['A', 'B', 'C'])
```

### From a CSV / Excel (see Section 4)

### Setting a Custom Index
```python
df = pd.DataFrame(data, columns=['name', 'age'], index=['r1', 'r2', 'r3'])
# or after creation:
df.index = ['r1', 'r2', 'r3']
df = df.set_index('name')      # use a column as the index
df = df.reset_index()          # move index back to a column
```

### Empty DataFrame
```python
df = pd.DataFrame(columns=['name', 'age', 'score'])
```

### From a Range / Sequence
```python
df = pd.DataFrame({'x': range(10), 'y': np.arange(0, 1, 0.1)})
```

---

## 4. Reading & Writing Data (I/O)

### CSV
```python
df = pd.read_csv('data.csv')
df = pd.read_csv('data.csv',
    index_col=0,          # first column as index
    header=0,             # row 0 as header
    sep=',',              # delimiter
    na_values=['NA','?'], # treat as NaN
    dtype={'age': int},   # force column types
    usecols=['name','age'], # load only these columns
    nrows=100,            # only read first 100 rows
    skiprows=2,           # skip first 2 rows
    encoding='utf-8'
)
df.to_csv('output.csv', index=False)
```

### Excel
```python
df = pd.read_excel('data.xlsx', sheet_name='Sheet1')
df.to_excel('output.xlsx', sheet_name='Results', index=False)

# Multiple sheets
with pd.ExcelWriter('output.xlsx') as writer:
    df1.to_excel(writer, sheet_name='Sheet1')
    df2.to_excel(writer, sheet_name='Sheet2')
```

### JSON
```python
df = pd.read_json('data.json')
df.to_json('output.json', orient='records')
```

### SQL
```python
import sqlite3
conn = sqlite3.connect('database.db')
df = pd.read_sql('SELECT * FROM users', conn)
df.to_sql('new_table', conn, if_exists='replace', index=False)
```

### Parquet (efficient columnar format)
```python
df = pd.read_parquet('data.parquet')
df.to_parquet('output.parquet')
```

### Clipboard / HTML
```python
df = pd.read_clipboard()          # paste from clipboard
tables = pd.read_html('page.html') # returns a list of DataFrames
```

---

## 5. Inspecting DataFrames

```python
df.head(5)        # first 5 rows (default)
df.tail(5)        # last 5 rows
df.sample(5)      # 5 random rows
df.shape          # (rows, cols)
df.ndim           # 2
df.size           # rows × cols

df.info()         # column names, dtypes, null counts, memory
df.describe()     # count, mean, std, min, max, quartiles (numeric)
df.describe(include='all')  # includes object columns too
df.describe(include=[np.object_])  # only string columns

df.dtypes         # dtype per column
df.columns.tolist()    # list of column names
df.index.tolist()      # list of index values

df.nunique()      # unique value count per column
df.value_counts() # (on a Series) count of each value
df['city'].value_counts(normalize=True)  # as proportions

df.isnull().sum()       # null count per column
df.notnull().sum()      # non-null count per column

df.memory_usage(deep=True)  # memory per column in bytes
```

---

## 6. Accessing Data (Indexing & Selection)

### Selecting Columns
```python
df['name']              # Series
df[['name', 'age']]     # DataFrame (double brackets)
df.name                 # attribute access (avoid if col name has spaces)
```

### Selecting Rows — by Position: `.iloc`
`iloc` is **integer-location based** (like regular Python indexing).

```python
df.iloc[0]          # first row → Series
df.iloc[-1]         # last row
df.iloc[0:3]        # rows 0, 1, 2 (slice)
df.iloc[[0, 2, 4]]  # rows 0, 2, 4

# rows AND columns
df.iloc[0, 1]         # row 0, col 1 (scalar)
df.iloc[0:3, 1:3]     # rows 0-2, cols 1-2
df.iloc[:, 0]         # all rows, first column
df.iloc[:, [0, 2]]    # all rows, cols 0 and 2
```

### Selecting Rows — by Label: `.loc`
`loc` is **label-based**. The end of a slice is **inclusive**.

```python
df.loc[0]             # row with index label 0
df.loc[0:2]           # rows with labels 0, 1, 2 (inclusive!)
df.loc[[0, 2]]        # rows with labels 0 and 2

# rows AND columns
df.loc[0, 'name']         # scalar
df.loc[0:2, 'name':'age'] # rows 0-2, cols name through age
df.loc[:, 'age']          # all rows, 'age' column
df.loc[:, ['name','score']] # all rows, specific columns

# With a named index (e.g., set_index('name'))
df.loc['Alice']
df.loc['Alice', 'score']
```

### `.at` and `.iat` — Scalar Access (Fastest)
```python
df.at[0, 'name']    # label-based scalar
df.iat[0, 1]        # position-based scalar
```

### Boolean Indexing (see Section 7)
```python
df[df['age'] > 25]
```

---

## 7. Filtering & Boolean Masking

### Basic Conditions
```python
df[df['age'] > 25]
df[df['city'] == 'NYC']
df[df['score'].between(80, 95)]
df[df['name'].str.startswith('A')]
```

### Combining Conditions
```python
# Use & (AND), | (OR), ~ (NOT) — wrap each condition in parentheses!
df[(df['age'] > 25) & (df['score'] > 85)]
df[(df['city'] == 'NYC') | (df['city'] == 'LA')]
df[~(df['age'] > 30)]   # NOT: age <= 30
```

### `.isin()` — Filter by a List of Values
```python
df[df['city'].isin(['NYC', 'LA'])]
df[~df['city'].isin(['Chicago'])]   # exclude Chicago
```

### `.query()` — SQL-style String Filtering
```python
df.query('age > 25')
df.query('age > 25 and score > 85')
df.query('city in ["NYC", "LA"]')
df.query('city == @my_city')   # use Python variable with @
```

### `.where()` — Keep values where condition is True, else NaN
```python
df['score'].where(df['score'] > 80)        # NaN where score <= 80
df['score'].where(df['score'] > 80, other=0)  # 0 where score <= 80
```

### `.mask()` — Inverse of where
```python
df['score'].mask(df['score'] < 80, other=0)  # replace low scores with 0
```

---

## 8. Modifying DataFrames

### Adding / Modifying Columns
```python
df['grade'] = 'A'                         # constant
df['grade'] = df['score'].apply(lambda x: 'A' if x >= 90 else 'B')
df['age_sq'] = df['age'] ** 2             # derived column
df['full'] = df['first'] + ' ' + df['last']  # string concat

# Insert at a specific position
df.insert(loc=2, column='new_col', value=0)
```

### Renaming Columns
```python
df.rename(columns={'name': 'full_name', 'age': 'years'}, inplace=True)
df.columns = ['col1', 'col2', 'col3']   # rename all at once
df.columns = df.columns.str.lower()     # lowercase all names
df.columns = df.columns.str.replace(' ', '_')  # replace spaces
```

### Dropping Columns / Rows
```python
df.drop(columns=['col1', 'col2'], inplace=True)
df.drop(index=[0, 1], inplace=True)     # drop rows by label
df.drop(index=df[df['age'] < 18].index) # drop rows by condition
```

### Changing Data Types
```python
df['age'] = df['age'].astype(int)
df['score'] = df['score'].astype(float)
df['date'] = pd.to_datetime(df['date'])
df['category'] = df['category'].astype('category')  # memory efficient

# Multiple columns at once
df = df.astype({'age': int, 'score': float})
```

### Replacing Values
```python
df['city'].replace('NYC', 'New York', inplace=True)
df.replace({'NYC': 'New York', 'LA': 'Los Angeles'})
df['score'].replace(0, np.nan)   # replace 0 with NaN
```

### Updating Values with `.loc` / `.iloc`
```python
df.loc[0, 'age'] = 26
df.loc[df['city'] == 'NYC', 'pop'] = 8.5
df.iloc[0, 2] = 99.0
```

### Reordering / Selecting Columns
```python
df = df[['name', 'score', 'age']]         # reorder by listing
df = df.reindex(columns=['name', 'age', 'score', 'new'])  # adds NaN for missing
```

### Copying DataFrames
```python
df_copy = df.copy()       # deep copy (independent)
df_view = df              # NOT a copy — changes affect original
```

---

## 9. Handling Missing Data

Missing data is represented as `NaN` (float) or `NaT` (datetime).

### Detecting Missing Values
```python
df.isnull()             # True where NaN
df.notnull()            # True where not NaN
df.isnull().sum()       # count per column
df.isnull().sum().sum() # total count
df.isnull().any()       # True if any NaN in column
df.isnull().all()       # True if all NaN in column
```

### Dropping Missing Values
```python
df.dropna()                     # drop rows with ANY NaN
df.dropna(axis=1)               # drop COLUMNS with any NaN
df.dropna(how='all')            # drop rows where ALL values are NaN
df.dropna(subset=['age','score']) # drop rows with NaN in these cols
df.dropna(thresh=3)             # keep rows with at least 3 non-NaN values
```

### Filling Missing Values
```python
df.fillna(0)                         # fill all NaN with 0
df['score'].fillna(df['score'].mean()) # fill with mean
df.fillna({'age': 0, 'score': 50})   # different fill per column
df.fillna(method='ffill')            # forward fill (propagate last valid)
df.fillna(method='bfill')            # backward fill
df['score'].fillna(method='ffill', limit=2)  # fill max 2 consecutive NaNs
```

### Interpolation
```python
df['score'].interpolate()              # linear interpolation
df['score'].interpolate(method='polynomial', order=2)
```

### Replacing NaN with `np.nan`
```python
df.replace('', np.nan)     # empty string → NaN
df.replace('N/A', np.nan)
```

---

## 10. Sorting & Ranking

### Sorting by Values
```python
df.sort_values('age')                          # ascending
df.sort_values('age', ascending=False)         # descending
df.sort_values(['city', 'age'])                # sort by multiple cols
df.sort_values(['city', 'age'], ascending=[True, False])
df.sort_values('score', na_position='last')   # NaN at end (default)
df.sort_values('score', inplace=True)
```

### Sorting by Index
```python
df.sort_index()              # ascending index
df.sort_index(ascending=False)
```

### Ranking
```python
df['rank'] = df['score'].rank()              # average rank by default
df['rank'] = df['score'].rank(method='min')  # min rank for ties
df['rank'] = df['score'].rank(ascending=False)  # rank 1 = highest
# methods: 'average', 'min', 'max', 'first', 'dense'
```

### nlargest / nsmallest
```python
df.nlargest(3, 'score')     # top 3 rows by score
df.nsmallest(3, 'age')      # bottom 3 rows by age
df.nlargest(5, ['score', 'age'])  # break ties using 'age'
```

---

## 11. Statistical Operations & Aggregations

### Basic Stats (on DataFrame or Series)
```python
df['score'].mean()
df['score'].median()
df['score'].std()          # standard deviation
df['score'].var()          # variance
df['score'].sum()
df['score'].min()
df['score'].max()
df['score'].count()        # non-NaN count
df['score'].quantile(0.75) # 75th percentile
df['score'].cumsum()       # cumulative sum
df['score'].cumprod()      # cumulative product
df['score'].cummax()
df['score'].cummin()

# On full DataFrame (numeric columns only)
df.mean()
df.sum()
df.corr()                  # correlation matrix
df.cov()                   # covariance matrix
```

### Aggregation with `.agg()`
```python
df['score'].agg(['mean', 'std', 'min', 'max'])

df.agg({
    'score': ['mean', 'max'],
    'age':   ['min', 'median']
})
```

### Value Counts & Frequency
```python
df['city'].value_counts()
df['city'].value_counts(normalize=True)   # proportions
df['city'].value_counts(dropna=False)     # include NaN
```

### Cross-tabulation
```python
pd.crosstab(df['city'], df['category'])
pd.crosstab(df['city'], df['category'], normalize='index')  # row %
pd.crosstab(df['city'], df['category'], values=df['score'], aggfunc='mean')
```

---

## 12. GroupBy Operations

`groupby` splits the data into groups, applies a function, and combines results.

### Basic GroupBy
```python
grouped = df.groupby('city')
grouped['score'].mean()       # mean score per city
grouped['score'].sum()
grouped['score'].count()
grouped.size()                # count of rows per group

# Group by multiple columns
df.groupby(['city', 'category'])['score'].mean()
```

### `.agg()` with GroupBy
```python
df.groupby('city').agg({'score': 'mean', 'age': 'max'})

df.groupby('city').agg(
    avg_score=('score', 'mean'),
    max_age=('age', 'max'),
    total=('score', 'count')
)
```

### `.transform()` — Returns same-size result (broadcast back)
```python
# Add a column with group mean
df['city_avg'] = df.groupby('city')['score'].transform('mean')

# Normalize within group
df['norm_score'] = df.groupby('city')['score'].transform(
    lambda x: (x - x.mean()) / x.std()
)
```

### `.filter()` — Keep groups matching a condition
```python
# Keep only cities with more than 10 entries
df.groupby('city').filter(lambda x: len(x) > 10)
```

### `.apply()` — Apply a function to each group
```python
def top_2(group):
    return group.nlargest(2, 'score')

df.groupby('city').apply(top_2)
```

### Iterate over Groups
```python
for name, group_df in df.groupby('city'):
    print(name, group_df.shape)
```

### `get_group()`
```python
df.groupby('city').get_group('NYC')
```

---

## 13. Merging, Joining & Concatenating

### `pd.concat()` — Stack DataFrames
```python
# Stack vertically (row-wise)
result = pd.concat([df1, df2], ignore_index=True)
result = pd.concat([df1, df2], ignore_index=True, axis=0)

# Stack horizontally (column-wise)
result = pd.concat([df1, df2], axis=1)

# Keep only common rows/columns
result = pd.concat([df1, df2], join='inner')  # default is 'outer'
```

### `pd.merge()` — SQL-style Joins
```python
# Inner join (default)
result = pd.merge(df_left, df_right, on='id')

# Left join
result = pd.merge(df_left, df_right, on='id', how='left')

# Right join
result = pd.merge(df_left, df_right, on='id', how='right')

# Outer join
result = pd.merge(df_left, df_right, on='id', how='outer')

# Join on different column names
result = pd.merge(df_left, df_right, left_on='user_id', right_on='id')

# Join on multiple columns
result = pd.merge(df_left, df_right, on=['city', 'date'])

# Suffix for overlapping column names
result = pd.merge(df_left, df_right, on='id', suffixes=('_left', '_right'))
```

### `.join()` — Join on Index
```python
df1.join(df2)                         # left join on index
df1.join(df2, how='inner')
df1.join(df2, lsuffix='_l', rsuffix='_r')
df1.join([df2, df3])                  # join multiple at once
```

### Merging on Index
```python
pd.merge(df1, df2, left_index=True, right_index=True)
pd.merge(df1, df2, left_on='id', right_index=True)
```

---

## 14. Reshaping DataFrames (Pivot, Melt, Stack)

### `pivot_table()` — Summarize Like a Spreadsheet Pivot
```python
pivot = df.pivot_table(
    values='score',
    index='city',
    columns='category',
    aggfunc='mean',
    fill_value=0
)
```

### `pivot()` — Reshape Without Aggregation
```python
# Requires unique row/col combinations
df_wide = df.pivot(index='date', columns='city', values='score')
```

### `melt()` — Wide to Long Format
```python
df_long = pd.melt(
    df,
    id_vars=['name'],           # columns to keep
    value_vars=['math', 'sci'], # columns to unpivot
    var_name='subject',
    value_name='score'
)
```

### `stack()` — Columns → Row Index (Long format)
```python
df_stacked = df.stack()   # columns become innermost index level
df_stacked.unstack()      # undo stack
df_stacked.unstack(level=0)  # choose which level to unstack
```

### `pd.get_dummies()` — One-Hot Encoding
```python
pd.get_dummies(df['city'])
pd.get_dummies(df, columns=['city', 'category'])
pd.get_dummies(df['city'], prefix='city', drop_first=True)
```

### `pd.cut()` and `pd.qcut()` — Binning
```python
# Equal-width bins
df['age_group'] = pd.cut(df['age'], bins=3)
df['age_group'] = pd.cut(df['age'], bins=[0, 18, 35, 60, 100],
                          labels=['teen', 'young', 'adult', 'senior'])

# Equal-frequency bins (quantile-based)
df['score_quartile'] = pd.qcut(df['score'], q=4, labels=['Q1','Q2','Q3','Q4'])
```

---

## 15. String Operations

Accessed via `.str` accessor on a Series of strings.

```python
s = df['name']

s.str.lower()
s.str.upper()
s.str.title()
s.str.strip()          # strip whitespace
s.str.lstrip()
s.str.rstrip()

s.str.len()            # length of each string
s.str.contains('Ali')  # True/False
s.str.startswith('A')
s.str.endswith('e')
s.str.replace('Alice', 'Alicia')
s.str.replace(r'\d+', '', regex=True)  # regex replace

s.str.split(',')             # split → list
s.str.split(',', expand=True) # split → new columns
s.str.get(0)                 # get first element of split list

s.str.slice(0, 3)    # first 3 characters
s.str[0:3]           # equivalent shorthand

s.str.count('l')     # count occurrences
s.str.find('li')     # position of first occurrence

s.str.cat(sep=', ')  # concatenate all strings into one
df['full'] = df['first'].str.cat(df['last'], sep=' ')

s.str.extract(r'(\d+)')        # capture groups → DataFrame
s.str.extractall(r'(\d+)')
s.str.match(r'^\d{3}-\d{4}$') # True if full match
```

### Handling Mixed/Dirty Strings
```python
df['col'] = df['col'].str.strip().str.lower().str.replace(' ', '_')
```

---

## 16. DateTime Operations

### Parsing Dates
```python
df['date'] = pd.to_datetime(df['date'])
df['date'] = pd.to_datetime(df['date'], format='%Y-%m-%d')
df['date'] = pd.to_datetime(df['date'], errors='coerce')  # NaT on error
```

### Accessing DateTime Components (`.dt` accessor)
```python
df['date'].dt.year
df['date'].dt.month
df['date'].dt.day
df['date'].dt.hour
df['date'].dt.minute
df['date'].dt.second
df['date'].dt.dayofweek    # 0=Monday, 6=Sunday
df['date'].dt.day_name()   # 'Monday', 'Tuesday', ...
df['date'].dt.quarter
df['date'].dt.is_month_end
df['date'].dt.is_leap_year
df['date'].dt.date         # date part only
df['date'].dt.time         # time part only
```

### Date Arithmetic
```python
df['date'] + pd.Timedelta(days=7)
df['end'] - df['start']                          # → Timedelta
(df['end'] - df['start']).dt.days                # number of days
(df['end'] - df['start']).dt.total_seconds()
```

### Date Ranges & Offsets
```python
pd.date_range(start='2024-01-01', end='2024-12-31', freq='D')  # daily
pd.date_range(start='2024-01-01', periods=12, freq='MS')        # monthly start
pd.date_range(start='2024-01-01', periods=4, freq='QS')         # quarterly
```

### Resampling Time Series
```python
df.set_index('date', inplace=True)

df['score'].resample('M').mean()   # monthly average
df['score'].resample('W').sum()    # weekly sum
df['score'].resample('Q').agg(['mean', 'max'])
```

### Filtering by Date
```python
df[df['date'] >= '2024-01-01']
df[df['date'].between('2024-01-01', '2024-06-30')]
df.loc['2024']                     # all of 2024 (with DatetimeIndex)
df.loc['2024-01':'2024-06']        # Jan–Jun 2024
```

---

## 17. Apply, Map & Vectorized Operations

### `.apply()` — Apply a function along an axis
```python
# On a Series
df['score'].apply(lambda x: x * 2)
df['score'].apply(lambda x: 'Pass' if x >= 50 else 'Fail')

# Custom function
def grade(x):
    if x >= 90: return 'A'
    elif x >= 80: return 'B'
    elif x >= 70: return 'C'
    else: return 'F'
df['grade'] = df['score'].apply(grade)

# On a DataFrame (axis=1 = row-wise)
df.apply(lambda row: row['score'] + row['bonus'], axis=1)

# On a DataFrame (axis=0 = column-wise, default)
df[['score', 'bonus']].apply(lambda col: col.max() - col.min())
```

### `.map()` — Element-wise on a Series
```python
df['city'].map({'NYC': 'New York', 'LA': 'Los Angeles'})
df['score'].map(lambda x: round(x, 1))
```

### `.applymap()` / `.map()` on DataFrame (element-wise)
```python
# pandas >= 2.1: use df.map()
df[['score', 'bonus']].map(lambda x: round(x, 2))
```

### Vectorized Operations (Preferred for Performance)
```python
# These are fast — avoid apply() when vectorized exists
df['score'] * 2
df['score'] + df['bonus']
np.log(df['score'])
np.where(df['score'] >= 90, 'A', 'B')   # conditional assignment

# np.select for multiple conditions
conditions  = [df['score'] >= 90, df['score'] >= 80, df['score'] >= 70]
choices     = ['A', 'B', 'C']
df['grade'] = np.select(conditions, choices, default='F')
```

### `pd.cut()` as Alternative to apply for Bucketing
```python
df['bucket'] = pd.cut(df['score'], bins=[0, 60, 80, 100], labels=['Low','Mid','High'])
```

---

## 18. MultiIndex

A MultiIndex (hierarchical index) allows more than one level of indexing.

### Creating a MultiIndex
```python
arrays = [['NYC', 'NYC', 'LA', 'LA'], ['2023', '2024', '2023', '2024']]
index = pd.MultiIndex.from_arrays(arrays, names=['city', 'year'])
df = pd.DataFrame({'score': [80, 85, 75, 78]}, index=index)

# From groupby
df_mi = df.groupby(['city', 'year'])['score'].mean()  # returns MultiIndex Series
df_mi = df_mi.reset_index()  # flatten back to columns
```

### Accessing MultiIndex Data
```python
df.loc['NYC']              # all NYC rows
df.loc[('NYC', '2024')]    # specific combo
df.loc['NYC', '2024']      # same
df.xs('2024', level='year') # cross-section by level name
```

### Manipulating MultiIndex
```python
df.swaplevel()                    # swap levels
df.sort_index()                   # sort by index
df.reset_index()                  # back to flat columns
df.unstack()                      # innermost level → columns
df.stack()                        # column level → innermost index
```

---

## 19. Performance Tips

### Use Vectorized Operations
```python
# SLOW:
df['result'] = df['a'].apply(lambda x: x * 2)
# FAST:
df['result'] = df['a'] * 2
```

### Use Categorical Dtype for Repeated Strings
```python
df['city'] = df['city'].astype('category')  # saves memory, speeds groupby
```

### Use `query()` for Filtering
```python
df.query('age > 25 and score > 80')   # often faster than boolean indexing
```

### Avoid Chained Indexing
```python
# BAD (SettingWithCopyWarning risk):
df[df['city'] == 'NYC']['score'] = 100
# GOOD:
df.loc[df['city'] == 'NYC', 'score'] = 100
```

### Read Only Needed Columns
```python
df = pd.read_csv('big_file.csv', usecols=['name', 'age'])
```

### Use Chunking for Large Files
```python
chunks = pd.read_csv('huge.csv', chunksize=10000)
result = pd.concat([chunk[chunk['age'] > 25] for chunk in chunks])
```

### Use `inplace=True` Sparingly
The `inplace=True` parameter modifies in place but does NOT save memory and can cause issues with chained operations. Prefer assignment:
```python
df = df.dropna()   # preferred over df.dropna(inplace=True)
```

### Checking Memory Usage
```python
df.memory_usage(deep=True).sum() / 1024**2  # in MB
```

---

## 20. Cheat Sheet Quick Reference

| Task | Code |
|---|---|
| Load CSV | `pd.read_csv('file.csv')` |
| Save CSV | `df.to_csv('out.csv', index=False)` |
| First N rows | `df.head(N)` |
| Last N rows | `df.tail(N)` |
| Shape | `df.shape` |
| Column names | `df.columns.tolist()` |
| Summary stats | `df.describe()` |
| Select column | `df['col']` |
| Select columns | `df[['a', 'b']]` |
| Row by position | `df.iloc[i]` |
| Row by label | `df.loc[label]` |
| Filter | `df[df['age'] > 25]` |
| Query | `df.query('age > 25')` |
| Drop column | `df.drop(columns=['col'])` |
| Drop NaN | `df.dropna()` |
| Fill NaN | `df.fillna(0)` |
| Rename cols | `df.rename(columns={'old':'new'})` |
| Add column | `df['new'] = values` |
| Sort | `df.sort_values('col')` |
| Group & aggregate | `df.groupby('city')['score'].mean()` |
| Merge | `pd.merge(df1, df2, on='id')` |
| Concat rows | `pd.concat([df1, df2])` |
| Pivot table | `df.pivot_table(values, index, columns, aggfunc)` |
| One-hot encode | `pd.get_dummies(df['col'])` |
| Apply function | `df['col'].apply(func)` |
| String contains | `df['col'].str.contains('text')` |
| Parse dates | `pd.to_datetime(df['date'])` |
| Correlation | `df.corr()` |
| Value counts | `df['col'].value_counts()` |
| Unique values | `df['col'].unique()` |
| Null count | `df.isnull().sum()` |
| Cast type | `df['col'].astype(int)` |
| Copy | `df.copy()` |

---

*These notes cover the full lifecycle of working with pandas — from loading data to complex transformations. Practice each section with real datasets for best retention.*
