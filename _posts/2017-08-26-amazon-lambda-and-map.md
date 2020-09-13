---
title: "Lambda II: Simple map framework for Amazon Lambda using zappa"
layout: post
date: 2017-08-26 15:00
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
This blog will walk you through the basic setup to develop applications with Amazon
Lambdas and `zappa` - including registration, basic setup and your first website.

This is part two on my series on Amazon Lambda for Data Science. To see how this all
comes together, we will develop a real-world data science applications in this blog post series over the
next time.

* [Introduction: Why you should use Amazon Lambda](../why-you-should-use-amazon-lambda)
* [Part 2: Simple map framework for Amazon Lambda using zappa](../amazon-lambda-and-map/)
* [Part 3: Data Science with Amazon Lambda](../amazon-lambda-and-map-2/)
* [Part 4: Use the Power of Lambda](../amazon-lambda-and-map-3/)

If you have read the [last post](why-you-should-use-amazon-lambda/) in this series,
you are now (hopefully) convinced, that Amazon Lambda helps you develop your data science application
quickly and inexpensively. There is plenty of material and tutorials out there in the web on how to
start with developing. However, because there is plenty of things you can do with Lambdas, it is easy
to get lost. Here, `zappa` comes into the game.

You can find the code of this series on my [github account](https://github.com/nils-braun/amazon-lambda-datascience) (will be updated when we go on).
The version of this post is stored in tag *part-two*.

## How can zappa help?
There have emerged quite some projects around Amazon Lambda in the recent time - which help setting up Lambdas even easier.
One of the most successful (and best in my opinion) is `zappa`. Zappa is a framework especially build for python
web application based on `flask` or `django` and sets up Lambda, S3 and API with the needed execution roles to
run a python web service without you doing anything. You can easily test your `flask` application locally and
then just run

    zappa init
    zappa deploy
    
to have the application running in the cloud. That's it! Whenever you have a new version of your software
you want to upload to the cloud, you can run

    zappa update

`zappa` can even handle different provision targets and lot's of more things we will discuss in the
following posts.

## Let's build a data science application!
We will use `zappa` (and of course Amazon Lambda) to write a simple data science application. 

Our users can upload their data as CSV in a defined format. We will group the data by ID and sum up the columns
in each of these groups (very sophisticated! :smile:), which we will hand back to the user. As we expect very large 
datasets and the sum calculation will be very time-consuming (well, at least we will make it look like this), 
we will do this in a parallel way.

This will take more than one post, so let's dive right into it! We will handle the setup stuff in this post and
talk about our general setup. In the next post(s), we will write the code and do some performance studies.

### Register with Amazon Lambda
After creating a new fresh virtual environment in an empty project folder with

    mkdir amazon-lambda-datascience
    cd amazon-lambda-datascience
    virtualenv venv
    source venv/bin/activate

this is the best time to register with Amazons AWS. If you have never used Amazon AWS before,
you start with a one-year free tier to use many of the AWS features. Basically, it included everything you will need to
follow this blog post series - so chances are good you will never pay for anything. If you have already used Amazon AWS before,
you may still end up with a charge of zero (or very few cents), because we will not use that much of computing power
and as described before, you only pay for what you really use.

Instead of repeating, what is best described elsewhere, I just point you to the [Amazon documentation](http://docs.aws.amazon.com/lambda/latest/dg/setup.html)
on how to create a new AWS account and configure the AWS CLI.
If you do not understand all terms in the manual, do not worry. I will try to explain every concept you will need in this tutorial.

Two things you have to keep in mind when following the tutorial:
* you have already created a virtual environment, so you might want to install `awscli` into this (by calling `pip install awscli` instead of `pip install awscli --user`)
* In the following, I will assume that you have named your profile `default` instead of the `adminuser`, because it makes things a bit easier (although it is not needed) in some configurations.

In the end, make sure that you can run `aws lambda list-functions` successfully.

By the way: I will assume you are running python 2.7 on a linux machine here, as this version is still the most common on ubuntu etc. You can do the whole
tutorial also with python 3.6 - just make sure to not use any other version, as Amazon Lambda is using either 2.7 or 3.6.

## Create a new project with flask
We will start a new project from scratch. As I want to keep things simple, I will use `flask` as a base for
our application instead of `django` or alike.
If you have never heard about `flask`, you can find a very nice introduction [here][https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-i-hello-world].
However, I will try to discuss all needed things in my blog post.
`flask` is a server-side framework for serving html websites. In former times, you might have used `php` for such
a purpose - today you might use `node.js` (but we want to use python of course).
`flask` is very easy to learn, has many nicely-documented add-ons and has everything we will need for our
application. It will run a *server* that answers to HTTP(S) requests with the content we configure
(much like the `SimpleHTTPServer`, much much more advanced and without that much of security flaws...)
`flask` is very convenient for small and simple projects - although it is mature enought to be also used in large applications.

Let's start by installing `flask` and `zappa` into our virtual environment:

    pip install zappa flask


The project structure will be as following:

    amazon-lambda-datascience
      amazon_lambda_datascience
        __init__.py
        app.py
        aws_utils.py
        df_utils.py
        zappa_distributor.py
      main.py

The main entry point to our application will be the `main.py` file, the core `flask` settings will
be handled in `amazon_lambda_datascience/app.py`. `aws_utils`, `df_utils` and `zappa_distributor`
will host helper functions, that we will need in the following.

You do not have to create the files now - we will do so when we go on.

We start with a very simple setup. The basic building block in `flask` is an application, which we will call
`app`. To create it only once, we define in it in the `__init__.py`:

{% highlight python %}
from flask import Flask
app = Flask(__name__)
{% endhighlight %}

Then, we use this application and run it in our `main.py`

{% highlight python %}
from amazon_lambda_datascience import app

def main():
    app.run(debug=True)

if __name__ == '__main__':
    main()
{% endhighlight %}

You can now already try out your first flask application by running

    python main.py

which will host your website on `localhost:5000`. However, as we have not defined any sites, you will not
see anything here :-) (you will end up with a "Not found" error).

So let's add some content! To add a website to our `flask` application, we add so called *route*. A `flask`
route is something like a rule: whenever the user requests a website with an URL as defined in the route,
the function attached to the route is called and the return statement of this function is served as
HTML to the user. That simple!

We will basically just need one single website, where the user uploads the data and gets back the
result, so we can add it as top-level. Let's create a function to add this route to our app in the
`amazon_lambda_datascience/app.py`:

{% highlight python %}
def add_routes(app):
    """Add all routes to the application"""
    @app.route('/', methods=['GET'])
    def main():
        """
        Main method: calculate the features of the given csv dataframe.
        """
        return "Nothing to see here!"
{% endhighlight %}

and call it in the `__init__.py`:

{% highlight python %}
from flask import Flask

from amazon_lambda_datascience.app import add_routes

app = Flask(__name__)
add_routes(app)
{% endhighlight %}

Now, you have already a first website, you can visit on `localhost:5000`, when running `python main.py`!

### Upload your project with zappa
We have a first website, so there is no reason not to upload it to the web and show it around to all of your
friends.

So, lets use `zappa` for the first time now. For this, we have to create a configuration file.
This can easily be done by running (make sure you are in the toplevel folder of our project):

    zappa init

You are greeted with some questions concerning your project, which you can all answer with the default option
(the `zappa` people are really good in finding out, what you probably want).
When finished, just run

    zappa deploy

as told by `zappa` to upload your project into Amazon AWS.

If you are using python 2.7 from the debian repository, chances are very high you are running into an error here:

    SSLError: HTTPSConnectionPool(host='<cour URL>', port=443):
    Max retries exceeded with url: /dev (Caused by SSLError(SSLError(1, '_ssl.c:510: error:14077410:SSL
    routines:SSL23_GET_SERVER_HELLO:sslv3 alert handshake failure'),))

The reason for this is, that `zappa` wants to test if your application is running and needs to do an HTTPS request for this. However, the python 2.7 in the
debian repository has no support for SSL - so this fails. Do not worry! Everything else worked like a charm and your application
is successfully deployed - so just ignore this error. Just note the URL in the error message - you will need it in the next step.

You can now visit your newly created website. The URL is typically a bit cryptical, something like https://zet5xbgsql.execute-api.eu-central-1.amazonaws.com/dev (make sure to add /dev).

Concrats! Your first Amazon Lambda based website!

### What has happened?
Ok, this was easy enough. But what exactly has happened? Zappa has used your aws credentials and the
Amazon AWS SKD, to do several things:
* Package your project with all its dependencies (which is `flask` and `zappa` itself in our case)
  into a zip file
* Add a wrapper around your `flask` application object, which does (in principle) the same thing
  as our `main.py` is doing - with the difference that the `zappa` wrapper is able to perform
  a lot more things than just answering your HTTP(S) request (e.g. logging, testing etc.)
* Upload this zip file into an S3 bucket in your region (and create this S3 bucket if needed)
* Create a new Amazon Lambda function with the name of your project, configure it to use the
  code in the zip-file and start the handler on every lambda invocation.
* Configure Amazons API service to host a new website with the URL you already know from above and to
  call the `zappa` handler (which calls out flask `application`) on every new HTTP(S) request.
* Add a keep warm callback (see below).
* Add execution roles and access roles everywhere, to allow the single pieces to speak with each other.

All those steps can of course also be done on our own - but `zappa` makes those much more convenient.

A word on the keep warm handler: as described in my last post, you can think of Amazon Lambda as something
like a Docker container or a virtual machine, if you are not familiar with this concept (I know, there
are large differences, but I just want to connect to something you may know). Whenever there is a new
request to the URL, the API Gateway invokes our Lambda function (basically the `zappa` handler). Before our
flask application can handle the request, it needs to do some initialisation stuff (e.g. load the libraries),
which is only a small fraction of the time, but may get more when our project grows larger.
We (and they!) do not want to spend time on initialization on every request, so Amazon keeps your Lambda
function "warm" to handle more requests without having to do the initialization again.
However, after a certain amount of time (the real number is not published by Amazon, but it is in the
region of minutes), Amazon shuts down your container to make space for other containers. As we do not
want our Lambda to "cool down" ever, `zappa` adds a scheduled very small call to our function every
4 minutes, which only lasts approximately 4 ms (so 100ms cost), but keeps your Lambda function warm.

## The general structure
Basically, we will build a flask application, that takes the users CSV data as an input, does the calculation
and dumps the result CSV directly as output of the call to our server. As example calculation, we will implement
a simple sum calculation - but basically every calculation will do it here.

The parallelisation will be done by chunking the data into parts - one for every ID and every column with `pandas`.
We will then spawn several additional Lambda functions (so to say: additional cores),
send them the data with the calculation to do and run it. Then, we collect all the results and write it back.

We will first write the general flask application without any parallelisation, then add the Lambda calculations
later on. You will see that the number of spawned lambda functions and the amount of streaming will have a large
impact on the speed of the calculation (as expected).

We will then add some basic functionality needed for your data science application, e.g. a data base for user
information or a small payment system.

There are still a lot more things to do, to create a full usable data science application as a service out of this
(authentication, settings given by the user, real data science worker code) - but I want to keep it simple 
in these posts.

## What to do next?
In my next post, I will handle the real project code with the parallelisation using Lambda functions.

## Links
* <https://aws.amazon.com/lambda/details/?nc1=h_ls>
* <https://github.com/Miserlou/Zappa>
