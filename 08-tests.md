# Tests

Probably if you've worked on a Windows computer, you must have used Microsoft Defender Full Scan at some point. During or at the end of the scan, some results were displayed. dbt works in almost the same way, only that this time they scan your data.

Tests in dbt are basically assertions of your data. That is, they are just some assumptions of your datasets that are correct. Tests enable one to know: 1) which assumptions are wrong about our data, and 2) which parts of our data diverge from the expected norm. Tests can sometimes feel like something to bemoan, can fail (as some do) but the overarching advantage is that they make us understand more about our data. They can also be lifesavers in that they can pinpoint a serious problem which could be harder to debug later on! In a nutshell, dbt tests perform much like a programmatic scan that will only spew out errors if something is suspicious.

## Types of tests in dbt

In dbt, a test can be defined in either of the following two ways:

1. generic test - this is a test written in a SQL model and defined inside a YAML file. The test in the YAML file is defined using jinja Macros. dbt comes in with  four, all-batteries included tests namely:

* unique - asserts that the column has no repeating values

* not_null - asserts that there are no null values

* accepted_values - checks if the values in your field correspond to those in a defined list

* relationships - checks if the field has an existing relationship with another field in a different table. 

2. singular test - some normally refer to this as a custom test. This is a SQL query which is used to check assertions in your data. The SQL files that make up your test are defined inside the `tests` directory or the path defined in the `test-paths` key inside the `dbt_project.yml` file. Each SQL file will have one test only. Nevertheless, if this test will be used across many fields and files, you can reference it using jinja macros `{{ }}`. If this is the case, it is no longer a singular test but a generic *custom* test. 

Let's start with a dbt out-of-the-box generic test.

## Generic tests in dbt

We will start with a very simple test, the `not_null` test on the `start_station_name` column of `citi_trips_long` model. Surely, unless someone is teleporting from somewhere, every rented bike must have an origin! 


```

- name: citi_trips_long
    description: '{{ doc("citi_trips_long") }}'
    columns:
      - name: starttime
        description: '{{ doc("starttime") }}'

      --snip--

      - name: start_station_name
        description: "Start Station Name"
        tests:
          - not_null
          
```


We use the below code to test only those models under `my_models` folder.

```
dbt test --select my_models
```

Here is the output.

```
--snip--

19:15:33  Concurrency: 1 threads (target='dev')
19:15:33  
19:15:33  1 of 1 START test not_null_citi_trips_long_start_station_name .................. [RUN]
19:15:36  1 of 1 PASS not_null_citi_trips_long_start_station_name ........................ [PASS in 3.20s]
19:15:36  
19:15:36  Finished running 1 test in 0 hours 0 minutes and 4.44 seconds (4.44s).
19:15:36  
19:15:36  Completed successfully
19:15:36  
19:15:36  Done. PASS=1 WARN=0 ERROR=0 SKIP=0 TOTAL=1

```

If we had gone with the more blanket code of `dbt test`, it would have also tested those models under the shipped `examples` folder where there is already a test designed to fail, purely for demonstration purposes. Back to our `citi_trips_long` model, we can see that the test passed successfully. As expected, there are no null values in our `start_station_name` field. If the opposite happened, then there must be a row with a null value under this field.

Now let's create a test that will fail on purpose. From a large dataset, obviously the station names can't be unique all through. So we insert the `unique` value under the `tests` key as follows:

```
- name: start_station_name
        description: "Start Station Name"
        tests:
          - not_null
          - unique
```

The output is as below. Note that it results in a failure and also shows the path to the SQL query inside the `target/` directory that was used to carry out the test. If you paste this SQL query from the `target` directory into the SQL tab of the BigQuery data warehouse, it will return the number of rows that failed the test.

```
19:23:35  Concurrency: 1 threads (target='dev')
19:23:35  
19:23:35  1 of 2 START test not_null_citi_trips_long_start_station_name .................. [RUN]
19:23:38  1 of 2 PASS not_null_citi_trips_long_start_station_name ........................ [PASS in 3.01s]
19:23:38  2 of 2 START test unique_citi_trips_long_start_station_name .................... [RUN]
19:23:41  2 of 2 FAIL 910 unique_citi_trips_long_start_station_name ...................... [FAIL 910 in 2.68s]
19:23:41  
19:23:41  Finished running 2 data tests in 0 hours 0 minutes and 6.78 seconds (6.78s).
19:23:41  
19:23:41  Completed with 1 error and 0 warnings:
19:23:41  
19:23:41  Failure in test unique_citi_trips_long_start_station_name (models/my_models/my_models.yml)
19:23:41    Got 910 results, configured to fail if != 0
19:23:41  
19:23:41    compiled code at target/compiled/dbt_book/models/my_models/my_models.yml/unique_citi_trips_long_start_station_name.sql
19:23:41  
19:23:41  Done. PASS=1 WARN=0 ERROR=1 SKIP=0 TOTAL=2

```

![Failed test](./images/failed_test.png)

## Singular tests in dbt

As mentioned earlier, singular tests are SQL models in your `tests` folder. They differ from the generic tests in that they are bespoke to your data needs. For example, if one of your tests doesn't conform to the four tests shipped with dbt, you can create one inside the `tests` folder. 

Below we create a singular test called `unnecessary_trips.sql` inside the `test` folder. The purpose of the test is to raise an error if a trip is less than 10 min. The latter must surely generate an error! Notice that one can reference another model within a test using the `ref()` function.

```
SELECT bikeid, start_station_name, end_station_name,
    birth_year, gender, tripduration
FROM {{ ref('citi_trips_round') }}
WHERE trip_min_round < 10

```

So to perform the litmus test, run the usual magic phrase:

```
dbt test --select unnecessary_trips

```

Below will be the output.

```
19:51:50  Concurrency: 1 threads (target='dev')
19:51:50  
19:51:50  1 of 1 START test unnecessary_trips ............................................ [RUN]
19:51:53  1 of 1 FAIL 25183763 unnecessary_trips ......................................... [FAIL 25183763 in 3.11s]
19:51:53  
19:51:53  Finished running 1 test in 0 hours 0 minutes and 4.70 seconds (4.70s).
19:51:53  
19:51:53  Completed with 1 error and 0 warnings:
19:51:53  
19:51:53  Failure in test unnecessary_trips (tests/unnecessary_trips.sql)
19:51:53    Got 25183763 results, configured to fail if != 0
19:51:53  
19:51:53    compiled code at target/compiled/dbt_book/tests/unnecessary_trips.sql
19:51:53  
19:51:53  Done. PASS=0 WARN=0 ERROR=1 SKIP=0 TOTAL=1
```

As you can see, a total 25183763 rows failed the test. 

A singular test can also be transformed into a generic test when its reused across the fields of your table(s) in the data warehouse. To demonstrate this, let's create some generic tests.

## Creating a generic test

*Custom* generic tests are created within a `generic` folder within the `tests` directory. Thus, within your `tests/generic` directory, you will place the SQL models for your tests. Anything returned by your SQL models is in fact your tests failing! If nothing is returned, then your test passed!

Create a SQL model called `long_characters` within the `tests/generic` directory.

```
{% test long_characters (model, column_name) %}
SELECT * FROM {{ model }}
WHERE LENGTH({{ column_name }}) > 15 
{% endtest %}
```

Let's go through the above query line by line. We begin a generic test by using the `{%test <model-name> (model, column_name) %}` *SQL statement in here* `{% endtest %}` tags. The `model` and `column_name` are standard arguments where one or both should be defined. 

* `model` - the resource on which the test will be operated on. In our case, this is any model which the test will run on.

* `column_name` - this is a field within which the model will be run against. In other words, the column at which this model will operate on.

The SQL statement within the test tags references the model and the columns using the `{{ model }}` and `{{ column_name }}` respectively. For example, if the test is placed in the `my_models` YAML file, under the `citi_trips_long` model name, for the field `start_station_name`, it is as though the test is running this SELECT statement:

```
SELECT * FROM citi_trips_long
WHERE LENGTH(start_station_name) > 15 

```

In very few words, the above SQL tells dbt to shout out an error if any station name is greater than 15 characters.

Let's create another test that will raise an alarm if there are special characters within your `start_station_name`. 

```
{% test special_characters (model, column_name) %}
SELECT * FROM {{ model }}
WHERE {{ column_name }}
LIKE '%^[a-zA-Z0-9+-]%'
{% endtest %}

```

Okay, it's now time for our litmus test. Going back to the `start_station_name` mapping where we already performed some litmus tests, lets also add our two *custom* generic tests of `long_character` and `special_characters`. 


```
- name: citi_trips_long
    description: '{{ doc("citi_trips_long") }}'
    columns:
      --snip--

      - name: start_station_name
        description: "Start Station Name"
        tests:
          - not_null
          - unique
          - long_characters
          - special_characters

```

If we go for the big bull and run all our tests with the trusty `dbt test` keyword, we can see the output of the nine tests we have so far. 

```
19:08:15  Concurrency: 1 threads (target='dev')
19:08:15  
19:08:15  1 of 9 START test long_characters_citi_trips_long_start_station_name ........... [RUN]
19:08:29  1 of 9 FAIL 11463868 long_characters_citi_trips_long_start_station_name ........ [FAIL 11463868 in 14.47s]

--snip--

19:08:55  5 of 9 START test special_characters_citi_trips_long_start_station_name ........ [RUN]
19:08:59  5 of 9 PASS special_characters_citi_trips_long_start_station_name .............. [PASS in 4.48s]

--snip--

19:09:07  9 of 9 START test unnecessary_trips ............................................ [RUN]
19:09:10  9 of 9 FAIL 25183763 unnecessary_trips ......................................... [FAIL 25183763 in 2.81s]

```

From the above output, we can see that there were some station names with quite some long names and (thankfully) no station names with special characters.

There is also another output following the above which shows how many records failed, depending on your test.

```
19:09:10  Completed with 4 errors and 0 warnings:
19:09:10  
19:09:10  Failure in test long_characters_citi_trips_long_start_station_name (models/my_models/my_models.yml)
19:09:10    Got 11463868 results, configured to fail if != 0
 
--snip--

19:09:10  Failure in test unique_citi_trips_long_start_station_name (models/my_models/my_models.yml)
19:09:10    Got 910 results, configured to fail if != 0

```

## Configuring custom generic tests

What if you feel that the *FAIL* alert like in the above tests is shouting too much?! That you would prefer them to be *WARNINGS* because they are non-critical?

You can do so by setting the configuration to display as a warning rather than as an error.

```
{% test long_characters (model, column_name) %}
    {{ config(severity = 'warn') }}
    SELECT * FROM {{ model }}
    WHERE LENGTH({{ column_name }}) > 15 
{% endtest %}
```

Here is the output as a warning.

```

19:16:34  1 of 9 START test long_characters_citi_trips_long_start_station_name ........... [RUN]
19:16:36  1 of 9 WARN 11463868 long_characters_citi_trips_long_start_station_name ........ [WARN 11463868 in 2.64s]
19:16:36  2 of 9 START test not_null_citi_trips_long_start_station_name .................. [RUN]
19:16:39  2 of 9 PASS not_null_citi_trips_long_start_station_name ........................ [PASS in 2.67s]
-- snip --
19:16:56  Warning in test long_characters_citi_trips_long_start_station_name (models/my_models/my_models.yml)
19:16:56  Got 11463868 results, configured to warn if != 0
-- snip --
```

However, in case you immediately change your mind that the failing tests of `long_characters` should be an error rather than a warning, you can override your custom generic SQL models by specifying the severity within the YAML definition files.

```
- name: start_station_name
        description: "Start Station Name"
        tests:
          - not_null
          - unique
          - long_characters:
            severity: 'error'
          - special_characters
```

In the generated output, it will be back to business as usual with the 'FAIL' keyword.

```
-- snip -- 
19:21:25  1 of 9 START test long_characters_citi_trips_long_start_station_name ........... [RUN]
19:21:28  1 of 9 FAIL 11463868 long_characters_citi_trips_long_start_station_name ........ [FAIL 11463868 in 3.45s]
-- snip --

19:21:50  Failure in test long_characters_citi_trips_long_start_station_name (models/my_models/my_models.yml)
19:21:50    Got 11463868 results, configured to fail if != 0
-- snip --

```

## Storing test failures

If you thought that dbt tests are a cool feature, then there is one more trick in the bag if you want a neat one-liner to view a dataset of the failing records. The *open sesame* key word is `--store-failures`. dbt will store the failing records as a table in the database. 

Let's try it out. 

```
dbt test --store-failures

```

Of course our test will generate failed records, but we've already seen them. Here is part of the output for the test of long characters above the 15 character threshold. 

```
19:26:40  Failure in test long_characters_citi_trips_long_start_station_name (models/my_models/my_models.yml)
19:26:40    Got 11463868 results, configured to fail if != 0
19:26:40  
19:26:40    compiled code at target/compiled/dbt_book/models/my_models/my_models.yml/long_characters_citi_trips_long_start_station_name.sql
19:26:40  
19:26:40    See test failures:
  -------------------------------------------------------------------------------------------------------------------
  select * from `dbt-project-437116`.`nyc_bikes_dbt_test__audit`.`long_characters_citi_trips_long_start_station_name`
  -------------------------------------------------------------------------------------------------------------------
```

If you paste the `SELECT` statement in one of the SQL tabs for BigQuery, it will not only return the number of failing records but also the data that is part of the failing records.

![Failing records](./images/failing_records.png)


That's a very convenient one-liner!


Below are other forms of generic tests shipped with dbt, for the `accepted_values` and `relationships`. The latter took too long to run and was cancelled midway.

```
- name: end_station_name
        description: "End Station Name"
         tests:
            - relationships:
                to: ref('citi_trips_round')
                field: end_station_name

      - name: gender
        description: "Gender (unknown, male, female)"
        tests:
          - accepted_values:
              values: ['unknown', 'male', 'female']

```

It's been quite an exciting journey with tests. And for sure dbt tests do grill your data!








