# Sources 

In the all-time favourite book *The voyages and adventures of Captain Hatteras*, every morning, the second-in-command, a well versed captain of the high seas, Richard Shandon by name, would receive a letter from an anonymous sender directing him on which direction to steer the ship. In dbt, sources are what make your data in the data warehouse be referenced in dbt operations such as running models, tests and checking the 'freshness' of your data. 

Just like in the above anecdote where, if any person in the crew would ask Sir Richard Shandon for justification of any task they were commanded to do, Richard would always refer to the authoritative letter from an anonymous source. Likewise, when working with sources, dbt will perform operations by referencing the sources using the `source` function (`{{ source("schema", "table") }}`).

Sources in dbt are defined inside YAML files, and they are referenced inside SQL files, just like regular models again!

## Defining a source

To demonstrate defining sources, we will work with two tables already in our data warehouse. These are the `2014-tripdata` and `citi_trips_round` tables under `nyc_bikes_nyc_bikes2014` (`nyc_bikes_nyc_bikes2014/2014-tripdata`) and `nyc_bikes` tree structures in BigQuery respectively. 

Create a new sibling directory called `sources` next to the `docs`, `example`, and `my_models` folders. Inside it, create a new YAML file called `sources_bikes.yml`. The path to this file should be `models/sources/sources_bikes.yml`. 

Copy paste these contents into the newly created YAML file.

```
version: 2

sources:
  - name: nyc_bikes_nyc_bikes2014
    schema: nyc_bikes_nyc_bikes2014
    tables:
      - name: 2014-tripdata

  - name: nyc_bikes
    schema: nyc_bikes 
    tables:
      - name: citi_trips_round

```

The above should be all too familiar since we've worked with several YAML files so far. Nevertheless, the `name` and `schema` values refer to the schema names in your data warehouse. For the `tables` dictionary, we refer to the names of those tables under a particular schema. For example, the `citi_trips_round` is definitely under the `nyc_bikes` schema. 


## Referencing sources 

Sources in our data warehouse are referenced using the `source()` function. Remember when referencing other models within models we used the `ref()` function? When working with sources, the `ref()` is now `source()`. 

Below is a demonstration of referencing a source to only select male bike riders. Notice the arrangement of the schema and table names within quotation marks (`''`) and separated by a comma. This is how we reference other data acting as the *source* in our `nyc_bikes_male.sql`. 

```
SELECT * FROM 
{{ source('nyc_bikes', 'citi_trips_round') }}
WHERE gender = "male"

```

The same also works for data uploaded as a seed in our data warehouse as seen in the `nyc_male_2014.sql`. 

```
SELECT * FROM 
{{ source('nyc_bikes_nyc_bikes2014', '2014-tripdata') }}
WHERE gender = 1

```

We run these two specific models using `dbt run --select sources` and we get this output: 

```
19:29:52  Concurrency: 1 threads (target='dev')
19:29:52  
19:29:52  1 of 2 START sql view model nyc_bikes.nyc_bikes_male ........................... [RUN]
19:29:56  1 of 2 OK created sql view model nyc_bikes.nyc_bikes_male ...................... [CREATE VIEW (0 processed) in 3.74s]
19:29:56  2 of 2 START sql view model nyc_bikes.nyc_male_2014 ............................ [RUN]
19:29:59  2 of 2 OK created sql view model nyc_bikes.nyc_male_2014 ....................... [CREATE VIEW (0 processed) in 3.29s]
19:29:59  
19:29:59  Finished running 2 view models in 0 hours 0 minutes and 12.75 seconds (12.75s).
19:30:00  
19:30:00  Completed successfully
19:30:00  
19:30:00  Done. PASS=2 WARN=0 ERROR=0 SKIP=0 TOTAL=2

```


If you check under the `nyc_bikes` schema in your BigQuery, you will notice two new views have been created: `nyc_bikes_male` and `nyc_male_2014`. 

![Sources](./images/sources.png)

One would have expected the `nyc_male_2014` view to be under the `nyc_bikes_nyc_bikes2014` schema because that's the seed dataset. Our assumption is that we set the `nyc_bikes` as the dataset to work with when setting up dbt, and thus it's very hard to deviate from this. But we stand to be corrected. One more thing, the dbt source can also work inside a `WITH` SQL statement like so in the `nyc_female_2014` model.

```
WITH nyc_female_2014 AS (
    SELECT * FROM 
        {{ source('nyc_bikes_nyc_bikes2014', '2014-tripdata') }}
    WHERE gender = 2
)

SELECT * FROM nyc_female_2014
```

## Defining properties in a `sources` file

Just like you would craft the properties for a given models' YAML file, the same can likewise be done for the sources YAML file. You can define descriptions and tests for your fields in a `sources` file. Again, what's good for the goose is good for the gander. Below is our enriched `sources` YAML file. 

```
version: 2

sources:
  - name: nyc_bikes_nyc_bikes2014
    schema: nyc_bikes_nyc_bikes2014
    tables:
      - name: 2014-tripdata
        description: '{{ doc("tripduration") }}'
        columns:
          - name: _id 
            description: 'Unique id'
            tests:
              - dbt_expectations.expect_column_values_to_not_be_null

          - name: tripduration 
            description: '{{ doc("tripduration") }}'

          - name: starttime 
            description: ''

          - name: stoptime 
            description: ''

          - name: start station id 
            description: ''

          - name: start station name
            description: ''

          - name: start station latitude
            description: ''

          - name: start station longitude
            description: ''

          - name: end station id
            description: ''

          - name: end station name
            description: ''

          - name: end station latitude
            description: ''

          - name: end station longitude
            description: ''

          - name: bikeid
            description: ''

          - name: usertype
            description: ''

          - name: birth year
            description: ''

          - name: gender
            description: ''

  - name: nyc_bikes
    schema: nyc_bikes 
    tables:
      - name: citi_trips_round
```

Let's start by running the sole test at the trusty `tripduration` key via our single-line slingshot code: `dbt test --select sources`. 

Everything ran fine meaning there were no null values in this field. 

```
19:32:18  Concurrency: 1 threads (target='dev')
19:32:18  
19:32:18  1 of 1 START test dbt_expectations_source_expect_column_values_to_not_be_null_nyc_bikes_nyc_bikes2014_2014-tripdata__id  [RUN]
19:32:21  1 of 1 PASS dbt_expectations_source_expect_column_values_to_not_be_null_nyc_bikes_nyc_bikes2014_2014-tripdata__id  [PASS in 3.24s]
```

To see if our descriptions will be updated in the dbt documentation, simply run `dbt docs generate` followed by `dbt docs serve` to start the local server. 

![Sources descriptions](./images/sources_definitions.png)


You should see your dbt documentation updated with the descriptions for `nyc_bikes_nyc_bikes2014` table. 

Below is our `sources` YAML file in full with additional descriptions and tests.

```
version: 2

sources:
  - name: nyc_bikes_nyc_bikes2014
    schema: nyc_bikes_nyc_bikes2014
    tables:
      - name: 2014-tripdata
        description: '{{ doc("seed_2014_tripdata") }}'
        columns:
          - name: _id 
            description: 'Unique id'
            tests:
              - dbt_expectations.expect_column_values_to_not_be_null

          - name: tripduration 
            description: '{{ doc("tripduration") }}'

          - name: starttime 
            description: ''

          - name: stoptime 
            description: ''

          - name: start station id 
            description: ''

          - name: start station name
            description: ''

          - name: start station latitude
            description: ''

          - name: start station longitude
            description: ''

          - name: end station id
            description: ''

          - name: end station name
            description: ''

          - name: end station latitude
            description: ''

          - name: end station longitude
            description: ''

          - name: bikeid
            description: ''

          - name: usertype
            description: ''

          - name: birth year
            description: ''

          - name: gender
            description: ''

  - name: nyc_bikes
    schema: nyc_bikes 
    tables:
      - name: citi_trips_round
        description: '{{ doc("citi_trips_round") }}'
        tests:
          - dbt_expectations.expect_table_row_count_to_equal_other_table:
              compare_model: ref("citi_trips_minutes")
          - dbt_expectations.expect_column_pair_values_A_to_be_greater_than_B:
              column_A: tripduration
              column_B: trip_min_round
        columns:
          - name: tripduration
            description: '{{ doc("tripduration") }}'

          - name: starttime
            description: '{{ doc("starttime") }}'
          
          - name: stoptime
            description: '{{ doc("stoptime") }}'

          - name: start_station_id
            description: "Start Station ID"
            tests:
              - dbt_expectations.expect_column_values_to_not_be_null
          
          - name: start_station_name
            description: "Start Station Name"
            tests:
              - dbt_expectations.expect_column_values_to_not_be_null
              - dbt_expectations.expect_column_value_lengths_to_be_between:
                  min_value: 1 # (Optional)
                  max_value: 70 # (Optional)

          - name: start_station_latitude
            description: "Start Station Latitude"
          
          - name: start_station_longitude
            description: "Start Station Longitude"

          - name: end_station_id
            description: "End Station ID"
            tests:
              - dbt_expectations.expect_column_values_to_not_be_null

          - name: end_station_name
            description: "End Station Name"
            tests:
              - dbt_expectations.expect_column_values_to_not_be_null
              - dbt_expectations.expect_column_value_lengths_to_be_between:
                  min_value: 1 # (Optional)
                  max_value: 70 # (Optional)

          - name: end_station_latitude
            description: "End Station Latitude"

          - name: end_station_longitude
            description: "End Station Longitude"
          
          - name: bike_id
            description: "Bike ID"
          
          - name: usertype
            description: "User Type (Customer = 24-hour pass or 7-day pass user, Subscriber = Annual Member)"

          - name: birth_year
            description: "Year of Birth"

          - name: gender
            description: "Gender (unknown, male, female)"
            tests:
              - dbt_expectations.expect_column_values_to_be_in_set:
                  value_set: ['unknown','male','female']

          - name: customer_plan
            description: "The name of the plan that determines the rate charged for the trip"

          - name: trip_duration_min
            description: '{{ doc("trip_duration_min") }}'
            tests:
              - dbt_expectations.expect_column_max_to_be_between:
                  min_value: 16 # (Optional)
                  max_value: 326000 # (Optional)

          - name: trip_min_round
            description: '{{ doc("trip_min_round") }}'
            tests:
              - dbt_expectations.expect_column_max_to_be_between:
                  min_value: 16 # (Optional)
                  max_value: 100000 # (Optional)
```















