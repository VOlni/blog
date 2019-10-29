---
layout: post
title: Why we have data pipelines today.
---

####[Under development]

####Plan:

- Why Data Pipelines is a next iteration of streaming/queues/workers? 
- Why streaming/messages data processing are bad today?
- Why data pipelines?
- What next?
 


###Why Data Pipelines is a next iteration of streaming/queues/workers?

I would like to start from core ideas and skip long introduction to 
queues/workers/streaming approach (Hope it will safe your time,  
which you can spent for more important things in your life). 
If you still want to know more about queues and workers check this 
out: [link](https://dzone.com/articles/what-is-a-data-pipeline), 
[link](https://medium.com/100m-io/data-pipeline-programming-principles-5cd1f0e44b38), 
[link](https://www.youtube.com/watch?v=4Spo2QRTz1k)

But let's simplify some terms:

Let's generalize queues/workers/streaming approach into one word "Streaming"
for simplicity (even tho, it's seems like [not the same terms](https://stackoverflow.com/questions/41744506/difference-between-stream-processing-and-message-processing))
<br>And let's call function which process sequence of messages/events as "Worker"


First of all, I truly believe that "real" data pipelines, should be based on 
streaming services (internally), it should be simple, fast and fault tolerant systems
for real-time processing and not only about coping data from one place to another. 
But current solutions like Airflow or Luigi has a complex batch 
schedulers, they hide a lot of complexity around Apache ecosystem and not very useful 
for real time processing.

I will try to explain how Data pipelines as a concept could be a next iteration
for using "Streaming" services and process sequence of events. <br>
Now let's see why we need to make "a next iteration" for "Streaming" services: 

###Why streaming/messages data processing are bad today?


I’m a big fun of "Streaming" approach, but it has some very painful issues
which sometimes really hard to solve in big projects:

- Side effects or global state changes (e.g. writes to database)
- It’s really hard to apply changes (for workers)
- It’s really hard to understand a full picture of how your data moving through your system
- Performance (Even using super fast streaming system, we have network which kind
  of not very performance friendly) 


**Problem 1. Performance**

Let's start from the simple one (at least to explain, not to solve)

Because of the network/file system "Streaming" services has some latency when 
we are moving data between workers. 
This is something which everyone knows about, and that’s why in most cases we 
are using "Streaming" for smth which require some hard CPU calculations or I/O things.

If you are calling database inside your worker, in most cases streaming service will be
able to deliver more jobs which you can process. Maybe because of that this kind
of approach is quite popular today on the web subject, for running background jobs
which handles hard db writes and logic.

And performance it’s not actually a problem if you know, when you should use 
"Streaming" approach.


**Problem 2. Hard to understand a full picture of your system**

I think workers are quite similar to micro-services. But if micro-services is 
something which are not super easy to make (setting up server, docker, API, 
documentation). In case of workers it’s just matter of one decorator, you can
generate hundreds/thousands of them and you not even noticed that. 

The problem of this situation that it’s really hard to understand how your data 
are moving through your system, how its changed, how each of your worker change 
the global state (e.g. database), and what results you will have in the end. 

As a result you will have a very chaotic graph, maybe even with circles. 
Also if you decided to change one worker in the middle of your graph it will 
influence a ton of others parts of your system 
(you will need to check input data for all connected workers)
In the end you will have a big graph of components which you don’t want to change or modify. 


**Problem 3. Hard to apply changes**

When you have a big graph of different components, which are hard to manage and 
work with, you start thinking about fixing that. But it's really, really
hard to do that. 

Image that you decided to change logic in your current worker, and this new 
logic implements new data structure (which then will be passed to next workers). 
If you deploy newly created workers to production you will faced with situation 
when old and new data sitting up in one queue in the same time. And “next workers” 
will raise an exception. In this situation you can not even revert everything 
back, because your old worker will also raise an exception for new data structure. 

Everything which could help you is a “dead letter queue” (like in SQS or RabbitMQ), 
but it’s also not trivial how to process jobs inside this queue if you already have 
a completely new logic in production. 

Right solution, is to create completely new worker function, deploy it, wait 
while old data will be processed and only then remove old worker.

Sometime it’s really hard, because due to the issue N2 your current 
worker depends on a lot of others and you need to relink everything, 
if you forgot something you will always have old data in the queue. BBrrrrr


**Side effects**


Now the most hardest and painful one.

If you heard something about functional programming you will know that 
Side effects (or changing global state) is one of the main enemy for best 
ever code/system in the world. 

In case of "Streaming" approach everything much worse.


Here an example:

```
app = Celery()

@app.task
def super_powerful_worker(input_data):
		analytics = calculate_analytics(input_data)
		
		save_input_data_to_database(input_data)
		next_super_powerful_worker.delay(analytics)
		save_analytics_to_database(analytics)

@app.task
def super_powerful_worker(input_data):
		pass
```

What is the problem here? 
(we just made two commits to database and call another worker, looks very simple) 

BUT, If something happened in the middle, like this:

```
@app.task
def super_powerful_worker(input_data):
		analytics = calculate_analytics(input_data)
		
		save_input_data_to_database(input_data)
		next_super_powerful_worker.delay(analytics)

		raise RuntimeError("eror here, oops")

		save_analytics_to_database(analytics)
```

Your worker will try to repeat this function, what happened next?<br><br>
Yeah, you will try to commit analytics to database once again 
and make another call to a worker (which will duplicate your data) 😱  <br>
Also, note that this worker will try to repeat this function multiple times
and in the end you will have uncontrollable copies of your data and broken db
states :(

What is more interesting if error happens in different places it will be almost 
impossible to fix your database (or identify a problem) because each time you 
will try to commit different data.

Again, that’s why we need “dead letter queue” (like in SQS or RabbitMQ), 
but if you have like a 1000+ failed jobs by e.g. db connection you will
probably just try to run all of them without thoughts about side effects 
or data duplication.

If you think this is something which you can try to fix by global transaction - you can not. <br>
First problem that you need super global transaction (because you can commit to 
database from different functions)<br>
Second problem that Database and "Streaming" service it’s a different systems
and you will have two different transactions, and if smth failed in one of them 
you will have the same situation like above :( 

How to solve this? - Make one db transaction only in a separate worker. 
And call next workers only in the end of "everything".

Example:
```
app = Celery()

@app.task
def super_powerful_worker(input_data):
		analytics = calculate_analytics(input_data)
		
		# only in the end of the function, and we should be
		# 100% sure that no workers called in 
		# calculate_analytics
		with celery.Transaction():
			commit_to_database.delay(analytics, input_data)
			next_super_powerful_worker.delay(analytics)

@app.task
def commit_to_database(analytics, input_data):
		with db.Transaction():
			save_input_data_to_database(input_data)
			save_analytics_to_database(analytics)

@app.task
def super_powerful_worker(input_data):
		pass

```

But yeah, this is not something easy to support in big projects. 

__*And that’s why we have Data pipelines today*__

###Why data pipelines? 

Data pipelines - it’s a way to connect your workers in the right way.  

And based on this term ^ let's presume that you can use data pipelines
for solving almost any tasks you want (!)

And first of all “Data pipelines” has nothing in common with ETL.  
Yes, you can implement ETL paradigm using Data pipelines. 
But it’s just two different concepts. Let’s say ETL it’s like MVC on the web, 
and data pipelines let’s say it’s something like uwsgi, you can’t compare 
MVC and uwsgi, right ? Here the same.

If you look at wiki you will find following: 
Data pipelines - Is a set of data processing elements connected in series, 
where the output of one element is the input of the next one. 

I’m not a big fun of this term, because 
`where the output of one element is the input of the next one.` it’s not a core idea, 
you can call next elements with any data you want and nothing specifically change 
(please correct me in comments if I’m wrong). 

Let’s transform this a little bit:
Data pipelines - Is a set of data processing elements connected in series, 
`where next element called only in the end of/after the current one.` 

BTW, we can say that element == worker function

I think the core idea here is to call all next steps only in the end of the 
current worker. It’s really easy to do using data pipelines concept.  
And current solutions allows you to do that. 

And because of that, we have a lot cool benefits:

**We have a very good picture of all our system**

Because we always knew a “next workers” 
(in most cases we should define all of them in one place). 
As a result we will have a very structured tree without circles and clear vision in 
the code whats going on after the current worker.
(at least it’s a right version of data pipelines)

From wiki: <br>
Flows with one-directional tree and 
[directed acyclic graph](https://en.wikipedia.org/wiki/Directed_acyclic_graph) 
topologies behave similarly to (linear) pipelines – 
the lack of cycles makes them simple – and thus may be loosely referred to as “pipelines”.

It’s really easy to follow all steps which you have in data pipeline and
understand how your data is change when moving through your system and 
you can build a cool tree graph like this:


**Much easy to apply changes**

From previous chapter we know that right way to manage change in "Streaming" service 
is to create completely separate function and route everything to new worker 
(while old one will process old data in the queue) 

With "Streaming" approach it’s not very easy to find all dependencies, 
in case of data pipelines we have all dependencies (next components) 
in one place. 

For framework which implements data pipelines, creating new branch of the tree 
it’s a very trivial task. (Right now it’s maybe not something which easy todo 
using Airflow or Luigi but it should be very easy based on data pipeline terms)


**Side effects**

When we move all workers calls in the end, we can easily wrapped them to one 
transaction in the end of everything. And if something failed in your 
function it will be safe to repeat this function and not duplicate data for next workers. 

BUT. We still have problems with database :( 

And here we solve side effects problem only partially,
But please, don’t cry. It’s possible to solve this completely(!) 

Basically using [ETL concept](http://olegivye.com/why-etl/). 
ETL concept allows you to separate "Transformation" parts and "Load" part and
gently wrapping them in isolated enviroment.  
Because of this separation between data transformations (Transform) and changing global state (Load) we can easily 
solve this problem. 

As a result we will have something like functional programming but 
in distributed calculations world 😍

**From playground to Production**

**New wave of data processing for AI**


### What next?

As you can see data pipelines it’s not only about copy data from one place to 
another, it’s a concepts which allows you to solve or reduce a lot of problems
which you have with "Streaming" services. 

The simplicity of data pipeline structure, allows us to do a lot of interesting
things in a very easy way. 

BUT, the problem that current solutions it’s not about “simplicity” 
or easy way to solve a problems. 

For example: Airflow and Luigi (as pioneers in this field) is very integrated 
with Spark ecosystem. In some way it’s looks like a high level abstraction 
around complex ecosystem which just hides a lot of problems under the hood. 
Also you have complex schedulers, it’s not easy to change pipelines components,
and this is not something what you want to use for kick starting e.g. 
kaggle project :(

If you go to the Ruby or Python main page you will find something like that:

- “Ruby is a dynamic, open source programming language with a focus on simplicity and productivity.”
- “Python is a programming language that lets you work quickly and integrate systems more effectively”. 

This is not something which related to current data pipelines solutions. 

That’s why year ago I started project called [Stairs](http://stairspy.com/). 
Which is a very simple and efficient solution around streaming services 
and written fully on python. The main goal of Stairs is to solve a lot of different 
tasks as simple as possible using all power of Streaming + data pipelines + ETL concept. 

BTW, Check out next article, of how ETL concept + data pipelines is an 
ultimatum solution for streaming systems. 

