# Hooks

At one time, I posed a question to a well-versed data engineer of why the technological space seems to be awash with exotic, outlandish names. Hooks in dbt seem to fit into the calibre of these outlandish names... for there is nothing regarding its purpose in dbt which seems to suggest it will lure and capture your data. 


You could take hooks as customized SQL models that are out of the box when it comes to dbt. There are various forms of hooks, namely:

1. `pre-hook` - executed before a model, snapshot or seed is built.

2. `post-hook` - executed after a model, snapshot or seed is built. 

3. `on-run-start` - executed at the start of implementing the following code executions: `dbt build`, `dbt compile`, `dbt docs generate`, `dbt run`, `dbt seed`, `dbt snapshot` or `dbt test`. 

4. `on-run-end` - executed at the end of the following code executions: `dbt build`, `dbt compile`, `dbt docs generate`, `dbt run`, `dbt seed`, `dbt snapshot` or `dbt test`. 

A confession to make: since I have minimal experience using hooks, I shall play it safe, thus the reason why this chapter is quite short. 

## Post-hooks

The format of writing a hook is:

```
{{ config(
  post_hook=[
    "<Place your SQl query here>"
  ]
) }}

SELECT * FROM raw_table

```

We tried creating a simple post-hook but no matter what we tried, `dbt` kept throwing back errors. Nevertheless, for more on how to set up hooks, see  [here](https://docs.getdbt.com/docs/build/hooks-operations) and [here](https://www.y42.com/docs/dbt-models/hooks).

















