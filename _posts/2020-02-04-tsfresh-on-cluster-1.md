---
title: "Time Series Feature Extraction on (Really) Large Data Samples"
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

Time Series data is everywhere today.
From stock market trends to EEG measurements, from Industry 4.0 production lines to IoT sensors - temporarily annotated data hides in a lot of places.
It comes in many flavours, sizes and complexities.

In this series of two posts we will explore how we can extract features from time series using `tsfresh` - even when the time series data is very large and the computation takes a very long time on a single core.

But first, let's define some common properties of time series data:
* The data is indexed by some discrete "time" variable.
  Often (but not necessarily) in fixed sampling intervals.
  The index does not need to be the wall clock time in all cases -  measurements of the height profile of a produced semiconductor in a  plant might share many properties with a classical time series although  the index is the location on the sensor here.
  To be general, we can call it the `sort` parameter - but to keep things simple in the following we still talk about the time.
* We assume there is a temporal dependency in the data - measurements at a certain timestamp are not just random, but depend (at least to a certain amount) on previous data points or on an underlying (maybe unknown) process.
* It does not need to be a single time series, but can consist of multiple distinct measurements.
  For example you can analyse the stock market prices of Apple, Alphabet and Facebook - so you have three different time series.
  We will say in the following that each of those time series has a different identifier, or `id`.
  Usually, you are interested in the different behavior of the data for different `id`s.
* Even within the data for the same `id`, you could have multiple measurements at each time step: maybe you measure the temperature and humidity at each day in the year at different locations.
  In our nomenclature, the different locations would be different `id`s and temperature and humidity are different `kind`s of the time series.

Many tasks around time series involve the application of machine learning algorithms, e.g. to classify the collected data of different `id`s or to predict future values of the time series.
But before an algorithm can deduce, if for example a measured heart rate shows first signs of a heart attack or a stock chart line indicates the next big thing, useful features of the time series need to be extracted.
Using these features, a machine learning algorithm can then learn to differentiate between signal and background data.

Side note: in recent times a lot of effort has been put into time series analysis with deep learning methods.
While this approach has definitely its benefits, it is only applicable for a certain range of problems.
There exist a lot of documentation on how to use deep learning with time series data (e.g. [here](https://towardsdatascience.com/time-series-analysis-with-deep-learning-simplified-5c444315d773)) so we will not cover it in this post.

Manual feature extraction is a time consuming and tedious task.
In most cases it involves thinking about possible features, writing feature calculator code, consulting library API documentation and drinking a lot of coffee.
And in the end, most of the features will not make it to the production machine learning pipeline anyways.

## Entering tsfresh

Therefore we invented [`tsfresh`](https://tsfresh.readthedocs.io/en/latest/) [^1], which is an automated feature extraction and selection library for time series data.
It basically consists of a large library of feature calculators from different domains (which will extract more than 750 features for each time series) and a feature selection algorithm based on hypothesis testing.
In todays post (which will be the first of two parts about feature extraction with large time series data), we will only cover the feature extraction, as this is typically the (computationally) time consuming part.
The feature extractors range from simple ones like `min`, `max`, `length` to more complex ones like autocorrelation, fast Fourier transformation or augmented Dickey Fuller tests.
You can find a list of all features [here](https://tsfresh.readthedocs.io/en/latest/text/list_of_features.html).

To extract the full set of features, all you need to do is installing `tsfresh` (via `pip` or `conda`) and calling with your `pandas` dataframe `df`:

```python
from tsfresh import extract_features

df_features = extract_features(df, column_id="id", column_sort="time")
```

The resulting `pandas` dataframe `df_features` will contain all extracted features for each time series `kind` and `id`.
`tsfresh` understands multiple input dataframe schemas, which are described in detail [in the documentation](https://tsfresh.readthedocs.io/en/latest/text/data_formats.html).
You can also control which features are extracted with the settings parameters (default is to extract all features from the library with reasonable default configuration).


## Challenge: Large Data Samples

So far, so easy.
But what happens if you not only have a bunch of time series, but multiple?
Thousands, millions?

The time spent on feature extraction scales linearly with the number of time series.
If you have gigabytes of time series, you should better grab another cup of coffee!

Well, or you read on! :-)
In this series of two posts we are going to discuss four possibilities to speed up the calculation.
Depending on the amount of data (and resources) you have, you can choose between:

1. Multiprocessing, if your data fits into a single machine but you want to leverage multiple cores for the calculation (this post).
2. `tsfresh`'s distributor framework, if your data still fits into a single machine but you want to distribute the feature calculation over multiple machines to speed up the calculation (this post).
3. If your data does not fit into a single machine, chances are high you are already using libraries like `Apache Spark` or `dask` for handling the data (before `tsfresh`), so we will discuss how to interact from them with `tsfresh` ([next post](../tsfresh-on-cluster-2/)).
4. And if you are not using `dask` or `Apache Spark` and your data does still not fit into a single machine, you can still leverage the power of a task scheduler such as `luigi` for this ([next post](../tsfresh-on-cluster-2/)).

If you want to follow along with the code examples, make sure to install the most recent version of `tsfresh` (the following was tested with v0.15.1).
You also need an example data set for testing.
`tsfresh` comes with multiple example data, so let's choose one of it: the robot failure time series data.
You can learn more on the data sample from the [UCI page](http://archive.ics.uci.edu/ml/datasets/Robot+Execution+Failures).
To load it, add the following lines to your script:

```python
from tsfresh.examples import robot_execution_failures

robot_execution_failures.download_robot_execution_failures()
df, target = robot_execution_failures.load_robot_execution_failures()
```

There is a lot to discover, so let's start!
Feel free to immediately jump into one of the parts, if you already know what you are looking for.

## Multiprocessing

As a first step to speed up the calculation of features, we can distribute the feature extraction to multiple cores.
Actually, if you have executed the code example from above, you have already done so!
`tsfresh` comes with native support for multiprocessing and will use all of your CPU cores for the calculation.
For this, each time series of distinct `kind` and `id` will be treated independently and send to a different core, where the feature extraction will happen.
The results are collected in the end and combined into a large dataframe.

You can control the multiprocessing behavior with parameters passed to `extract_features`:
* `n_jobs`: how many jobs to run in parallel (defaults to all your CPUs)
* `chunksize`: how many "chunks" of time series (a chunk is one combination of `id` and `kind`) a single job should extract the features

Especially the chunksize has a large potential for optimizations.
By default, it is chosen according to a heuristic which worked best in our tests - use it to balance between parallelism speedup and introduced overhead.
For most of the applications though, you can keep the defaults.

**Note**:
As `tsfresh` uses Python's `multiprocessing` library under the hood, you need to fulfill all the requirements for its usage.
Especially on Windows this means you need to wrap your code with a `if __name__ = '__main__'` line, otherwise it will fail (See [here](https://docs.python.org/3/library/multiprocessing.html#the-spawn-and-forkserver-start-methods) for more information).
You can also turn multiprocessing off by setting `n_jobs` to 0.

## Distributors

Using multiple cores instead of one core for the feature calculation is already better, but how can you increase the processing speed even more?
To distribute the calculation of the features to multiple machines, `tsfresh` gives you the possibility to define a *distributor*.
The data is still loaded on your local machine (and is required to be a `pandas` dataframe, we will release this constraint in the next post), but you can use a distributor to e.g. let the heavy lifting of the feature calculation be done on multiple machines.
This has the benefit of speeding up your computation without the need to change anything on your usual data science pipeline.

Distributors offer a very general framework for implementing your own way, how you want your calculation be shuffled on your cluster.
As an example, an implementation of a distributor utilizing a cluster of `dask` workers comes already shipped with `tsfresh`.
If you do not know what `dask` is: up to now think of it as a possibility to distribute work over a cluster of machines.
We will come to the more complicated features of dask in the next post.

**Please note**: with a distributor you can only control how the feature extraction calculation is distributed.
This means the `dask` distributor just uses `dask`'s capabilities to use multiple workers for performing a task.
We are not (yet) using any of `dask`'s distributed data features - this means you still need to be able to load the data into your (scheduling/driver) machine all at once.

To use it, all you need to do is to define an instance of the `ClusterDaskDistributor` and pass it to the `extract_features` call:

```python
from tsfresh.utilities.distribution import ClusterDaskDistributor

distributor = ClusterDaskDistributor(address="<dask-master>")
X = extract_features(df, distributor=distributor, ...)
```

The address of the dask master depends on the way you have set up your dask cluster.
Please refer to the awesome [dask documentation](https://docs.dask.org/en/latest/setup.html) for this!
For testing, you could create your own small `dask` cluster by calling (after having installed `dask` via `pip` or `conda`):

```bash
$ dask-scheduler
Scheduler at:   <dask-master>
```

in one terminal and

```bash
$ dask-worker <dask-master>
Start worker at:  ...
Registered to:    <dask-master>
```

in another one.
Use the printed scheduler address in your code.
Now, you can scale your computation very easily by just adding more workers (or using one of the other possibilities to scale a `dask` cluster, e.g. via kubernetes, YARN, etc.).
Please note that by default only a single CPU is used per worker.
If you want to run on more processes on each machine, use the `--nprocs` command line option when starting each worker.

Distributing work with `dask` is just an example.
If you want to support your own distribution framework, you can create your own `Distributor` class and implement the job scheduling logic.
Some documentation is available [here](https://tsfresh.readthedocs.io/en/latest/text/tsfresh_on_a_cluster.html).

## First Summary

So far we have covered how to extract time series features on large amount of data by speeding up the computation.
Either by distributing the feature extracting over multiple CPU cores on your local machine or by distributing the work over a cluster of machines, e.g. using a cluster of `dask` workers.
The clear benefit of using these possibilities: no need to change the rest of your pipeline.
You start and end with `pandas` dataframes and you still read in the data on your local machine - it basically looks like you would do your feature extraction on just a small sample.

In the [next post](../tsfresh-on-cluster-2/) we will go one step further: what happens if you need to distribute the data because it does not fit into a single machine?

Thanks to [@dotcsDE](https://twitter.com/dotcsDE) and [snwalther](https://twitter.com/snwalther) for reviewing this post!

[^1]: *Time Series FeatuRe Extraction on basis of Scalable Hypothesis tests (tsfresh - A Python package)*; Maximilian Christ, Nils Braun, Julius Neuffer, Andreas W. Kempa-Liehr; Neurocomputing 2018: DOI: [10.1016/j.neucom.2018.03.067](https://www.researchgate.net/deref/http%3A%2F%2Fdx.doi.org%2F10.1016%2Fj.neucom.2018.03.067?_sg%5B0%5D=tBc7DaUqEurrQ0vQZEv588MgNUdcIuBQFVnzpFU3SEDJQ01kEieoWsciCt1VcXq5dC5rcFgdWpjv8SbgnozIsU7vxQ.1zdUmVbdlNSY8cet2FDuNx6J-R-hq7vSi_R3xdciVwtgtPyQTEPQLVAHwOOcgqrKz85i_DQexYFCsWiVj1RGpg)
