---
title: "tsfresh on Large Data Samples - Part II"
layout: post
date: 2020-04-02 8:00
tag: [dataengineering, python, tsfresh]
image: /assets/images/time-series.png
headerImage: true
projects: false
hidden: false
description: ""
category: 'blog'
author: nils
externalLink: false
---

In the [last post](../tsfresh-on-cluster-1/) we have explored how `tsfresh` automatically extracts many time series features from your input data.
We have also discussed two possibilities to speed up your feature extraction calculation: using multiple cores on your local machine (which is already tuned on by default) or distributing the calculation over a cluster of machines.

In this post we will go one step further: what happens if your data is so large that loading the data into your scheduling machine is not an option anymore?
For many applications, this is is not the case (and keeping everything local speeds up the development cycle as well as decreased the amount of possible mistakes).
However, sometimes you really need to think big.

## Apache Spark and dask bindings

There exist multiple solutions to solve the problem of distributing not only the work, but also the data - actually the whole field of big data and data engineering builds around that.
The general idea is to only load a fraction of the data into each machine and perform the calculation on this partition of the data - instead of loading all the data at the same time.
The inputs and results are distributed either via network transmissions or via a distributed file system, such as `HDFS`.

Two of the best known examples for this (at least in the Python world) are [`Apache Spark`](https://spark.apache.org/) (with its python bindings `PySpark`) and [`dask`](https://dask.org/).
We will not cover the basics of `Apache Spark` nor `dask` here - they are both wonderful libraries and if you never used them you should definitely check them out!
If you do not want to use any of those frameworks, but you still have too much data to handle, have a look into the next section where we will discuss a simplified (but also less powerful) solution.

Since version 0.15, `tsfresh` contains convenience functions to input a `Spark` dataframe or a `dask` dataframe into `tsfresh` (remember: normally you can only use `pandas` dataframes).
They are both defined in the `tsfresh.convenience.bindings` module (with documentation [here](https://tsfresh.readthedocs.io/en/v0.15.1/api/tsfresh.convenience.html)) and we will cover them in the remainder of this section.

The idea is as follows:
* You take care of bringing the data into the correct format for `tsfresh` (see below, what this means). For `pandas` dataframes this is something we can do on our side. But for those distributed systems, so much careful craftmanship goes into transforming the data - we can not handle this in a general way.
* You use one of the bindings mentioned above, to add the `tsfresh` feature extraction to the data pipeline. Both the input as well as the output of these functions are `dask` or `PySpark` dataframes.
  Internally, `tsfresh` will convert each chunk of your data to a `pandas` dataframe and use the normal feature extraction procedure.
  Afterwards, the result is converted back - so you will not notice this.
* You continue working with this dataframe, e.g. store it as a file or execute additional data transformations.

As an example, we will again use the robot failure datasample from our [Quickstart](https://tsfresh.readthedocs.io/en/latest/text/quick_start.html).
Even though it is still small enough to fit into your memory, we will treat it as "big data" and spill it out to multiple `parquet` files on disk.
Make sure to have a pandas parquet binding installed for this (such as `pyarrow`, see [installation instructions](https://arrow.apache.org/install/)) or use a different format.

```python
from tsfresh.examples import robot_execution_failures
import os

robot_execution_failures.download_robot_execution_failures()
timeseries, _ = robot_execution_failures.load_robot_execution_failures()

def store_data(data_chunk):
    data_id = data_chunk["id"].iloc[0]

    os.makedirs(f"data/{data_id}", exist_ok=True)
    data_chunk.to_parquet(f"data/{data_id}/data.parquet", index=False)

timeseries.groupby("id").apply(store_data)
```

Lets start with `Apache Spark` first.

### (Py)Spark and tsfresh

`Apache Spark` is basically *the* framework for writing and distributing fault-tolerant data pipelines.
Even though it is written in Java (and Scala), it has very good and well documented python bindings called `pyspark`.
After having installed `Apache Spark` (see [here](https://spark.apache.org/docs/latest/)), you need to create a `Spark` cluster for distributing the work.
As usual, there exist many ways for this (see [here](https://spark.apache.org/docs/latest/cluster-overview.html) for a start), but as an example we will just use the standalone cluster, which is started when you do not specify differently (also called "local" mode).
We will demonstrate the calculation in the following with a `pyspark` interactive console, but you can of course also write it into a python script and submit it via `spark-submit` (check out the [documentation](https://spark.apache.org/docs/latest/submitting-applications.html)).

**Important**: `Spark` leverages the [`arrow`](https://arrow.apache.org/docs/index.html) bindings for efficient transformation between `pandas` and `Spark` dataframes.
Therefore, you need to have arrow installed.
In a recent `arrow` version, the internal data format has changed and is now incompatible with `Spark`.
For it to still work, you need to add the line

```bash
  ARROW_PRE_0_15_IPC_FORMAT=1
```

to your `Spark` environment settings file `$SPARK_HOME/conf/spark-env.sh` as stated in the [documentation](https://spark.apache.org/docs/latest/sql-pyspark-pandas-with-arrow.html#compatibiliy-setting-for-pyarrow--0150-and-spark-23x-24x).

Now let's create a data pipeline!
We will use `Spark`'s structured declarative dataframe API in the following.
Remember that `Spark` will only build up the computation DAG until you trigger an action in the end.

So first, lets spin up an interactive `pyspark` shell and read in the `parquet` files.
As we are running in local mode we do not need to make sure that the data is available for all workers (as there is only one local worker).
In a productive environment you would probably use `S3` or `HDFS` or any other shared data storage.

```bash
$ pyspark
```

```
>>> df = spark.read \
...         .parquet("data/*/data.parquet")
>>> df.printSchema()
root
|-- id: integer (nullable = true)
|-- time: integer (nullable = true)
|-- F_x: integer (nullable = true)
|-- F_y: integer (nullable = true)
|-- F_z: integer (nullable = true)
|-- T_x: integer (nullable = true)
|-- T_y: integer (nullable = true)
|-- T_z: integer (nullable = true)

>>> df.show()
+---+----+----+---+-----+----+----+---+
| id|time| F_x|F_y|  F_z| T_x| T_y|T_z|
+---+----+----+---+-----+----+----+---+
| 71|   0| -59| 44| -205|-305|-226| 37|
| 71|   1| -71| 55| -273|-368|-271| 42|
| 71|   2| -75| 58| -305|-403|-294| 48|
| 71|   3| -89| 66| -408|-471|-342| 51|
| 71|   4| -95| 64| -461|-503|-372| 47|
| 71|   5|-100| 67| -524|-545|-400| 51|
| 71|   6|-104| 55| -567|-547|-429| 56|
...
```

This is the format of the robot data.
`id` is the identifier for each time series, `time` is the time (sorting) parameter and the `F_*` and `T_*` are the different value series (remember: what we call the `kind` of the time series).
Please note that this data format and the pre-processing we will perform in the following is just an example - your data might look differently and other data transformation steps might be necessary.
For example if your column names are different (e.g. the `id` column is not called "id"), you will need to edit the steps accordingly.

The feature extraction will later run on each of these series separately - so we first need to group it both by the timeseries identifier `id` and the `kind`.
However, the data is not easily groupable by `kind` (data of the same kind are not separated) - so we first need to reshape the data.
This transformation is usually called *melting* and there exist various ways to do this in `PySpark`.
For example you could follow one of the answers to this [StackOverflow discussion](https://stackoverflow.com/questions/41670103/how-to-melt-spark-dataframe).
Just make sure to change the column name identifiers if needed.

```
>>> ... your melt function, e.g. see in the link above
>>> df_melted = melt(df, id_vars=["id", "time"],
...                  value_vars=["F_x", "F_y", "F_z", "T_x", "T_y", "T_z"],
...                  var_name="kind", value_name="value")
>>> df_melted.printSchema()
root
|-- id: integer (nullable = true)
|-- time: integer (nullable = true)
|-- kind: string (nullable = false)
|-- value: integer (nullable = true)
>>> df_melted.show()
+---+----+----+-----+
| id|time|kind|value|
+---+----+----+-----+
| 71|   0| F_x|  -59|
| 71|   0| F_y|   44|
| 71|   0| F_z| -205|
| 71|   0| T_x| -305|
| 71|   0| T_y| -226|
| 71|   0| T_z|   37|
| 71|   1| F_x|  -71|
| 71|   1| F_y|   55|
| 71|   1| F_z| -273|
| 71|   1| T_x| -368|
| 71|   1| T_y| -271|
| 71|   1| T_z|   42|
...
```

After melting, the resulting dataframe is a concatenated set of each `kind` of time series, while keeping the `id` and `time` columns.
When we now group by `id` and `kind`

```
>>> df_grouped = df_melted.groupby(["id", "kind"])
```

each grouped chunk of data only contains the data of one `kind` and one `id`.
We are now ready to use `tsfresh`!
The preprocessing part might look different for your data sample,
but you should always end up with a dataset grouped by `id` and `kind` before using `tsfresh`.

With the given column names in the example, the call to `tsfresh` looks like this:

```
>>> from tsfresh.convenience.bindings import spark_feature_extraction_on_chunk
>>> from tsfresh.feature_extraction import ComprehensiveFCParameters
>>> features = spark_feature_extraction_on_chunk(df_grouped, column_id="id",
...                                              column_kind="kind",
...                                              column_sort="time",
...                                              column_value="value",
...                                              default_fc_parameters=ComprehensiveFCParameters())
```

Please remember that `Spark` will only trigger the calculation once you call an action, so it is still only building up the calculation DAG.

Internally, `tsfresh` will call the following on each grouped chunk:
* Transform the chunk to a pandas dataframe (which is very efficient due to the usage of `arrow`).
* Extract the features using the parameters and settings you gave (check out [the documentation](https://tsfresh.readthedocs.io/en/v0.15.1/text/feature_extraction_settings.html) for more information in the settings).
    We are using the full set of feature calculators here just as an example.
* Then convert back to a format `PySpark` understands.

In the end, it will return a `Spark` dataframe, where each line is the value for a different feature of a different time series `id` and `kind`.

```
>>> features.printSchema()
root
|-- id: long (nullable = true)
|-- variable: string (nullable = true)
|-- value: double (nullable = true)
>>> features.show()
+---+--------------------+-------------------+
| id|            variable|              value|
+---+--------------------+-------------------+
| 12|     F_x__sum_values|               -6.0|
| 12|         F_x__median|               -1.0|
| 12|           F_x__mean|               -0.4|
| 12|         F_x__length|               15.0|
...
```

Depending on your use case, you might want to bring them into the usual tabular form (but please note that this might imply heavy shuffling and especially some immediate calculations as the column names need to be known):

```
>>> from pyspark.sql import functions as F
>>> pivoted_features = features.groupby("id").pivot("variable")
>>> feature_table = pivoted_features.agg(F.first("value"))
>>> feature_table.printSchema()
root
|-- id: long (nullable = true)
|-- F_x__length: double (nullable = true)
|-- F_x__maximum: double (nullable = true)
...
```

In any case, you can now apply additional transformations or write out the result of the calculation.
As long as you have enough workers, this basically scales infinitely.

### Dask and tsfresh

In principle interacting with `tsfresh` from `dask` follows the same principles as with `Spark`, so lets quickly walk through them:

1. Spin up a dask cluster. There are again multiple ways how to do this depending on your environment. A very good starting point is the [dask documentation](https://docs.dask.org/en/latest/setup.html).
    If you do not have a cluster to play around, local mode will also work (which means you do not need to setup or import anything).
    For testing, you can use an interactive python shell or a `jupyter` notebook.
2. Read in the data sample

    ```
    >>> from dask import dataframe as dd
    >>> # Make sure to setup your client here if you have a cluster
    >>> df = dd.read_parquet("data/*/data.parquet")
    ```

3. Bring the data into the same format as above, where data with different `id` and `kind` can be separated easily. Fortunately, `dask` has already a `melt` feature, so you just need to call it:

    ```
    >>> df_melted = df.melt(id_vars=["id", "time"],
    ...                     value_vars=["F_x", "F_y", "F_z", "T_x", "T_y", "T_z"],
    ...                     var_name="kind", value_name="value")
    >>> df_melted.columns
    Index(['id', 'time', 'kind', 'value'], dtype='object')
    >>> df_melted.head()
        id  time kind  value
    0   1     0  F_x     -1
    1   1     1  F_x      0
    2   1     2  F_x     -1
    3   1     3  F_x     -1
    4   1     4  F_x     -1
    ```

4. Now separate the data for different `id` and `kind` by grouping

    ```
    >>> df_grouped = df_melted.groupby(["id", "kind"])
    ```

5. The data is in the correct format - we can apply `tsfresh`! We are again using the full set of feature calculators as an example here, but have a look into [the documentation](https://tsfresh.readthedocs.io/en/v0.15.1/text/feature_extraction_settings.html) for more information.

    ```
    >>> from tsfresh.convenience.bindings import dask_feature_extraction_on_chunk
    >>> from tsfresh.feature_extraction.settings import ComprehensiveFCParameters
    >>> features = dask_feature_extraction_on_chunk(df_grouped,
    ...                                             column_id="id",
    ...                                             column_kind="kind",
    ...                                             column_sort="time",
    ...                                             column_value="value",
    ...                                             default_fc_parameters=ComprehensiveFCParameters())
    ```

    Again, this will internally transform each chunk into a pandas dataframe, apply the feature extraction and transform back into a dask dataframe.
    The result will be a dataframe with one extracted feature for one `id` and `kind` per row.

    ```
    >>> features.columns
    Index(['id', 'variable', 'value'], dtype='object')
    >>> features.head()
                id                 variable      value
    id kind
    11 T_x  0  11          T_x__sum_values -49.000000
            1  11              T_x__median  -5.000000
            2  11                T_x__mean  -3.266667
            3  11              T_x__length  15.000000
            4  11  T_x__standard_deviation   3.549022
    ```

6. Continue with your calculation, e.g. transform the results into the usual tabular form (again: this might be computationally very intensive!):

    ```
    >>> features = features.categorize(columns=["variable"])
    >>> features = features.reset_index(drop=True)
    >>> feature_table = features.pivot_table(index="id", columns="variable",
    ...                                      values="value", aggfunc="sum")
    ```

    In case you are wondering: the aggregation function "sum" does not really matter here: there is only ever a single value per feature name and identifier.

You might ask yourself what the difference is between using a `ClusterDaskDistributor` (which distributes to a `dask` cluster) as described in the last post and this `dask` binding.
* The `ClusterDaskDistributor` allows you to distribute the feature extraction calculation via a `dask` cluster of workers, while still keeping all the data in a non-distributed `pandas` dataframe format.
  It is therefore useful if you want to parallelize the calculation, but the amount of data is still small enough for you to handle with `pandas` (on a single machine).
* The `dask` (or `PySpark`) bindings described here allow you to have a non-pandas input, where the data itself is already distributed.
  It gives you much more flexibility to add `tsfresh` into your data pipeline.

The data preprocessing depends a lot on the shape of your data and the way shown here is most likely not the most efficient way for you (because it might lead to a lot of shuffling or re-partitioning).
Please think about how you can transform your data efficiently into the input format for `tsfresh`.

## Side Note: luigi - the simple way

You have a lot of data which does not fit into memory, but you do not want to add the burden of using a distributed framework (such as `dask` or `PySpark`) to your project?
Then read on!

In the following, we will follow the ideas of a distributed framework such as `dask` or `Apache Spark` without actually using one (which of course also has some downsides, see below).
The idea is:
* Remember that we can extract the features for a time series of given `id` and `kind` independently from others.
* This means we can separate each cycle of: 1. read in the data for a specific `kind` and `id`, 2. extract the features on this data, 3. write out the data.
* Now we just need to orchestrate these work tasks.

For this part, we will use the powers of `luigi`, a task orchestrator framework.
If you have never used `luigi` have a look into the [documentation](https://luigi.readthedocs.io/en/stable/) for a first overview before you continue.
We will also try to walk through the basic ideas in the following.

We will also assume that your data is stored to disk (maybe on a distributed storage like `HDFS`) and already partitioned by `id` and `kind`.
For example, it might be in the form

    <path>/<id>/<kind>/input.parquet

If you want to generate some test data with the robot dataset, you can use the following python snippet:

```python
from tsfresh.examples import robot_execution_failures
import os

robot_execution_failures.download_robot_execution_failures()
timeseries, _ = robot_execution_failures.load_robot_execution_failures()

def store_data(data_chunk):
    data_id = data_chunk["id"].iloc[0]
    data_kind = data_chunk["kind"].iloc[0]

    os.makedirs(f"data/{data_id}/{data_kind}", exist_ok=True)
    data_chunk.to_parquet(f"data/{data_id}/{data_kind}/input.parquet", index=False)

timeseries.melt(id_vars=["id", "time"],
                value_vars=["F_x", "F_y", "F_z", "T_x", "T_y", "T_z"],
                var_name="kind", value_name="value").groupby(["id", "kind"]).apply(store_data)
```

Please note that the data is basically already melted due to the way it is stored.
It is also possible to store the data only separated by `id` and let `tsfresh` extract the features for all `kind`s simultaneously.
Feel free to adjust the `luigi` script below to your needs.

The core building block of `luigi` are tasks.
Tasks can have dependencies among each other, they define which output files they create and of course what to do when the task runs.
They are controlled by parameters, which can be used to distinguish different instances of the same task.
Let's define a `Task` for the cycle of reading, extracting and writing the data as described above.

```python
import luigi
import pandas as pd
from tsfresh import extract_features

# Where the input data is stored.
DATA_INPUT_NAME = "data/{data_id}/{data_kind}/input.parquet"
# Where the output data will be stored
DATA_OUTPUT_NAME = "data/{data_id}/{data_kind}/output.parquet"

class FeatureExtractorTask(luigi.Task):
    """
    Task to extract the features for one time series
    of given `id` and `kind`.
    Reads in the data, extracts the features and stores the data again.
    """
    data_id = luigi.Parameter()
    data_kind = luigi.Parameter()

    def output(self):
        """Define what this task will output"""
        return luigi.LocalTarget(DATA_OUTPUT_NAME.format(data_id=self.data_id,
                                                         data_kind=self.data_kind))


    def run(self):
        """Define, what the task will actually do"""
        # 1. Read in the time series data from disk
        input_file = DATA_INPUT_NAME.format(data_id=self.data_id, data_kind=self.data_kind)
        df = pd.read_parquet(input_file)

        # 2. Extract the features.
        # Turn of multiprocessing - the parallelism comes with multiple luigi workers.
        features = extract_features(df, column_id="id",
                                    column_kind="kind",
                                    column_sort="time",
                                    column_value="value",
                                    n_jobs=0)

        # 3. Store the data
        features.to_parquet(self.output().path)
```

As you can see, the code is quite straightforward.
We are again assuming only local setup for this example - in a real world application the input and output paths will be on a shared file system (`NFS`, `HDFS`, `S3`) so that both the scheduler and the worker can read/write the files.

Once we have defined a task for this cycle, we just need to run it.
There exist multiple ways to do this - you could use a local scheduler, a central scheduler, or even a batch system (maybe with the help of my package [b2luigi](https://b2luigi.readthedocs.io/en/stable/) ;-)).
As an example, we will just define a list of tasks to run and give it to the `luigi.build` function.
Also, we are only using a single worker and a local scheduler.

```python
# ... continue from above

if __name__ == "__main__":
    task_list = []
    for data_kind in ["F_x", "F_y"]:
        for data_id in range(1, 5):
            task_list.append(FeatureExtractorTask(data_kind=data_kind, data_id=str(data_id)))
    luigi.build(task_list, local_scheduler=True)
```

Running this script will give you a happy smiley output and your output data stored in the specified paths.
For distributing the work among several machines, start a central scheduler

```bash
$ luigid
```

And remove the `local_scheduler` flag.
Each call to the luigi script will now start another worker, which will connect to the central scheduler and start processing work (if your code is running on multiple machines, you need to give the scheduler hostname as well).
The [documentation](https://luigi.readthedocs.io/en/stable/running_luigi.html) describes more options.
If you want to test it out, make sure to increase the number of `id`s it needs to process - otherwise it will run out of work quickly.

Using `luigi` comes with the benefit of a very simple code base and a simplified execution model.
But of course can not replace a distributed framework such as `dask` or `Apache Spark` with all its features:
* You basically need to implement all the data handling by yourself (e.g. joining) using file operations. If you need this, you should really think about using one of the frameworks.
* There is a lot of file IO involved (luigi tasks can only communicate via file output).
* There is no "high-level" API for data pipelines, everything needs to be implemented as single tasks by hand.

But following the KISS principle, I would always prefer a small `luigi` application over running a complex distributed application (and have done so successfully several times in the past).

## Summary

After all these possibilities, the question arises: When should you use what?

Let's try to give some usage hints:
* Use the simplest possibility that solves your task. If you can effort to read in all the data in memory, stick with `pandas` and use multiprocessing or a distributor. If you do not need to build a complex data pipeline, but rather only extract the features, think about using `luigi` (or anything alike, such as Airflow or celery)
* Same applies to single or multi-node calculation: less machines, less problems. Try to stay with single-processing or local calculations as long as possible.
* If the rest of your data pipeline is already implemented in another framework which is not mentioned here, try to stick with your framework. The code for the bindings is actually [quite simple](https://github.com/blue-yonder/tsfresh/blob/master/tsfresh/convenience/bindings.py) - so you can easily reproduce it in your framework.

There is a lot more to discover when it comes to distributed computing or distributed data.
If you find out about interesting bindings between your framework and `tsfresh` or of you developed a cool `distributor` you want to share, we are already happy for pull requests.
Happy coding!

Big thanks to [@dotcsDE](https://twitter.com/dotcsDE) for great suggestions!