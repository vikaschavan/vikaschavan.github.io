---
title: "Lambda III: Data Science with Amazon Lambda"
layout: post
date: 2017-09-02 20:00
tag: [datascience, python, lambda]
image: /assets/images/lambda.png
headerImage: true
projects: false
hidden: false
description: "Amazon Lambda and pythons `map` together with zappa"
category: 'blog'
author: nils
externalLink: false
---
After we have done the basic setup in the [last post](../amazon-lambda-and-map/)
we are now ready to add some real data science application and fully use the parallelisation
of Lambda. We will write a very simple map-framework for this.

This is part three on my series on Amazon Lambda for Data Science. To see how this all
comes together, we will develop a real-world data science applications in this blog post series over the
next time.

* [Introduction: Why you should use Amazon Lambda](../why-you-should-use-amazon-lambda)
* [Part 2: Simple map framework for Amazon Lambda using zappa](../amazon-lambda-and-map/)
* [Part 3: Data Science with Amazon Lambda](../amazon-lambda-and-map-2/)
* [Part 4: Use the Power of Lambda](../amazon-lambda-and-map-3/)

You can find the code of this series on my [github account](https://github.com/nils-braun/amazon-lambda-datascience) (will be updated when we go on).
The version of this post is stored in tag *part-three*.

As you have now seen how to create a website using `flask`, `zappa` and Lambda, we (only) have to
add some content now. For this, we basically have to add three core points
* add functionality to read in the data the user sends with his request. We will make our live
  very easy here and just allow the data to be send with the request itself as CSV (you could also
  think about reading it from S3, different formats etc.)
* add the data science part. Again, we simplify things and do not really add some data science
  analysis or alike, but just calculate the mean in every column. We will use `pandas` for
  doing the calculations and the data handling. In case you are not familiar with `pandas`,
  I really advice you to do some tutorials and read some examples, because this package is
  one of the core concepts in data science.
* Do the calculations in a parallel way, by executing a lambda function for each "core" we want to have
  and do the calculation independently.

All three steps will be handled in this and the next post. So lets start!

Remember to install `pandas` into your virtual environment with

    pip install pandas

if you have not already done so.

## Add our data science application - the sequential way
First, we need some data to play with. You can either generate some on your own, e.g. by using this script:

{% highlight python %}
import pandas as pd
import numpy as np


def generate_data(length):
    for index in range(length):
        for time, random_value in enumerate(np.random.rand(100)):
            yield {"id": index, "value": random_value, "time": time}

if __name__ == '__main__':
    df = pd.DataFrame(generate_data(1000))
    df.to_csv("data.csv")
{% endhighlight %}


or you download some random data from the internet. The data should have at least two columns:
* the `id` column (not to be confused with pandas `index` column!), where every time series has a unique id.
* the `value`column with the actual value of the time series.
In other words, the data sample is a long list of values of different time series, partitioned by their id,
like so

    | id | value |
    |----|-------|
    | 0  | x1    |
    | 0  | x2    |
    | .. | ..    |
    | 0  | xN    |
    | 1  | y1    |
    | .. | ..    |



We first add the utility functions for the handling of the user interaction. We need three new functions
in the `aws_utils.py`:

{% highlight python %}
from flask import request
import pandas as pd


def get_dataframe():
    """Return the dataframe given by the user"""
    return pd.read_csv(request.stream)


def get_options():
    """Return the options given by the user"""
    return request.args


def print_result(result):
    """Return the result in a form, that can be returned to the user"""
    return result.to_csv(index="_index", header=True)
{% endhighlight %}

The `request` object we import from the `flask` package has some very convenient
functionality for accessing the information passed by the user.
The `args` attribute is some sort of a dictionary, that includes the parameters like `query=hello` passed in the URL in the form `test.com/search?query=hello`
and the `stream` attribute is the payload, that is passed as data (we will see later how this
can be done).

Before adding the methods to our main route, we first add the real data science code. To mock
a real world application, we first import some typical libraries (that are however unused in our case)
and fake some heavy calculation.

We already discussed before, we want to do thins in parallel later. For this, we already prepare
things now. I will start with giving you all the code and walk through it later:

In `df_utils.py`:
{% highlight python %}
import pandas as pd

from zappa_distributor import my_map


def calculate_result(df, options):
    """
    Main function of our application: calculate the sum of each time series for each "id" separately
    and return the dataframe.
    :param df: The input data.
    :param options: The options for the calculation given by the user.
    :return: The output data.
    """
    chunksize = int(options.get("chunksize", 5))
    id_column = options.get("id_column", "id")
    value_column = options.get("value_column", "value")

    grouped_data = df.groupby(id_column)[value_column]
    result = my_map(feature_calculation, grouped_data, chunksize=chunksize)
    result = pd.DataFrame(result).set_index("id", drop=True)

    return result


def feature_calculation(chunk):
    """
    Here, the actual calculation of the sum happens for a chunk consisting of
    the id and the time series of the data.
    :param chunk: The id and the time series the sum will be calculated for.
    :return: A dict with the id and the result of the calculation.
    """
    series_id, timeseries = chunk
    return {"id": series_id, "result": timeseries.sum()}
{% endhighlight %}

I think this part can already be understood without knowing, how the `my_map`
function works exactly. The `calculate_result` function will be our main
function of the module. We group the data according to the column `id` (which name
can be given by the user) and extract only the `value` column (again, given by
the user). For each chunk of data (the `id` and the time series of values
with this `id` - the `values`), we will run the `feature_calculation`
function once, which will return a dictionary in the form

{% highlight python %}
    {"id": id_of_the_chunk, "result": result_of_the_calculation}
{% endhighlight %}

If you fill in time series with 1000 ids, the `feature_calculation`
will be called 1000 times. The dictionaries can be gather together in a
list and transformed into a `pandas.DataFrame` (the last half of the `calculate_result`
function). We can then return the data back to the caller.

The `feature_calculation` will later run in parallel in another Lambda function.

The `my_map` function will later handle the invocation of the parallel Lambda functions.
Until now, we will only use a normal pandas `map` here - but we will already
include some "twist": as we will not start one Lambda function for every chunk of data
later (that would mean we start 1000 parallel Lambda functions, which would give us
a huge overhead because of streaming), we will summarize some chunks together
(dependent on the chunk size parameter) and only start a smaller number of parallel
Lambdas.

This chunk size handling can already be implemented, although we are using
a sequential ansatz. The idea is: we will start walking through the grouped data chunks
(remember, one chunk means all time series data with the same `id`), stop every
`chunksize` items, wrap these elements up and send them to the calculation Lambda. There,
the list of chunks is unwrapped, the `feature_calculation` is
calculated for each chunk in the chunk list and the wrapped result is send back to the
calling function. On code, this may look something like this (in `zappa_distributor.py`):

{% highlight python %}
import itertools
from functools import partial


def my_map(f, data, chunksize):
    """
    Own implementation of the python "map" function looping
    over the data and calculating f on each of the items.
    Returns a list of the results calculated by each of the
    single calculations.
    :param f: The function to calculate.
    :param data: The data to loop over.
    :param chunksize: The chunksize the data is chunked into before doing the calculation.
    :return: A list of the results from every calculation.
    """
    partitioned_chunks = partition(data, chunk_size=chunksize)
    map_function = partial(feature_calculation_on_chunks, f=f)

    result = map(map_function, partitioned_chunks)
    reduced_result = list(itertools.chain.from_iterable(result))

    return reduced_result


def partition(data, chunk_size):
    """
    Helper function to chunk a list of data items with the given chunk size.
    This is done with the help of some iterator tools.
    :param data: The data to chunk.
    :param chunk_size: The size of one chunk. The last chunk may be smaller.
    :return: A generator producing the chunks of data.
    """
    # Create a generator out of the input list
    iterable = iter(data)
    while True:
        next_chunk = list(itertools.islice(iterable, chunk_size))
        if not next_chunk:
            return

        yield next_chunk


def feature_calculation_on_chunks(chunk_list, f):
    """
    Helper function to make the partitioning undone: loop over all chunks in the
    chunk list and calculate the features for each. Return a list of all results.
    :param chunk_list: The list of chunks to calculate for.
    :return: A list of results.
    """
    results = [f(chunk) for chunk in chunk_list]
    return results
{% endhighlight %}

You see, I am still using the default `map` function here for running over the chunks - this will change
later.

Now we have everything in place to start our first test! Enable the calculation by using all our written tools
in the `app.py`:

{% highlight python %}
from __future__ import print_function
from time import time

from amazon_lambda_datascience import aws_utils, df_utils


def add_routes(app):
    """Add all routes to the application"""
    @app.route('/', methods=['PUT'])
    def main():
        """
        Main method: calculate the features of the given csv dataframe.
        """
        df = aws_utils.get_dataframe()
        options = aws_utils.get_options()

        start = time()
        result = df_utils.calculate_result(df, options)
        end = time()

        print("Calculation needed", end - start, "seconds.")

        return aws_utils.print_result(result)

{% endhighlight %}

Note that we are now using PUT instead of GET, as we want to transport some data.
And we can already run

    python main.py

to start our server locally and test it (the server will run on [http://localhost:5000](http://localhost:5000)).
When you visit the server, you will get greeted by the lovely error message

    Method Not Allowed

What is wrong? Well, we have changed the method from GET to PUT and our normal browser
 will not help us here :smile:
So lets write a small script that
passes the test data (stored in `data.csv` on disk) to our web service and write back the result to the screen

{% highlight python %}
from __future__ import print_function
import requests
import pandas as pd
from io import BytesIO, StringIO
from time import time

if __name__ == '__main__':

    df = pd.read_csv("data.csv")
    start = time()
    url = "http://localhost:5000"
    with BytesIO() as stream:
        df.to_csv(stream)
        stream.seek(0)
        answer = requests.put(url, data=stream,
                              params={"chunksize": "10", "id_column": "id", "value_column": "value"})

    end = time()
    s = StringIO()
    s.write(answer.text)
    s.seek(0)
    result_df = pd.read_csv(s, index_col="id")
    s.close()

    print(result_df.head())

    print("The calculation took", end - start, "seconds")
{% endhighlight %}

The output should be something like

               result
    id
    0   44.675536
    1   45.564449
    2   54.431959
    3   46.572613
    4   47.840048
    The calculation took 0.363618850708 seconds

Very good! We have a first working implementation of a data science application. We just have to upload things into the Amazon cloud.

## Upload our application into the amazon cloud

After you are finished implementing your framework, you can now upload it into the cloud with a simple

    zappa update

However, you will probably run into an error message saying something like

    Unzipped size must be smaller than 262144000 bytes

The problem is already described in the error message: the maximum size of you zip, that packs the virtual environment,
your flask code and the `zappa` handler is 250 MB - however the dependencies of pandas, numpy etc. make it larger.

Fortunately, the `zappa` team has already provided a solution for this: just enable the `slim_handler` in your json

{% highlight json %}
{
    "dev": {
        ...
        "slim_handler": true
        ...
    }
}
{% endhighlight %}

and run `zappa update` again. This will create two zip files:
* one which only included the `zappa` handler and is loaded directly
* a second one with all the rest (your dependencies, your own code) and is stored in S3. It is loaded later during the first invocation
  into your lambda function.

With that, you are ready to go and by changing the URL in your test script, you can now upload your data
and get the calculated features in return.

Two things to note:
* the first invocation may be really slow. Remember the cold and warm container thing from my [last post](../amazon-lambda-and-map/)?
* Also the later invocation may not be as fast as on your local machine. This is mainly because of streaming issues. We will discuss
  performance in a later post.


### Python 2 - Python 3

Well, this is a bit embarrassing: if you are using python 2, you probably run into the SSL problem
mentioned in the post before: python 2 can not handle https correctly, so when changing the URL in your
test script, the request will fail. But if you are using python 3, the script itself will fail because of
`byte`/`str` things. Life is hard :-(

Ok, two possibilities here: you use python 3 to invoke your script, but then you need a slightly updated version of it:

{% highlight python %}
from __future__ import print_function
import requests
import pandas as pd
from io import BytesIO, StringIO
from time import time

if __name__ == '__main__':

    df = pd.read_csv("data.csv")
    start = time()
    url = "http://localhost:5000" # fill in your AWS URL here (do not forget the dev)
    with StringIO() as stream:
        df.to_csv(stream)
        stream.seek(0)

        with BytesIO() as bstream:
            bstream.write(stream.read().encode())
            bstream.seek(0)
            answer = requests.put(url, data=bstream,
                                  params={"chunksize": "10", "id_column": "id", "value_column": "value"})

    end = time()
    s = StringIO()
    s.write(answer.text)
    s.seek(0)
    result_df = pd.read_csv(s, index_col="id")
    s.close()

    print(result_df.head())

    print("The calculation took", end - start, "seconds")
{% endhighlight %}

I have added this script as `test3.py` in the github repository.

Or you use Python 3.6 right from the beginning, which is supported by Amazon Lambda.

## What to do next?
We will finally add the Lambda invocation in the next post - which is quite easy now as we have everything around. We will
discuss some implications of running Lambda functions, and do some performance studies later.

## Links
* <https://pandas.pydata.org/pandas-docs/stable/>
* <http://flask.pocoo.org/>
