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
- **`DataFrame.collect()`** :- collects the distributed data to the driver side as the local data in Python. Note that this can throw an out-of-memory error when the dataset is too large to fit in the driver side because it collects all the data from executors to the driver side.
- **`DataFrame.take()/DataFrame.tail()`** :- In order to avoid throwing an out-of-memory exception, use DataFrame.take() or DataFrame.tail().
