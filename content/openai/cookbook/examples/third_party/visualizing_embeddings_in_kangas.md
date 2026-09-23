> For the complete documentation index, see [llms.txt](/llms.txt). Markdown versions of documentation pages are available by appending `.md` to the page URL.

## Visualizing the embeddings in Kangas

In this Jupyter Notebook, we construct a Kangas DataGrid containing the data and projections of the embeddings into 2 dimensions.

## What is Kangas?

[Kangas](https://github.com/comet-ml/kangas/) as an open source, mixed-media, dataframe-like tool for data scientists. It was developed by [Comet](https://comet.com/), a company designed to help reduce the friction of moving models into production. 

### 1. Setup

To get started, we pip install kangas, and import it.

```python
%pip install kangas --quiet
```

```python
import kangas as kg
```

### 2. Constructing a Kangas DataGrid

We create a Kangas Datagrid with the original data and the embeddings. The data is composed of a rows of reviews, and the embeddings are composed of 1536 floating-point values. In this example, we get the data directly from github, in case you aren't running this notebook inside OpenAI's repo.

We use Kangas to read the CSV file into a DataGrid for further processing.

```python
data = kg.read_csv("https://raw.githubusercontent.com/openai/openai-cookbook/main/examples/data/fine_food_reviews_with_embeddings_1k.csv")
```

```text
Loading CSV file 'fine_food_reviews_with_embeddings_1k.csv'...
```

```text
1001it [00:00, 2412.90it/s]
100%|████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1000/1000 [00:00<00:00, 2899.16it/s]
```

We can review the fields of the CSV file:

```python
data.info()
```

```text
DataGrid (in memory)
    Name   : fine_food_reviews_with_embeddings_1k
    Rows   : 1,000
    Columns: 9
#   Column                Non-Null Count DataGrid Type       
--- -------------------- --------------- --------------------
1   Column 1                       1,000 INTEGER             
2   ProductId                      1,000 TEXT                
3   UserId                         1,000 TEXT                
4   Score                          1,000 INTEGER             
5   Summary                        1,000 TEXT                
6   Text                           1,000 TEXT                
7   combined                       1,000 TEXT                
8   n_tokens                       1,000 INTEGER             
9   embedding                      1,000 TEXT
```

And get a glimpse of the first and last rows:

```python
data
```

<table><tbody><tr><th colspan="1">          row-id </th> <th colspan="1">        Column 1 </th> <th colspan="1">       ProductId </th> <th colspan="1">          UserId </th> <th colspan="1">           Score </th> <th colspan="1">         Summary </th> <th colspan="1">            Text </th> <th colspan="1">        combined </th> <th colspan="1">        n_tokens </th> <th colspan="1">       embedding </th> </tr><tr>
<td colspan="1">               1 </td> <td colspan="1">               0 </td> <td colspan="1">      B003XPF9BO </td> <td colspan="1">  A3R7JR3FMEBXQB </td> <td colspan="1">               5 </td> <td colspan="1"> where does one  </td> <td colspan="1"> Wanted to save  </td> <td colspan="1"> Title: where do </td> <td colspan="1">              52 </td> <td colspan="1"> [0.007018072064 </td> </tr><tr>
<td colspan="1">               2 </td> <td colspan="1">             297 </td> <td colspan="1">      B003VXHGPK </td> <td colspan="1">  A21VWSCGW7UUAR </td> <td colspan="1">               4 </td> <td colspan="1"> Good, but not W </td> <td colspan="1"> Honestly, I hav </td> <td colspan="1"> Title: Good, bu </td> <td colspan="1">             178 </td> <td colspan="1"> [-0.00314055196 </td> </tr><tr>
<td colspan="1">               3 </td> <td colspan="1">             296 </td> <td colspan="1">      B008JKTTUA </td> <td colspan="1">  A34XBAIFT02B60 </td> <td colspan="1">               1 </td> <td colspan="1"> Should advertis </td> <td colspan="1"> First, these sh </td> <td colspan="1"> Title: Should a </td> <td colspan="1">              78 </td> <td colspan="1"> [-0.01757248118 </td> </tr><tr>
<td colspan="1">               4 </td> <td colspan="1">             295 </td> <td colspan="1">      B000LKTTTW </td> <td colspan="1">  A14MQ40CCU8B13 </td> <td colspan="1">               5 </td> <td colspan="1"> Best tomato sou </td> <td colspan="1"> I have a hard t </td> <td colspan="1"> Title: Best tom </td> <td colspan="1">             111 </td> <td colspan="1"> [-0.00139322795 </td> </tr><tr>
<td colspan="1">               5 </td> <td colspan="1">             294 </td> <td colspan="1">      B001D09KAM </td> <td colspan="1">  A34XBAIFT02B60 </td> <td colspan="1">               1 </td> <td colspan="1"> Should advertis </td> <td colspan="1"> First, these sh </td> <td colspan="1"> Title: Should a </td> <td colspan="1">              78 </td> <td colspan="1"> [-0.01757248118 </td> </tr><tr>
</tr><tr><td colspan="10" style="text-align: left;">...</td></tr><tr><td colspan="1">             996 </td> <td colspan="1">             623 </td> <td colspan="1">      B0000CFXYA </td> <td colspan="1">  A3GS4GWPIBV0NT </td> <td colspan="1">               1 </td> <td colspan="1"> Strange inflamm </td> <td colspan="1"> Truthfully wasn </td> <td colspan="1"> Title: Strange  </td> <td colspan="1">             110 </td> <td colspan="1"> [0.000110913533 </td> </tr><tr>
<td colspan="1">             997 </td> <td colspan="1">             624 </td> <td colspan="1">      B0001BH5YM </td> <td colspan="1">   A1BZ3HMAKK0NC </td> <td colspan="1">               5 </td> <td colspan="1"> My favorite and </td> <td colspan="1"> You've just got </td> <td colspan="1"> Title: My favor </td> <td colspan="1">              80 </td> <td colspan="1"> [-0.02086931467 </td> </tr><tr>
<td colspan="1">             998 </td> <td colspan="1">             625 </td> <td colspan="1">      B0009ET7TC </td> <td colspan="1">  A2FSDQY5AI6TNX </td> <td colspan="1">               5 </td> <td colspan="1"> My furbabies LO </td> <td colspan="1"> Shake the conta </td> <td colspan="1"> Title: My furba </td> <td colspan="1">              47 </td> <td colspan="1"> [-0.00974910240 </td> </tr><tr>
<td colspan="1">             999 </td> <td colspan="1">             619 </td> <td colspan="1">      B007PA32L2 </td> <td colspan="1">  A15FF2P7RPKH6G </td> <td colspan="1">               5 </td> <td colspan="1"> got this for th </td> <td colspan="1"> all i have hear </td> <td colspan="1"> Title: got this </td> <td colspan="1">              50 </td> <td colspan="1"> [-0.00521062919 </td> </tr><tr>
<td colspan="1">            1000 </td> <td colspan="1">             999 </td> <td colspan="1">      B001EQ5GEO </td> <td colspan="1">  A3VYU0VO6DYV6I </td> <td colspan="1">               5 </td> <td colspan="1"> I love Maui Cof </td> <td colspan="1"> My first experi </td> <td colspan="1"> Title: I love M </td> <td colspan="1">             118 </td> <td colspan="1"> [-0.00605782261 </td> </tr><tr>
</tr><tr>
<td colspan="10" style="text-align: left;"> [1000 rows x 9 columns] </td> </tr><tr>
</tr><tr><td colspan="10" style="text-align: left;"></td></tr><tr><td colspan="10" style="text-align: left;">*  Use DataGrid.save() to save to disk</td></tr><tr><td colspan="10" style="text-align: left;">** Use DataGrid.show() to start user interface</td></tr></tbody></table>

Now, we create a new DataGrid, converting the numbers into an Embedding:

```python
import ast # to convert string of a list of numbers into a list of numbers

dg = kg.DataGrid(
    name="openai_embeddings",
    columns=data.get_columns(),
    converters={"Score": str},
)
for row in data:
    embedding = ast.literal_eval(row[8])
    row[8] = kg.Embedding(
        embedding, 
        name=str(row[3]), 
        text="%s - %.10s" % (row[3], row[4]),
        projection="umap",
    )
    dg.append(row)
```

The new DataGrid now has an Embedding column with proper datatype.

```python
dg.info()
```

```text
DataGrid (in memory)
    Name   : openai_embeddings
    Rows   : 1,000
    Columns: 9
#   Column                Non-Null Count DataGrid Type       
--- -------------------- --------------- --------------------
1   Column 1                       1,000 INTEGER             
2   ProductId                      1,000 TEXT                
3   UserId                         1,000 TEXT                
4   Score                          1,000 TEXT                
5   Summary                        1,000 TEXT                
6   Text                           1,000 TEXT                
7   combined                       1,000 TEXT                
8   n_tokens                       1,000 INTEGER             
9   embedding                      1,000 EMBEDDING-ASSET
```

We simply save the datagrid, and we're done.

```python
dg.save()
```

### 3. Render 2D Projections

To render the data directly in the notebook, simply show it. Note that each row contains an embedding projection. 

Scroll to far right to see embeddings projection per row.

The color of the point in projection space represents the Score.

```python
dg.show()
```

<p>Run this notebook locally to view this interactive output.</p>

Group by "Score" to see rows of each group.

```python
dg.show(group="Score", sort="Score", rows=5, select="Score,embedding")
```

<p>Run this notebook locally to view this interactive output.</p>

An example of this datagrid is hosted here: https://kangas.comet.com/?datagrid=/data/openai_embeddings.datagrid