---
title: "Why you should use Amazon Lambda for your data science application"
layout: post
date: 2017-08-13 22:00
tag: [datascience, python, lambda]
image: /assets/images/lambda.png
headerImage: true
projects: false
hidden: false
description: "The benefits of Amazon Lambda for data science applications"
category: 'blog'
author: nils
externalLink: false
---
Have you ever thought about writing your own data science application/platform in the cloud?
But you do not know where to start and with which technologies? This post will talk about Amazon
Lambda and why it will help you develop data science applications quickly. 

This is part one on my series on Amazon Lambda for Data Science.
To see how this all comes together, we will develop a real-world data science applications
in this blog post series over the next time.

* [Introduction: Why you should use Amazon Lambda](../why-you-should-use-amazon-lambda)
* [Part 2: Simple map framework for Amazon Lambda using zappa](../amazon-lambda-and-map/)
* [Part 3: Data Science with Amazon Lambda](../amazon-lambda-and-map-2/)
* [Part 4: Use the Power of Lambda](../amazon-lambda-and-map-3/)

## What is Amazon Lambda?
Never heard of Amazon Lambda? [Amazon Lambda][1] is part of Amazon's large cloud computing platform and in my
opinion one of the most sophisticated ones. It is called *Function as a Service* or *serverless* framework.
It gives you the opportunity to run every function you like (written in Python, node.js, C+ or Java)
without thinking about server infrastructure, security updates, network or storage etc. It is really
a hassle free platform for running your applications in the cloud. And as in most of Amazons cloud
computing services, you only pay what you use: in this case even as precise as 100 ms runtime.
Plus: you will have a lot of free contingent with what you can play around - and scaling up is
a no-brainer, as Amazon handles the load balancing and availability for you.

The simple setup together with the tight integration into the rest of the Amazon cloud services makes development
of services, that take some input (e.g. from Amazon S3), do some calculations and write back (e.g. to one of Amazon's
database services) very easy.

You data science application may start with an API call from the user, which is handled by the Lambda function.
It first checks if the user is valid in a connected databasem does the balancing part (maybe your data
science application is not for free?) and reads in the data (maybe from S3).
Then it spawns worker lambda functions to do the calculation, collects the results and hands back the data to the user.


## Why not use "normal" parallelization, e.g. with EC2?
If you have never heard about Amazon Lambda, your first intention would maybe be to build such an application
using a "normal" virtual machine/docker container (or even a bare metal server) - like [Amazons EC2][2]. You
would install your python environment there, would add some amount of RAM and CPU cores, add some load balancing
and connect it to the API framework. This in itself is some work to do, which you would not have with Amazon Lambdas - 
but there are plenty of tutorials in the net - so let's assume this is no problem. Additionally, you would have to
take care on how to trigger your calculation running on your VM on your own - but let's also assume this is no
problem for you (you are smart).

Still, there is another reason,why Lambdas might be more compelling to you: cost.

Expecting your new data science application will not ring the bell from the start on and you are still in some sort of
try-out-phase, we might say you have something like 1000 users every day (which is quite large already, I would say).

So let's calculate the expected costs for the VM and the Lambda scenario.

**The Lambda scenario:** For this we have to approximate the runtime for every execution and the number of needed
Lambda functions ( = cores). Just assume we need 5 s and 10 Lambda functions for every user request. This means we have
10 cores for 5 seconds - and as we probably need a lot of RAM, lets use the highest one for each Lambda function, which 
is 1536 MB, summing up to 15 GB in total. Neglecting the really small S3 storage and transfer costs, we end up with

      10 (# Lambda functions) * 1000 (number of requests) *
      (0.2 / 1e6 (price per request) + 0.000002501 (price for 100 ms, 1.5 GB RAM) * 50 (5 s in units of 100 ms))
    = 1.25 $ per day.
    
Here, we do not take into account the 1 million free request and the 400.000 GB-seconds per month that are included in
your free tier (if you only have those 1000 requests per day, you end up with `30 * 1000 * 5 * 1.5 = 225.000` GB-seconds
per month, you so end up totally free!)

**The VM scenario:** As we do not want our costumers to wait approx. 1 minute until our VM starts when he does a request 
to our application, we would have to let the VM run all day. Maybe we know when exactly we will have a peak in the number 
of requests, so we might be able to spawn different numbers of VMs for different hours of the day - but we would definitely
need one VM for the whole time. We will take one of the medium-sized machines for the "all-day-round-workload" and 
a bigger one for only 2 hours a day:

      0.054 (t2.medium price per hour) * 24 + 0.216 (t2.xlarge price per hour) * 2
    = 1.73 $ per day
    
We are using the prices for on-demand EC2 instances. You definitely end up with less using the reserved or spot instances,
but this not fit into our idea of small, quickly evolving data science application.

We have also not taken into account the free tier in this calculation (750 hours per month), but as these is only
valid for t2.micro - and we probably need at least a 2.medium, this is useless here - in contrast to the lambda scenario.

As you see, the costs for a day with Lambda is smaller than for a day with EC2. Of course, this is highly 
oversimplified, but I think that the described scenario is more or less reasonable.

And: we have ended up with 2 cores the whole day and 2 + 4 cores for the spot time of 2 hours in the VM case. If we 
would really need 10 cores for every request (what we have assumed in the Lambda case) to handle it in 5 seconds,
the VM calculation would not only need longer, but would only be able to handle one single request at a time, whereas the
Lambda functions can be scaled up to 1000 automatically (1000 is the default limit for concurrent calculations - you
can request a higher limit if needed).

Additionally, you only pay what you use in the Lambda scenario - so if you have only one request, you only pay
0.00125 $ per day. The "you-only-pay-what-you-use" is also true in the EC2 case - but as you never know when a request 
will come, you have to keep your VM spinning the whole day. 

*Note:* I skipped some subtle thing here: also the Lambda function will go into "standy" if not used, and you may have
a so called "cold" Lambda (think about a docker image, you need to start), if unused. However, the startup time of a
Lambda container is usually a lot smaller compared to the VM start up time and there are very cost-effective and easy
ways, to keep your Lambda warm (e.g. by sending a short heart-beat every 10 minutes, which will only cost you 1e-7 $ per
day).   

## What to do next?
As you have seen, Amazon Lambda has some benefits compared to non-serverless approaches. To use
these benefits, we will create a small data science application in the next blog post, which
we will use as play-ground for several techniques needed in this context (stream- and
batch processing, handle API calls and large python environments, database access, automatic payment etc).


## Links
* <https://aws.amazon.com/Lambda/details/?nc1=h_ls>
* <https://aws.amazon.com/ec2/pricing/on-demand/?nc1=h_ls>
* <https://aws.amazon.com/lambda/pricing/?nc1=h_ls>

[1]: https://aws.amazon.com/Lambda/details/?nc1=h_ls
[2]: https://aws.amazon.com/ec2/
