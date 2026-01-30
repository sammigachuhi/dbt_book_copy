# The `dbt` architecture

In my first time working with dbt, I was overwhelmed with its architecture. It felt like that individual who is sitting before that large screen in a nuclear power plant and in charge of all the controls. Nevertheless, if people can gain confidence in holding a nuclear power plant on their fingertips, then surely you can crack dbt.

The main components that make up dbt are as follows:

* models

* tests

* documentation

* sources

Let's go through each one.

## Models 

This is the component of dbt that you will most likely work with. In dbt, a model is simply a SQL statement. As simple as that. dbt will use the SQL statements to perform the transformations in your data warehouse that have been defined in your SQL statement. For example, say I want to create a new column of the table in my Google BigQuery. I will create a SQL statement that does just that. That SQL statement is what is referred to as a model in dbt.

Below is an example of a model that creates a table called `customers`. The model is saved as `customers.sql`.

```
with customer_orders as (
    select
        customer_id,
        min(order_date) as first_order_date,
        max(order_date) as most_recent_order_date,
        count(order_id) as number_of_orders

    from jaffle_shop.orders

    group by 1
)

select
    customers.customer_id,
    customers.first_name,
    customers.last_name,
    customer_orders.first_order_date,
    customer_orders.most_recent_order_date,
    coalesce(customer_orders.number_of_orders, 0) as number_of_orders

from jaffle_shop.customers

left join customer_orders using (customer_id)
```

## Tests

"Do not put me to test", is a familiar statement from an impatient person. However, dbt allows us to test our data and see if it meets certain assertions. In other words, does our data meet the requirements that have been set for it? 

dbt offers two ways to perform your tests: 

1. generic, and,

2. custom tests. 

Generic tests involve just using a pre-defined test that comes packaged in dbt. For example, for every field key you place in a YAML file in dbt, you can specify which kind of test to perform on that particular field from the following options: `unique`, `not_null`, `accepted_values` and `relationships`.

* `unique` - the values should be radically distinctive all through
* `not_null` - there shouldn't be a missing value in the particular column name in the table
*`accepted_values` - only the values contained in the accepted values key will be considered valid. Anything outside of this will result in an error
*`relationships` - the values in this field can be referenced in a different column elsewhere in the table or on a different table altogether. 


An example of a generic test is below:

```
version: 2

models:
  - name: orders
    columns:
      - name: order_id
        tests:
          - unique
          - not_null
      - name: status
        tests:
          - accepted_values:
              values: ['placed', 'shipped', 'completed', 'returned']
      - name: customer_id
        tests:
          - relationships:
              to: ref('customers')
              field: id
```

For custom tests, these involve one creating a SQL model and referencing it in a YAML file using Jinja template language. 

For example, here is a custom test written in a SQL file called `transaction_limit_test.sql`. 

```
-- tests/transaction_limit_test.sql
select user_id, sum(transaction_amount) as total_spent
from {{ ref('transactions') }}
group by user_id
having total_spent > 10000  -- Assuming the limit is 10,000


```

The test is referenced in a YAML file and called over a column called `transactions`. 

```
models:
  - name: transactions
    tests:
      - transaction_limit_test

```

## Documentation

Now, the favourite part of dbt, and possibly the easiest is documentation. Documentation is the description of various components of your data. To write a description of any piece of your data, the `description` key is used. 

For example here is a description of a field called `event_id` inside a YAML file. 

```
version: 2

models:
  - name: events
    description: This table contains clickstream events from the marketing website

    columns:
      - name: event_id
        description: The D-day is the Deed day
        tests:
          - unique
          - not_null

```

Documentation will be performed where you have placed your tests. There is also a more complex, but scalable manner of writing descriptions. It uses jinja template tags. It works well for large data where the descriptions are many or the descriptions are shared across several tables. 


A short example of the jinja templates' documentation is shown below. The description is within a markdown file (`.md`) other than the one containing my field names. The descriptions will be like so:

```
{% docs table_events %}

I am not so very robust, but I’ll do the best I can.

Some text here

1) and here
2) and here
3) and also here

{% enddocs %}

```

So when one returns to their YAML file, they will reference the particular field of interest with the above description like so:

```
version: 2

models:
  - name: events
    description: '{{ doc("table_events") }}'

    columns:
      - name: event_id
        description: The D-day is the Deed day
        tests:
            - unique
            - not_null

```

## Sources 

`sources` enable one query the data in your data warehouse. Once you specify the existing table in your data warehouse under the `sources` key, you can access every data from within this table using SQL. To work with a source table, you first have to wrap it inside a `{{ source(table-name) }}` jinja template. Below is an example of how to declare a source.

```
version: 2

sources:
  - name: jaffle_shop
    database: raw  
    schema: jaffle_shop  
    tables:
      - name: orders
      - name: customers

  - name: stripe
    tables:
      - name: payments

```

You can reference the above source inside a SQL model like so:

```
select
  ...

from {{ source('jaffle_shop', 'orders') }}

left join {{ source('jaffle_shop', 'customers') }} using (customer_id)


```

dbt will thereafter know that it will perform some operations using data from the `orders` and `customers` data from the `jaffle_shop` --the origin of all our data in this example.




