---
title: "Lambda IV: Use the Power of Lambda"
layout: post
date: 2017-11-01 11:00
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
In my [last post](../amazon-lambda-and-map-2/)
in this series, we have developed all the building blogs for our data science application -
we only miss the parallelisation setup.

This is part four on my series on Amazon Lambda for Data Science. To see how this all
comes together, we will develop a real-world data science applications in this blog post series over the
next time.

* [Introduction: Why you should use Amazon Lambda](../why-you-should-use-amazon-lambda)
* [Part 2: Simple map framework for Amazon Lambda using zappa](../amazon-lambda-and-map/)
* [Part 3: Data Science with Amazon Lambda](../amazon-lambda-and-map-2/)
* [Part 4: Use the Power of Lambda](../amazon-lambda-and-map-3/)

You can find the code of this series on my [github account](https://github.com/nils-braun/amazon-lambda-datascience) (will be updated when we go on).
The version of this post is stored in tag *part-four*.

The application is running and working as expected - but where is the benefit of using possibly 1000 parallel invocations
of Amazon Lambda if we run our code sequentially on only one core? Before we go into the details, let me give you some
first remarks on this subject
* In this tutorial I will assume, that your data science code can easily be parallelised for a given data set - in our case
the feature calculation can run independently for different `id`s of the data set. This may not be the case in your use-case.
* The benefit of parallelisation always comes with the drawback of overhead because of process management (or Lambda function
  management in our case) and streaming (which is even worse in our case, as we have to stream over the network to the next
  lambda function instead of only streaming to a different process). Do not underestimate this! We will see this effect
  clearly, when we do some performance studies.
* The current limit for parallel executions of Lambda functions is 1000 invocations. If we assume something like up to 10
  needed Lambda invocations for a given request, we can handle 100 users in parallel before the request needs to be throttled
  down. This is probably enough if you are running a small data science application, but will not be sufficient in large
  scale applications. You can in principle call the Amazon support to increase your limit here, but as discussed in the
  [first post](../why-you-should-use-amazon-lambda/) in this series, it may be beneficial
  to use other possibilities (e.g. Amazon EC2). The good thing: the developed `flask` application runs without any
  problems on EC2 - you just have to change the way the parallelisation is done (e.g. use a multiprocessing `Pool`
  or packages like `distributed`).

After this is settled, lets start!

## One - Two - Many
The important parts of the current version of the `my_map` function looks as follows:
{% highlight python %}
def my_map(f, data, chunksize):
    partitioned_chunks = partition(data, chunk_size=chunksize)
    map_function = partial(feature_calculation_on_chunks, f=f)

    result = map(map_function, partitioned_chunks)
    reduced_result = list(itertools.chain.from_iterable(result))

    return reduced_result
{% endhighlight %}

We will replace the `map` call with the following (pseudo-like code):
{% highlight python %}
def distribute_to_lambda(map_function, iterable_data):
    for each data in iterable_data (run in parallel):
        arguments = map_function and data
        pack arguments to make them streamable
        start new lambda function, that:
            unpacks the arguments
            calls the map_function
            packs the result
        wait for the end of the Lambda call
        unpack the result
     return every result

{% endhighlight %}

We will start all Lambda functions in parallel, because their invocation will block until
they are finished. We are not using an event or callback based workflow
here - although it would be possible (but harder to implement).

We will start by writing the packer and unpacker functions.

### Packing and Unpacking
We will need to transport the results of the calculation from and the input chunk list
to the Lambda function. So we need methods for packing up arbitrary python objects
into a stream of chars, that is transportable in a JSON (because this is how the
Lambdas will communicate). We will make a very strong assumption here: everything
we want to stream over the network will always be in the form `list of dicts of
native python types`. This is quite limiting on our implementation, but if you think
of it, it is exactly what we need in case of data science: everything can be broken down
to lists of numbers and strings.

However, we first have to make sure that our `calculate_result` function really
outputs a list of dictionaries to the `my_map` function. We will do this by including
the following code snippet right below the `groupby`

{% highlight python %}
def convert_to_dict(x):
    series_id, series = x
    return {"id": series_id, "values": list(series.values), "name": series.name}

grouped_data = map(convert_to_dict, grouped_data)
{% endhighlight %}

I am reassigning to the same variable, so you do not need to change the rest of the code,
but you can of course also change this. The function will use the grouped pandas
data frame (which is basically a list in the form `id` and `data`) and turn it into a list of dictionaries, where the data is not a pandas dataframe but a
list of python base types.

We need to refine our `feature_calculation` function on the other side of the pipe also to:

{% highlight python %}
def feature_calculation(chunk):
    timeseries = pd.Series(chunk["values"], name=chunk["name"])
    timeseries_id = chunk["id"]
    return {"id": timeseries_id, "result": timeseries.sum()}
{% endhighlight %}

to use the dictionary instead of the tuple.

Now we can define our streamer functions, e.g. like this (in `zappa_distributor.py`):

{% highlight python %}
import json
import zlib

def encode_payload(payload, compression):
    """
    Encode an arbitrary python object to be transportable in
    JSON objects.
    :param payload: The object to transport.
    :param compression: Turn on compression or not.
    :return: A string with the object representation.
    """
    json_string = json.dumps(payload)

    if compression:
        compressed_string = zlib.compress(json_string)
        return compressed_string
    else:
        return json_string


def decode_payload(compressed_string, compression):
    """
    Decode a string representation of a python object
    back to the object.
    :param compressed_string: The python representation of the object.
    :param compression: Was compression turned on during encoding?
    :return: The python object
    """
    if compression:
        json_string = zlib.decompress(compressed_string)
    else:
        json_string = compressed_string

    return json.loads(json_string)
{% endhighlight %}

Notice that I have already included a flag for compression, so we can later test
if it helps or not. You can also make this compression flag configurable
like the value or id column name.

We can now already start putting everything together - although we still need
a method to invoke another lambda function (but we can cheat and just call
the function itself instead calling it in another lambda instance).

So we add three new functions to the `zappa_distributor.py`:

{% highlight python %}
def distribute_to_lambda(map_function, iterable_data, compression):
    """
    Method comparable to pythons `map` function, which
    calls a given map function on an iterable data set.
    However, the single items of the iterable are all given
    to different lambda functions in parallel.

    The lambdas are invoked using threading.

    The data needs to be in the format list of dictionaries of simple python types.

    :param map_function: The function that should be called on each item
    :param iterable_data: The list of items that is distributed
    :param compression: Turn on compression on streaming the data.
    :return: The list of results for each lambda.
    """
    from multiprocessing.pool import ThreadPool
    pool = ThreadPool()

    prefilled_map = partial(run_lambda, map_function=map_function, compression=compression)
    results = pool.map(prefilled_map, iterable_data)

    return results


def run_lambda(data, map_function, compression):
    """
    Run a given map_function on the given data and return the result.

    For this:
    * the data is encoded (using compression or not),
    * a lambda function is invoked, which decoded the data, calls the map_function and
      encoded the result again
    * the result is decoded again and returned.
    :param data: The data that is sent to the lambda function
    :param map_function: The function that is called in the lambda on the data.
    :param compression: Turn on compression during streaming or not.
    :return: The result of the function call.
    """
    encoded_data = encode_payload(data, compression)
    # TODO: we are cheating a bit here, because we are just calling the function instead of
    # sending it to another lambda
    encoded_result = function_in_lambda(encoded_data, map_function, compression)
    return decode_payload(encoded_result, compression)


def function_in_lambda(encoded_data, map_function, compression):
    """
    Helper function that is actually called in the lambda (instead of the map_function directly),
    because the input as well as the output data must be encoded/decoded.
    :param encoded_data: The encoded data that will be decoded before feeding into the map_function.
    :param map_function: The function that is called on the data.
    :param compression: Turn on compression during streaming.
    :return: The encoded result of the function call.
    """
    data = decode_payload(encoded_data, compression)
    result = map_function(data)
    return encode_payload(result, compression)
{% endhighlight %}

The `distribute_to_lambda` function is our replacement for the map in `my_map` (and you should already
replace it with this). It calls the functions to invoke another lambda on each data separately, but in
parallel (using threading). The `run_lambda` function will (later) invoke a new lambda and execute
the `map_function` on it. As this function should not see the streaming and encoded data, we
wrap it with the `function_in_lambda`.


You can already go and test this setup (either locally or remotely, as you like). You will notice
(especially of you run it locally), that the execution time is already larger than before. This
comes solely from the streaming and packing of data - although we are not streaming over network yet.

#### Short amendment
Before going on, I have noticed that we make our life a lot easier, if we move the
call to the `feature_calculation_on_chunks` into the `function_in_lambda` instead of defining
the `map_function` as partial function (because it is not pickle-able anymore). So we throw away the

{% highlight python %}
map_function = partial(feature_calculation_on_chunks, f=f)
{% endhighlight %}

in the `my_map` function and just rename the argument `f` to `map_function`.
In the `function_in_lambda`, we replace the `map_function` call by

{% highlight python %}
# was result = map_function(data)
result = feature_calculation_on_chunks(data, map_function)
{% endhighlight %}

As these were quite some changes in total, I have added another git tag called *part-four-middle*,
where you can see the status of the code so far.

### Let's invoke ourselves!

It's time we fill in our last missing part! To make this part configurable, we add another option
to `run_lambda` (as well as to `distribute_to_lambda`
and to `my_map`, to make it configurable in `calculate_result`) called `invoke_lambda`.
With this new parameter, our `run_lambda` body looks something like this:

{% highlight python %}
from zappa.async import get_func_task_path, import_and_get_task

from aws_utils import send_to_other_lambda

def run_lambda(data, map_function, compression, invoke_lambda):
    encoded_data = encode_payload(data, compression)
    map_function = get_func_task_path(map_function)
    if not invoke_lambda:
        encoded_result = function_in_lambda(encoded_data, map_function, compression)
    else:
        encoded_result = send_to_other_lambda(function_in_lambda, encoded_data, map_function, compression)
    return decode_payload(encoded_result, compression)
{% endhighlight %}

We still have to define the `send_to_other_lambda` function in aws_utils, but what else has happened?

To explain this, I have to tell you a bit in forehand, on how we will call the lambda function.
We will use the zappa framework for this. We will make an Amazon Lambda API call to the same lambda function
we are running in the moment, which will again
call the zappa handler, that we are already using to
answer out HTTP API calls (remember?).
However, this time we are telling zappa not to execute our
normal flask entry point as usual, but to call a special
function for us, namely the `function_in_lambda`.
For this, we will have to give the arguments to this
function also, which means we have to convert them to JSON before
(Amazon communicates via JSON between Lambdas). But the `map_function` is a function object - we can not easily convert it to JSON.
Or? Well we can convert it to a string quite easily,
because a normal python function can be described by its
full name (with module etc.). And this is exactly what the `get_func_task_path` from the zappa package is doing.

The counterpart in `function_in_lambda` is the `import_and_get_task` function, which turns a name back into a function, which we add at the beginning

{% highlight python %}
map_function = import_and_get_task(map_function)
{% endhighlight %}

But still we have no Lambda caller! Ok, I will just dump the new code for `aws_utils.py` first and we discuss it afterwards:

{% highlight python %}
from zappa.async import get_func_task_path, LambdaAsyncResponse, AsyncException

def send_to_other_lambda(function_in_lambda, *args, **kwargs):
    """
    Call the function_in_lambda in another lambda instead of the running one
    and return the result.
    :param function_in_lambda: The function to call.
    :param args: The arguments to this function.
    :param kwargs: The lwargs to this function.
    :return: The result of this call.
    """
    lambda_function_name = os.environ.get('AWS_LAMBDA_FUNCTION_NAME')
    aws_region = os.environ.get('AWS_REGION')

    task_path = get_func_task_path(function_in_lambda)
    result = LambdaSyncResponse(lambda_function_name=lambda_function_name,
                                aws_region=aws_region).send(task_path, args, kwargs).response

    return result


class LambdaSyncResponse(LambdaAsyncResponse):
    """
    Helper class inheriting from LambdaAsyncResponse to start a new lambda function.
    """
    def _send(self, message):
        """
        Send method overloading the one from LambdaAsyncResponse, to call the lambda
        synchronously. Still mostly copied from the original function.
        """
        message['command'] = 'zappa.async.route_lambda_task'
        payload = json.dumps(message, encoding="utf8").encode('utf-8')
        if len(payload) > 6000000:  # pragma: no cover
            raise AsyncException("Payload too large for sync Lambda call")

        self.response = json.loads(self.client.invoke(
            FunctionName=self.lambda_function_name,
            InvocationType='RequestResponse',
            Payload=payload
        )["Payload"].read())
{% endhighlight %}

First of all you may notice, that we are heavily relying on zappa here - because it brings (mostly) everything
we already need (actually, it has everything for calling a lambda function asynchronously, but we want a synchronous
invocation).

The `send_to_other_lambda` itself is nearly a copy from the run function in `zappa.async`, but using
our own response handler for sending the Lambda: `LambdaSyncResponse`. This class derived from `LambdaAsyncResponse`
uses an Amazon API call (with `boto3`) to invoke a new lambda, giving it the parameters the `zappa` handler needs,
which is:
* the function to call as a string (created with the `get_func_task_path` function)
* the name of the Lambda to invoke (which is our own Lambda, written into this Lambda variable)
* the arguments and keyword arguments of the function call

Everything is nicely packed using JSON and this is it. With this last part, you can now
upload everything to your Amazon account (using `zappa update`) and call your own little
data science application using the test script in the github repository (just remember to change the URL).

Feel free to test using different data samples, different chunk sized and different setting for compression etc.
You can also try to boost the performance using different streaming techniques etc.

## What to do next?

After we are finished with our implementation, we can now now go on and profile/test the whole thing.
I will describe in the next post, how good this scales, what can be done better and if this is fine
for a data science application (or not).

Remember, you can find all code of this post in my [github repo](https://github.com/nils-braun/amazon-lambda-datascience)
under the tag *part-four*.