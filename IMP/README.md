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
