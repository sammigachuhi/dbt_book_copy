# dbt Expectations package

What is dbt-expectations? dbt-expectations is an extension package for dbt which works much akin to the Great Expectations package for Python. It was intentionally designed to provide Great Expectations like features in dbt, but now from dbt itself rather than integrating Great Expectations (GE).

Unless you've used GE, you may be wondering what this is in the first place, and its okay to feel lost. GE is much like tests in the previous chapter, it conducts quality tests on your data, thus flagging those that deviate from the set assertions. 

I would put dbt-expectations and GE on the same plane and use an allegory to drive the point home: that of a car. When buying a car, there are some common checklist items, and others bespoke depending on your car model. For example, an ordinary car must have the following features:

* have four wheels
* have a driver's seat
* have a gear (whether manual or automatic)
* have headlights
* have a windshield

The above list can go on and on depending on your knowledge of cars. But your checklist can also contain some unique items, but which are a must-have depending on your car make. For example, here is a checklist of the Volvo XC60 T6:

* 0.9l/100km fuel consumption
* Allowed emissions 22g/km (the less the better)
* Hybrid fuel type

So if you go to a showrooms and the beautiful or handsome sales agent takes you to the Volvo XC60, you will be perusing it as you cross your checklist. dbt-expectations and GE work in the same way.


## `dbt-expectations` installation

According to the [documentation](https://hub.getdbt.com/calogica/dbt_expectations/latest/) dbt-expectations will work for dbt versions 1.7x and higher. Let's first pass this little test.

```
dbt --version
```

If you get your dbt-core version is above 1.7x, then you can proceed. If not, you need to update your dbt. You can do so using `python -m pip install --upgrade dbt-core` or if you want to be more specific, this will do: `python -m pip install --upgrade dbt-core==0.19.0`.

Ours, at the moment of writing this book, was version `1.8.7`. Therefore we have a clean bill of health to proceed.

dbt-expectations isn't installed in the same *type and enter* kind of means like we did for `dbt-core` and `dbt big-query`. Nevertheless, some code is written in some YAML files and from henceforth dbt recognises it.

First create a `packages.yml` file in the same level as your `dbt_project.yml` file. You can do so by running this command:

```
touch packages.yml
```

On the `packages.yml` file, insert the following:

```
packages:
  - package: calogica/dbt_expectations
    version: [">=0.10.0", "<0.11.0"]
```


Apart from that, the `dbt-date` dependency must also be installed. This is because `dbt-expectations` references it. However, this will be installed in the `dbt_project.yml` file rather than the `packages.yml` file. So inside the `dbt_project.yml` paste the following just before the `materializations` dictionary.

```
vars:
  'dbt_date:time_zone': 'Africa/Nairobi'
```

You may insert any valid timezone apart from the one specified above, but we highly suggest that you use your timezone.

Now run `dbt deps` to seal the deal by installing the `dbt-expectations` package.

```
dbt deps
```

Here is the output showing the successful installation of the package in our environment.

```
19:31:10  Running with dbt=1.8.7
19:31:12  Updating lock file in file path: /home/sammigachuhi/dbt_book2/dbt_book/package-lock.yml
19:31:13  Installing calogica/dbt_expectations
19:31:44  Installed from version 0.10.4
19:31:44  Up to date!
19:31:44  Installing calogica/dbt_date
19:31:45  Installed from version 0.10.1
19:31:45  Up to date!

```

## Types of `dbt-expectations` tests

`dbt-expectations` comes with a plethora of tests' functions which can be classified into the following categories.

* Table shape
* Missing values, unique values, and types
* Sets and ranges
* String matching
* Aggregate functions
* Multi-column
* Distributional functions

We will perform one test in each category just to exemplify the potential of `dbt-expectations`.

### Table shape

#### `expect_table_row_count_to_equal_other_table`

**Description**: Expect the number of rows in a model match another model.

We will expect the `citi_trips_round` and the `citi_trips_minutes` tables to have the same number of rows since their respective models used the same `citi_bike_trips` table. Therefore, the two tables should pass this test, or will they?

Since we only want to concentrate on the models within the `my_models` directory, just run: `dbt test --select models/my_models`

Here is the output.

```
-- snip --
07:57:38  2 of 8 FAIL 1 dbt_expectations_expect_table_row_count_to_equal_other_table_citi_trips_minutes_ref_citi_trips_round_  [FAIL 1 in 4.83s]
07:57:38  3 of 8 START test dbt_expectations_expect_table_row_count_to_equal_other_table_citi_trips_round_ref_citi_trips_minutes_  [RUN]
07:57:42  3 of 8 FAIL 1 dbt_expectations_expect_table_row_count_to_equal_other_table_citi_trips_round_ref_citi_trips_minutes_  [FAIL 1 in 4.53s]
-- snip --

```

This leaves one puzzled, why?

A close look at the model for `citi_trips_round` reveals the answer. This model was designed to only work on non-null rows, unlike the `citi_trips_minutes` which worked on all rows, null or not. Therefore the `citi_trips_round` had less rows and thus the generated the error. In case your tests results seem incongruent, it is always good to recheck the models to refresh your memory, as we did here.

In fact, dbt did a good job of generating an SQL to show us the error:

```
07:58:03  Failure in test dbt_expectations_expect_table_row_count_to_equal_other_table_citi_trips_minutes_ref_citi_trips_round_ (models/my_models/my_models.yml)
07:58:03    Got 1 result, configured to fail if != 0
07:58:03  
07:58:03    compiled code at target/compiled/dbt_book/models/my_models/my_models.yml/dbt_expectations_expect_table__c00100dada30a31f15f90b9c1ba0b295.sql
```

If you click on the destination of the SQL statement and copy the contents to the SQL query tab of BigQuery, you will see the difference in row count for the two tables. 

![Row count difference](./images/row_count_difference.png)

### Missing values, unique values, and types

#### `expect_column_values_to_not_be_null`

**Description**: Expect column values to not be null.

This is a *no-brainer* kind of test. Can you guess which columns in any of our tables in the `nyc_bikes` dataset should never be null? Here is a clue: station_id and station names, unless the biker teleports to or from somewhere!

So on the `my_models` YAML file, insert the below test on the `start_station_id`, `start_station_name`, `end_station_id` and `end_station_name` fields.

```
tests:
  - dbt_expectations.expect_column_values_to_not_be_null
```

You will get some interesting results. some of the tests fail for the `citi_trips_minutes` model because of the many null rows in the table. However, none of this particular test fail for the `citi_trips_round` table; it has zero null rows.

A caveat when using the `dbt_expectations.expect_column_values_to_not_be_null`, only add a colon `:` when specifying more optional parameters such as `row_condition: "id is not null" # (Optional)`. Otherwise, leave it out.

### Sets and Ranges

#### `expect_column_values_to_be_in_set`

**Description**: Expect each column value to be in a given set.

This test works best for where you are sure that a certain column will only accept certain values. A good example is the `gender` column. There can only be three results: male, female and other. Here we insert the test in our `citi_trips_minutes` and `citi_trips_round` models.

```
- name: gender
  description: "Gender (unknown, male, female)"
  tests:
    - dbt_expectations.expect_column_values_to_be_in_set:
        value_set: ['unknown','male','female']

```

If you run the above test for both the `citi_trips_minutes` and `citi_trips_round` models, the test will fail for the former. Why, because of the pesky null rows. However, to take the null rows into consideration and take them as accepted values in the `citi_trips_minutes` model only, we simply add an empty quotation marks, like so (`''`). Here is our modified test: `value_set: ['', 'unknown','male','female']`. The test will then pass for our `citi_trips_minutes` table.

### String matching

#### `expect_column_value_lengths_to_be_between`

**Description**: Expect column entries to be strings with length between a min_value value and a max_value value (inclusive).

Because our `citi_trips_minutes` table has several null rows, we will put the minimum expected value to be 0. Since we also want to catch those station names with overly long names, we will put the max value as 70. So for both the `start_station_name` and the `end_station_name`, we inserted the following test:

```
tests:
  - dbt_expectations.expect_column_values_to_not_be_null
  - dbt_expectations.expect_column_value_lengths_to_be_between:
      min_value: 1 # (Optional)
      max_value: 70 # (Optional)
```

We are glad to know both models passed this simple test.

### Aggregate

#### `expect_column_max_to_be_between`

**Description**: Expect the column max to be between a min and max value

You may wonder what the purpose of this test is. But don't dismiss it yet, it can come quite in handy when searching for outlier values. We will demonstrate it in catching overly long bike trips. However, this test needs some background knowledge of your data. 

Applying the below queries on BigQuery will help.

```
SELECT AVG(trip_duration_min) FROM dbt-project-437116.nyc_bikes.citi_trips_minutes; -- 16

SELECT MAX(trip_duration_min) FROM dbt-project-437116.nyc_bikes.citi_trips_minutes; -- 325167.48

SELECT * FROM dbt-project-437116.nyc_bikes.citi_trips_minutes
WHERE trip_duration_min > 200000;
```

Now lets place the limits of our maximum trip duration values to be between the average of 16 and some intermediate value such as 100,000 minutes (1,666 hours)!

```
tests:
  - dbt_expectations.expect_column_max_to_be_between:
      min_value: 16 # (Optional)
      max_value: 100000 # (Optional)
```

That will flag off some errors, but if you change the `max_value` parameter to `360000`, the tests will pass. However, in the `trip_min_round` field of the `citi_trips_round` model, we set the `max_value` as 100000 to demonstrate an error of this test.

### Multi-column

#### `expect_column_pair_values_A_to_be_greater_than_B`

**Description**: Expect values in column A to be greater than column B.

This kind of test comes in handy when you want to ensure that one of your columnar values is greater than, or less than that of a different column. A good example is a comparison of trip duration in seconds in the `tripduration` column versus the trip duration in minutes from `trip_min_round` column in our `citi_trips_round` table. Definitely time in seconds will always have a greater value in terms of length than the more concise minutes values!

In our `citi_trips_round` model, insert the test as follows:

```
- name: citi_trips_round
    description: '{{ doc("citi_trips_round") }}'
    tests:
      - dbt_expectations.expect_table_row_count_to_equal_other_table:
          compare_model: ref("citi_trips_minutes")
      - dbt_expectations.expect_column_pair_values_A_to_be_greater_than_B:
          column_A: tripduration
          column_B: trip_min_round
```

It surely does pass the test.

```
-- snip --
09:40:46  5 of 26 PASS dbt_expectations_expect_column_pair_values_A_to_be_greater_than_B_citi_trips_round_tripduration__trip_min_round  [PASS in 2.20s]
```

### Distributional functions

This is another category of shipped-in tests of `dbt-expectations`. However, they require some statistical homework to be conducted on your data prior to applying the tests. The tests under this category include: `expect_column_values_to_be_within_n_moving_stdevs`, `expect_column_values_to_be_within_n_stdevs` and `expect_row_values_to_have_data_for_every_n_datepart`. 

















