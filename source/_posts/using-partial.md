---
title: Using partial
summary: "Using partial"
date: "2023-10-01"
tags:
    - Python
---

Some libariries in python provide methods or functions that enable the user to pass a `function` object as an argument. One limitation that I faced was the impossibility to pass extra arguments associated with the user defined `function` object. This was a problem until I discover the role of `partial`.

## The problem

Two use cases, same problem:

```python
from functools import partial
from airflow import DAG
from airflow.models import Variable
from airflow.operators.python import PythonOperator
from airflow.utils.dates import days_ago

import logging
import sqlalchemy
import os


def process_bi_tables():
    prefix = "parent/path/folder"
    bucket = Variable.get("BUCKET")
    region = Variable.get("REGION")
    project_id = Variable.get("PROJECT_ID")
    postg_connex_string = f"{project_id}:{region}:{Variable.get('POSTGRE_SERVER_NAME')}"

    def postgre_pool():
        connector = Connector()
        return connector.connect(
            instance_connection_string= postg_connex_string,
            driver="your_driver",
            user=os.environ.get("DB_USER"),
            password=os.environ.get("DB_PASS"),
            db=os.environ.get("DB_NAME"),
        )

    postgre_pool = sqlalchemy.create_engine(
        "postgresql+pg8000://",
        creator=postgre_pool
    )

    tables = ["patients_doctors", "nights", "users", "doctors", "sensors"]
    views = ["nights_view"]

    with postgre_pool.connect() as postgre_conn:
        for name in tables + views:
            logging.info(f"Processing table : {name}")
            # read_query is not defined here to be brief
            query = read_query(bucket=bucket, prefix=prefix, filename=name)
            postgre_conn.execute(query)

    logging.info("BI databases updated")

with DAG(
    dag_id="pipeline_bi",
    default_args={"owner": "airflow"},
    schedule_interval="0 */3 * * *",
    start_date=days_ago(1),
    catchup=False,
) as dag:
    update_bi_tables = PythonOperator(
        task_id="update_bi_tables",
        python_callable=partial(
            process_bi_tables,
        ),
    )
```

Let's have a look to the `process_bi_tables` function:

- The purpose of this functions is to realize some data processing suing SQL scripts. We iterate through tables names and load `sql` scripts named the same way. The BI pipeline and Composer instances are deployed in different environments in US and Europe. By default Cloud Composer set up a bucket where you can find the `airflow.cfg` file and upload your dags and python modules. You can add there any folders and files you need to use. In our case, we need to load these `sql` scripts.
- First, you can see in the definition of the `pocess_bi_tables` function that we defined a variable `bucket` and `prefix`. They are defined inside the function which is not convenient if I want the code to be modular for each environment and avoid having to define each time some airflow variables. The `pocess_bi_tables` is passed to the `PythonOperator` as argument.
- Second, ntoice the way I created the `postgre_pool` function. It is an inner functions that return a `Connector` object that will be necessary to setup the connection with my database. This functions is passed to `sqlalchemy.create_engine` as argument.

I forced myself to define these two functions without any parameters because at first I had no idea how to use the `function` object as argument and pass its `args`. To circumvent this issue I had to define some variables inside the function body and this makes the code poorly mainainable and not that beautiful to read. This was until I found `partial`.

## `partial` in action

In the documentation you read:

>Returns a new partial object which when called will behave like `func` called with the positional arguments `args` and keyword arguments keywords. If more arguments are supplied to the call, they are appended to `args`. If additional keyword arguments are supplied, they extend and override keywords.

This looks exactly what I need. Let's refactor the code using `partial`, first with `postgre_pool`:

```python
def connect(connstring: str, user: str, password: str, db: str, driver: str):
    connector = Connector()

    return connector.connect(
        instance_connection_string=connstring,
        driver=driver,
        user=user,
        password=password,
        db=db,
    )

def process_bi_tables():
    prefix = "parent/path/folder"
    bucket = Variable.get("BUCKET")
    region = Variable.get("REGION")
    project_id = Variable.get("PROJECT_ID")
    postg_connex_string = f"{project_id}:{region}:{Variable.get('POSTGRE_SERVER_NAME')}"

    postgre_pool = sqlalchemy.create_engine(
        "postgresql+pg8000://",
        creator=partial(
            utils.connect,
            user=Variable.get("POSTGRE_DB_USER"),
            password=Variable.get("POSTGRE_DB_PASSWORD"),
            db=Variable.get("POSTGRE_DB_NAME"),
            driver="pg8000",
            connstring=postg_connex_string,
        ),
    )
```

We factored out the `postgre_pool` function into a function called `connect`. To be used in our `create_engine` we used the following construct:

```python
partial(
  my_functions,
  **kwargs
)
```

The beauty of partial is that it will passed the `function` object but also its arguments. It is pretty convenient. Moreover we can use `connect` for any database and we do not need to use inner functions with handcoded arguments.

We can do a similar move for the `process_bi_tables` function:

```python
def process_bi_tables(bucket: str, prefix: str):
    region = Variable.get("REGION")
    project_id = Variable.get("PROJECT_ID")
    postg_connex_string = f"{project_id}:{region}:{Variable.get('POSTGRE_SERVER_NAME')}"
    postgre_pool = sqlalchemy.create_engine(
        "postgresql+pg8000://",
        creator=partial(
            utils.connect,
            user=Variable.get("POSTGRE_DB_USER"),
            password=Variable.get("POSTGRE_DB_PASSWORD"),
            db=Variable.get("POSTGRE_DB_NAME"),
            driver="pg8000",
            connstring=postg_connex_string,
        ),
    )

    tables = ["patients_doctors", "nights", "users", "doctors", "sensors"]
    views = ["nights_view"]

    with postgre_pool.connect() as postgre_conn:
        for name in tables + views:
            logging.info(f"Processing table : {name}")
            query = utils.read_query(bucket=bucket, prefix=prefix, filename=name)
            postgre_conn.execute(query)

    logging.info("BI databases updated")
```

Now both `bucket` and `prefix` are function parameters. With partial we can use them in our `PythonOperator`:

```python
with DAG(
    dag_id="pipeline_bi",
    default_args={"owner": "airflow"},
    schedule_interval="0 */3 * * *",
    start_date=days_ago(1),
    catchup=False,
) as dag:
    update_bi_tables = PythonOperator(
        task_id="update_bi_tables",
        python_callable=partial(
            process_bi_tables,
            bucket=Variable.get("BUCKET-AIRFLOW"),
            prefix="query/task_bi_processing",
        ),
    )
```

## Conclusion

I am glad I find my way through `partial`. It was specifficaly designed for this use case and it is very elegant to use in your code.

## References

- https://docs.python.org/3/library/functools.html
