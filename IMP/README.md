#### Data Frame commands
- **`df.show(n=20)`**, truncate=True, vertical=False) : display the contents of a DataFrame in a tabular format in the console
    truncate is true the it will display 20 character only, if false whole date of a column , if any number then n0. of characters
- **`print(df)`** :-	Prints the entire DataFrame (may truncate if large).
- **`df.head(n)`** :-	Shows first n rows (default 5).
- **`df.tail(n)`**	:- Shows last n rows.
- **`df.sample(n)`**	:- Shows n random rows.
- **`display(df)`**	:- (In Jupyter/IPython) Renders as a nice HTML table.
- **`df.columns`** :- Print all the column names in list format9['a', 'b', 'c', 'd', 'e']0
- **`df.printSchema()`** :- list the columns and there types
```
root
 |-- a: long (nullable = true)
 |-- b: double (nullable = true)
 |-- c: string (nullable = true)
 |-- d: date (nullable = true)
 |-- e: timestamp (nullable = true)
```
- **`DataFrame.collect()`** :- collects the distributed data to the driver side as the local data in Python. Note that this can throw an out-of-memory error when the dataset is too large to fit in the driver side because it collects all the data from executors to the driver side.
- **`DataFrame.take()/DataFrame.tail()`** :- In order to avoid throwing an out-of-memory exception, use DataFrame.take() or DataFrame.tail().
- **`df.toPandas()`** :- conversion back to a pandas DataFrame to leverage pandas API.
- **`df.select(df.c).show()`** :- It will select the column c from the rdd df
- **`df.withColumn('upper_c', upper(df.c)).show()`** :- It will add 1 more column into the df rdd with name as upper_c and value as df.c columns in upper case
- **`df.filter(df.a == 1).show()`** :- select a subset of rowsbase on the columns codition.
-**`df.filter((col("age") < 30) | (col("name") == "Eve"))`**  : If we need multiple filter conditions
  & → AND
| → OR
~ → NOT

- **`df.filter((col("age") < 30) )`**function returns a Column object based on the given column name
## Applying a Function

 - **`UDF FUNCTIONS`** : Below code will do +1 to df.a columns
```
   @pandas_udf('long')
def pandas_plus_one(series: pd.Series) -> pd.Series:
    //Simply plus one by using pandas Series.
    return series + 1

df.select(pandas_plus_one(df.a)).show()
```
- **`DataFrame.mapInPandas()`** :- It allows users directly use the APIs in a pandas DataFrame without any restrictions such as the result length.
```
def pandas_filter_func(iterator):
    for pandas_df in iterator:
        yield pandas_df[pandas_df.a == 1]

df.mapInPandas(pandas_filter_func, schema=df.schema).show()
```

## Grouping Data
- **`df.groupby('color').avg().show()`** : It group the data based on column color and give avg of numberi field as oupput
- **`df.groupBy("Category", "Type").agg(
        sum("Value").alias("TotalValue"),
        avg("Value").alias("AverageValue"),
        count("*").alias("RowCount")`**
  It will first group based on the columns Category and Type and then find sum,avg and count

- **`.cogroup()`** :- In PySpark, the cogroup() transformation is used to group values from two or more Pair RDDs (RDDs of key-value pairs) by their keys.
It returns a new RDD where each key is associated with a tuple of iterables — one iterable for each RDD’s values for that key.

```
df1 = spark.createDataFrame(
    [(20000101, 1, 1.0), (20000101, 2, 2.0), (20000102, 1, 3.0), (20000102, 2, 4.0)],
    ('time', 'id', 'v1'))

df2 = spark.createDataFrame(
    [(20000101, 1, 'x'), (20000101, 2, 'y')],
    ('time', 'id', 'v2'))

def merge_ordered(l, r):
    return pd.merge_ordered(l, r)

df1.groupby('id').cogroup(df2.groupby('id')).applyInPandas(
    merge_ordered, schema='time int, id int, v1 double, v2 string').show()

OUTPUT 
+--------+---+---+---+
|    time| id| v1| v2|
+--------+---+---+---+
|20000101|  1|1.0|  x|
|20000102|  1|3.0|  x|
|20000101|  2|2.0|  y|
|20000102|  2|4.0|  y|
+--------+---+---+---+
```

# Getting Data In/Out
- df.write.csv('foo.csv', header=True)  : It will write header also
- spark.read.csv('foo.csv', header=True,inferSchema=True).show() :- It automatically detects column data types instead of treating all as strings. and If you omit inferSchema, all columns will be read as strings by default.
- df.write.parquet('bar.parquet')
- spark.read.parquet('bar.parquet').show()
- df.write.orc('zoo.orc')
- spark.read.orc('zoo.orc').show()

# Pandas API on Spark

```
import pandas as pd
import numpy as np
import pyspark.pandas as ps
from pyspark.sql import SparkSession
```
- **`s = ps.Series([1, 3, 5, np.nan, 6, 8]) `** :- Creating a pandas-on-Spark Series by passing a list of values
```
s
```
```
0    1.0
1    3.0
2    5.0
3    NaN
4    6.0
5    8.0
dtype: float64
```

- Creating a pandas-on-Spark DataFrame by passing a dict of objects that can be converted to series-like.
  
``
psdf = ps.DataFrame(
    {'a': [1, 2, 3, 4, 5, 6],
     'b': [100, 200, 300, 400, 500, 600],
     'c': ["one", "two", "three", "four", "five", "six"]},
    index=[10, 20, 30, 40, 50, 60])
``

```
psdf
	a	b	c
10	1	100	one
20	2	200	two
30	3	300	three
40	4	400	four
50	5	500	five
60	6	600	six

```

- **`dates = pd.date_range('20130101', periods=6)`** : it will generate a list of 5 dates, starting from mentioned date
- **`psdf = sdf.pandas_api()`** :- Creating pandas-on-Spark DataFrame from Spark DataFrame.
- **`psdf.T`** :- Transposing your data(means row will change to columns and vice-versa)
- **`psdf.sort_index(ascending=False)`** :- sorting by index
- **`psdf.sort_values(by='B')`** :- sorting bu value

- Pandas API on Spark primarily uses the value np.nan to represent missing data. It is by default not included in computations.
- Use the reindex() method to modify the row or column labels.
- Below code will add column E to the data frame

  ```
  pdf1 = pdf.reindex(index=dates[0:4], columns=list(pdf.columns) + ['E'])
  ```

- **`DataFrame.dropna(axis=0, how='any', thresh=None, subset=None, inplace=False)`**
```
Key Parameters:
axis:
0 → drop rows (default)
1 → drop columns
how:
'any' → drop if any value is NaN (default)
'all' → drop only if all values are NaN
thresh: Keep rows/columns with at least this many non-NaN values.
subset: Specify columns to check for NaNs.
inplace: Modify the DataFrame in place without returning a new one.
```
- **`psdf1.dropna(how='any')`** :- To drop any rows that have missing data.
- **`psdf1.fillna(5')`** :- Filling missing data.
- 
