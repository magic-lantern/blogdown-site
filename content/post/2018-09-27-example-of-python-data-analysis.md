---
title: Example of Python data analysis
author: Seth Russell
date: '2018-09-27'
categories:
  - Python
tags:
  - bigquery
  - data analysis
  - google cloud
  - python
  - blog
slug: example-of-python-data-analysis
description: A demonstration of a few of the things you can do to analyze data stored
  in Google BigQuery using Python.
---

_Work in Progress..._


This Analytics example is meant to demonstrate a few of the things you can do to analyze data stored in Google BigQuery using Python. Originally developed some time ago, I thought I'd clean it up and post it here.

For those that aren't familiar with BigQuery, it is a 'serverless' database system that is fully managed and always available. Instead of charging per environment/instance/hour of time like many cloud database systems, Google charges based on the amount of data a query processes. For those familiar with Amazon Web Services, BigQuery is more like DynamoDB rather than an Aurora/Redshift/etc instances where you pay per hour.

For more details on this example, setup, etc. please see the project wiki: https://github.com/CUD2V/analytics_examples/wiki

For some additional examples showing more of the capabilities of Google BigQuery or Google Cloud Storage see:
* [Google Cloud BigQuery Example](https://gist.github.com/magic-lantern/904e22ca625404da489dab4f2706fdc7) - Some alternative methods of getting data from Google BigQuery.
* [Google Cloud Storage Example](https://gist.github.com/magic-lantern/c11500847f06a6e63bae4ca010595773) - Most analysis that a person would want to do will likely include accessing external data or storing results of some process for further downstream analysis. Google Cloud Storage is one option for that phase of an anlysis pipeline.

```python
import pandas

from IPython.core.display import display, HTML
display(HTML("<style>.container { width:95% !important; }</style>"))

import psycopg2 as pg
import pandas.io.sql as psql

from bokeh.io import output_notebook, show
output_notebook()
```

```python
# this is to hide useless errors if using OAuth with BigQuery
import logging
logging.getLogger('googleapiclient.discovery_cache').setLevel(logging.CRITICAL)
# don't want to be messaged about future warnings as I'm not explicitly calling code that is being warned about
import warnings
warnings.simplefilter(action='ignore', category=FutureWarning)

# set this to either postgres or bigquery ####
datasource = 'bigquery'
##############################################

if datasource == 'postgres':
    # get connected to the database
    connection = pg.connect("host=localhost dbname=ohdsi user=ohdsi password=ohdsi")

    # print the connection string we will use to connect
    print("Connecting to database: ", connection)

    # conn.cursor will return a cursor object, you can use this cursor to perform queries
    cursor = connection.cursor()
    print("Connected to Postgres database!\n")
elif datasource == 'bigquery':
    connection = {
        'project_id' : 'synpuf-omop-project',
        'dialect'    : 'standard'
    }
    print("Setup Google BigQuery connection")
else:
    connection = None

def read_data(sql):
    if datasource == 'postgres':
        return pandas.read_sql(sql, connection)
    elif datasource == 'bigquery':
        return pandas.read_gbq(sql, **connection)
    else:
        return pandas.DataFrame()
```

## Exploration and Visualization

_These next cells are charts looking at births by year - of the population still active in medicare today._

```python
from bokeh.charts import Bar
from bokeh.io import output_notebook, show
from bokeh.charts import defaults
defaults.width = 900
defaults.height = 700

age_df = read_data('''
select
    count(year_of_birth) count,
    year_of_birth,
    c1.concept_name gender
from synpuf_omop.person p
left join synpuf_omop.concept c1 on p.gender_concept_id = c1.concept_id
group by gender, year_of_birth
order by year_of_birth, gender
''')

p = Bar(age_df,             # source of data
        'year_of_birth',    # columns from dataframe to use
        #label='origin', 
        agg='sum',
        values='count',
        stack='gender',
        title="Births by year, stacked by gender",
        legend='top_right')
show(p)
```

```python
from bokeh.charts import Bar
from bokeh.io import output_notebook, show
from bokeh.charts import defaults
defaults.width = 900
defaults.height = 700

pct_df = read_data(
'''
select
    year_of_birth,
    count(case when c1.concept_name = 'FEMALE' then 1 end) gender_count,
    'FEMALE' gender,
    count(1) total_births
from synpuf_omop.person p
left join synpuf_omop.concept c1 on p.gender_concept_id = c1.concept_id
group by year_of_birth
union all
select
    year_of_birth,
    count(case when c1.concept_name = 'MALE' then 1 end) gender_count,
    'MALE' gender,
    count(1) total_births
from synpuf_omop.person p
left join synpuf_omop.concept c1 on p.gender_concept_id = c1.concept_id
group by year_of_birth
order by year_of_birth, gender asc
''')

def f(i):
    return float(i['gender_count']) / float(i['total_births'])
pct_df['pct'] = pct_df.apply(f, axis=1)


p = Bar(pct_df,             # source of data
        values='pct',          # y axis
        label='year_of_birth', # x axis 
        agg='sum',
        stack='gender',
        title="Percentage of births by year, stacked by gender",
        legend='top_right')
show(p)
```

_These next few cells look at drug duration (how long a perscription is to last)_

Due to differences in SQL dialects, this is the PostgreSQL version - inline below is the Google BigQuery version. Might be possible to make one statement work for both...

```sql
select
    --person_id,
    --drug_concept_id,
    c1.concept_name drug_name,
    --drug_era_start_date,
    --drug_era_end_date,
    drug_era_end_date - drug_era_start_date duration
from synpuf_omop.drug_era d
left join synpuf_omop.concept c1 on d.drug_concept_id = c1.concept_id
where c1.concept_name in (
select drug_name from (
    select
    c1.concept_name drug_name,
    count(1) count
    from synpuf_omop.drug_era d
    left join synpuf_omop.concept c1 on d.drug_concept_id = c1.concept_id
    group by drug_name
    order by count desc
    limit 25
   ) x
)
order by drug_name
```

```python
# perhaps modify this query to look at drugs with most variation in duration?

from bokeh.charts import BoxPlot, output_file, show
from bokeh.sampledata.autompg import autompg as df
from bokeh.charts import defaults
defaults.width = 900
defaults.height = 900

dd_df = read_data(
'''
select
    --person_id,
    --drug_concept_id,
    c1.concept_name drug_name,
    --drug_era_start_date,
    --drug_era_end_date,
    date_diff(cast(drug_era_end_date as date), cast(drug_era_start_date as date), day) as duration
from synpuf_omop.drug_era d
left join synpuf_omop.concept c1 on d.drug_concept_id = c1.concept_id
where c1.concept_name in (
select drug_name from (
    select
    c1.concept_name drug_name,
    count(1) count
    from synpuf_omop.drug_era d
    left join synpuf_omop.concept c1 on d.drug_concept_id = c1.concept_id
    group by drug_name
    order by count desc
    limit 25
   ) x
)
order by drug_name
''')

p = BoxPlot(dd_df,
            values='duration',      # y axis
            label='drug_name',      # x axis
            title="Drug Duration Box Plot",
            legend=False,
           )
p.xaxis.axis_label = "Drug"
p.yaxis.axis_label = "Duration (days)"


show(p)
```

_Alternatively, you can just get all the data via more simple SQL SELECT statment and do the data processing via Pandas_

Again as with the previous query, due to SQL dialect differences, this is the PostgreSQL version - in cell below is the BigQuery version:

```sql
select
    c1.concept_name drug_name,
    drug_era_end_date - drug_era_start_date duration
from synpuf_omop.drug_era d
left join synpuf_omop.concept c1 on d.drug_concept_id = c1.concept_id
order by drug_name
```

```python
size_df = None
drug_df = None
top_25 = None
top_drugs = None

drug_df = read_data(
'''
select
    c1.concept_name drug_name,
    date_diff(cast(drug_era_end_date as date), cast(drug_era_start_date as date), day) as duration
from synpuf_omop.drug_era d
left join synpuf_omop.concept c1 on d.drug_concept_id = c1.concept_id
order by drug_name
''')

# if we only want to look at 25 most common drugs
# count rows grouping by drug_name
size_df = drug_df.groupby("drug_name").size()
# sort the counted result and only return top 25
top_25 = size_df.sort_values(ascending = False).head(25)
# for verification purposes show all rows from original dataset matching most common drug
#drug_df[drug_df.drug_name.str.contains(top_25.index[0]) == True]
# only keep rows that match the top_25 pandas series (single column of a dataframe)
top_drugs = drug_df[drug_df['drug_name'].isin(top_25.index)]
# for verification sql method says there are 1683795 rows
print("Does Pandas version match SQL results:", top_drugs.shape[0] == 1683795)
```


```python
from bokeh.charts import BoxPlot, output_file, show
from bokeh.sampledata.autompg import autompg as df
from bokeh.charts import defaults
defaults.width = 900
defaults.height = 900
p = BoxPlot(top_drugs,
            values='duration',      # y axis
            label='drug_name',      # x axis
            title="Drug Duration Box Plot",
            legend=False,
           )
p.xaxis.axis_label = "Drug"
p.yaxis.axis_label = "Duration (days)"


show(p)
```

## Cohort identification & Prediction

Suppose we want to do some analysis including prediction for patients that complain of lower back pain (SNOMED code 279039007 - see http://bioportal.bioontology.org/ontologies/SNOMEDCT?p=classes&conceptid=279039007 or https://phinvads.cdc.gov/vads/http:/phinvads.cdc.gov/vads/ViewCodeSystemConcept.action?oid=2.16.840.1.113883.6.96&code=279039007 for more information)

The OMOP data model has [CONDITION_OCCURRENCE](http://www.ohdsi.org/web/wiki/doku.php?id=documentation:cdm:condition_occurrence) table to document findings. The [CONDITION_ERA](http://www.ohdsi.org/web/wiki/doku.php?id=documentation:cdm:condition_era) table is a calculation of a condition duration.

While there are many elements that we could use for predicting condition duration, suppose we start with basic demographic information about the patient. Here's a query that creates a pandas dataframe with our desired cohort.

```python
backpain_df = read_data(
'''
SELECT
  c.person_id,
  gender_concept_id,
  year_of_birth,
  race_concept_id,
  ethnicity_concept_id,
  location_id,
  DATE_DIFF(CAST(condition_era_end_date AS date), CAST(condition_era_start_date AS date), day) AS duration
FROM
  synpuf_omop.condition_era c
LEFT JOIN
  synpuf_omop.person p
ON
  c.person_id = p.person_id
WHERE
  condition_concept_id = 194133
''')
```

```python
# View summary information about dataset
print(backpain_df.describe().to_string())

# prevent wrapping when printing the full dataframe
pandas.set_option('display.expand_frame_repr', False)
print(backpain_df)
```

```python
bf = backpain_df.copy()
```

```python
# Most of the data in this dataset is categorical, so need to use dummy encoding on each categorical column
# so that regression will work correctly. Here's the categorical columns:
#   gender_concept_id     (binary)
#   race_concept_id       (multiple categories)
#   ethnicity_concept_id  (binary)
#   location_id           (multiple categories)
#
# can't do them all at once - so step through one at a time
bf = pandas.get_dummies(bf, columns=['gender_concept_id'], drop_first=True)
bf = pandas.get_dummies(bf, columns=['race_concept_id'], drop_first=True)
bf = pandas.get_dummies(bf, columns=['ethnicity_concept_id'], drop_first=True)
bf = pandas.get_dummies(bf, columns=['location_id'], drop_first=True)
print(bf.head())
```

```python
# only run this cell if previous results look correct
backpain_df = bf.copy()
```

```python
print(backpain_df.head())
```

```python
# divide data into independent and dependent variables

# exclude person_id and duration from independent variables
input_df = backpain_df.drop(['person_id', 'duration'], axis=1)

output_df = backpain_df['duration']

# split data into training vs testing dataset
from sklearn.model_selection import train_test_split

input_train, input_test, output_train, output_test = train_test_split(input_df, output_df, test_size = 0.2, random_state = 0)
```

```python
# now generate linear regression model on training data
from sklearn.linear_model import LinearRegression
lm = LinearRegression()
lm.fit(input_train, output_train)
```

```python
# print intercept and coefficients
print("Intercept: ", lm.intercept_)
print("Coefficients: ", lm.coef_)
print("R^2 value: ", lm.score(input_train, output_train))
```


For a brief explaination of what R<sup>2</sup> means, see http://blog.minitab.com/blog/adventures-in-statistics-2/regression-analysis-how-do-i-interpret-r-squared-and-assess-the-goodness-of-fit

In general, the closer R<sup>2</sup> is to 1, the better the model explains variation in data. Plotting or visualizing data along with the predictive model helps to visually understand how close they are to each other.

```python
# now evaluate model on test data

# now predict answers (regression) - since we only care about whole days, round all output to whole numbers
output_pred = pandas.Series(data=lm.predict(input_test))
output_pred = output_pred.round()

for index, value in output_pred.iteritems():
    print('Real value: ', output_test.values[index], 'Predicted value: ', value)
    if index >= 10:
        break
        
# should next calculate various measures such as precision, recall, etc. 
# See https://stackoverflow.com/questions/31421413/how-to-compute-precision-recall-accuracy-and-f1-score-for-the-multiclass-case
# for some examples.
```

## Final notes

In order to decide which variables are meaningful in the model, methods such as back-propigation, forward-propigation or similar methods should be used.

Additionally, other regression methods may work better for predicting duration - in fact the data may not even be suited for Linear Regression. In order for Linear Regression to work, certain assumptions about the data must be true. For more information see: http://pareonline.net/getvn.asp?n=2&v=8

Also, since this data set has other elements, the inclusion of other factors may help in predicting condition duration.