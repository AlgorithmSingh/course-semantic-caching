0:00
In this lesson, you'll build a working semantic cache from scratch
0:02
so you can see how each part works.
0:04
And then you'll reimplement it using Redis' open source SDK
0:07
and database. All right, let's give it a shot.
0:10
In this lesson, we're going to build a semantic cache from scratch.
0:14
on a customer support use case.
0:16
Customer support is a very common agentic use case
0:19
where agents can help accelerate customer resolution
0:22
to common questions and inquiries and issues.
0:25
The data set we're going to work
0:26
with is a set of frequently asked questions
0:28
from a customer support system and a knowledge
0:30
base of content to support the agentic
0:33
retrieval of different kinds of knowledge. Just as a quick recap,
0:37
semantic caching involves first checking a cache using semantic search
0:41
to see if there's a question that we've processed in
0:44
the past that's similar enough to the current user question.
0:47
If it's similar enough, we can hit the
0:49
cache and we can return directly to the user.
0:51
And if it involves a cache miss, then
0:53
we need to go through our agentic RAG process.
0:55
Ultimately getting a result back to the user and updating our cache.
0:59
All right, in this lesson, we're going to use Redis,
1:02
which stands for Remote Dictionary Server.
1:05
Redis is an open source, fast, in-memory KV database.
1:09
This means that you can store
1:11
different data structures on the Redis server.
1:13
and distribute them across multiple nodes and your applications
1:16
can go read and write and access this data at scale.
1:19
Redis is commonly used for caching, but also
1:22
has the ability to do secondary indexing,
1:24
which means we can store and search across vectors, text,
1:27
numerics, tags, and even geospatial data.
1:30
These properties allow us to do semantic caching in Redis.
1:33
The first thing we'll see when we test with Redis
1:35
is we can optimize our vector search
1:37
for low-latency retrieval and indexing.
1:39
This means we can quickly insert and check the cache
1:42
for questions that are very similar
1:44
to the user query that comes in.
1:46
We'll also use an open source, open weight, fine-tuned embedding model
1:50
that improves the semantic caching accuracy.
1:53
Second, we'll use an open source RedisVL SDK.
1:56
This gives us ergonomic control
1:58
over cache configuration and different CRUD operations.
2:01
And lastly, we can take advantage of TTL or time to live.
2:04
and eviction and namespacing policies that allow us to configure
2:08
how data moves and flows through the cache
2:10
and how tenants can be isolated from
2:12
one another. Let's see this in code.
2:14
So the first step is we're going to load the FAQ Dataset.
2:18
The FAQ dataset comes from a CSV file.
2:22
Here in the system, and this contains frequently asked questions,
2:26
question and answer pairs, and some test data we can use.
2:30
Let's load our FAQs and take
2:32
a look at what they look like.
2:34
You'll notice each entry in our FAQ data set contains a question,
2:37
like how do I get a refund?
2:40
and an answer like to request a refund,
2:42
visit our orders page and select, etc, etc.
2:46
In order to do semantic caching, we need embeddings.
2:49
We will use the SentenceTransformers library in order to do so.
2:53
For our first example, we're going to use the popular
2:55
all-mpnet-base-v2 model.
2:58
We will use this model to encode our FAQ data set
3:01
or list of questions into embeddings.
3:09
The first time you run, you'll need to download the model,
3:12
which may take a couple seconds given your network connection.
3:15
At the end here, you can see
3:17
this is a sample of our first embedding.
3:19
A couple float values to show you what it looks like.
3:22
Next, we will use two functions in
3:24
order to implement semantic search from scratch.
3:28
The first function calculates the cosine distance between two sets of vectors.
3:32
And the second function implements semantic search
3:35
by taking a query, constructing an embedding of the query,
3:39
calculating the cosine distance between the query_embedding
3:42
and our matrix of faq_embeddings.
3:44
And at the end, we find
3:46
the index and value of the best
3:49
entry in our FAQ embeddings matrix.
3:52
At the end we return the index and the distance value.
3:55
Let's try the semantic search function over a
3:57
sample query to see what it comes back with.
4:00
We will run semantic search over the query,
4:02
How long will it take to get a refund for my order?
4:06
So the most similar FAQ in our
4:08
index is how do I get a refund?
4:10
and the cosine distance was 0.331.
4:15
Now we're going to turn our semantic search into a semantic cache.
4:19
We will do this by implementing another helper function called check_cache.
4:22
which takes a query as a string
4:25
and a distance threshold float value.
4:28
First, within our check_cache function,
4:30
we implement semantic_search with the query
4:32
to get the idx index and the distance value.
4:35
If our cosine distance value is less than the stated threshold,
4:39
we treat this as a cache hit
4:42
and return the entries from the cache.
4:44
Otherwise, we return None. This would be a cache miss.
4:47
Let's test our cache over a couple queries.
4:52
So we can see the first query,
4:53
is it possible to get a refund,
4:55
hit on an entry with a distance of 0.262.
4:59
But the other two entries in our test
5:01
queries did not hit on anything in the cache.
5:03
We can also extend our cache with new
5:05
entries as new data comes in over time.
5:08
Let's add a helper function to do that.
5:11
This helper function takes a question and answer pair.
5:14
It adds it to our data
5:16
frame, concatenating with what was already there.
5:18
generates a new embedding and adds this to our faq_embeddings.
5:23
Now, we're going to try updating
5:25
our original cache with three new entries.
5:31
Originally the cache had only eight entries,
5:34
but after all three were added successfully, the cache now has 11.
5:37
This operation worked exactly as we expected.
5:40
Now that we've extended our cache with a few new entries,
5:43
let's run a test to see what hits on the cache now.
5:47
Here we can see all three of
5:49
our test queries now hit the cache
5:50
because we've added new entries to it.
5:53
Now that we've seen how a
5:54
cache works in practice and from scratch,
5:57
let's move towards a more production ready scenario.
6:00
We're going to move to a Redis database instance.
6:03
First we need to connect to a Redis server.
6:06
You can do this through using the REDIS_URL parameter,
6:09
which in most cases is redis://localhost:6379.
6:13
Next, we need to quickly test the connection
6:15
to make sure we can speak to Redis properly.
6:20
We can see our Redis is
6:22
running and accessible from this notebook here.
6:25
The next ingredient is a cache-optimized embedding model,
6:28
langcache-embed-v1.
6:29
This is an open source open weight model available on Hugging Face
6:33
that's been fine-tuned specifically for semantic caching operations.
6:37
Let's load this model using the HFTextVectorizer class.
6:41
This class will reach out to Hugging Face and download
6:44
the weights of the model here on our server.
6:47
Now that we have the embedding model downloaded and ready to go,
6:50
we can create our semantic cache
6:51
using the RedisVL open source SDK.
6:54
Here, we pass a name to our semantic cache
6:57
which creates a unique namespace in the database.
7:00
Second, we pass the LangChain embedding model that we just created.
7:04
Third, we pass our Redis client connection object.
7:08
And lastly, we configure a baseline distance threshold
7:11
to use for the cache checks.
7:14
Let's hydrate our cache with our FAQ data set.
7:17
This will set up Redis with all of that data
7:19
and make it ready for use in our example.
7:23
Here we just iterate through the data frame.
7:25
and entry by entry we store these in the cache.
7:30
Let's quickly check what's in our cache.
7:32
We can ask the question I need a refund for my purchase.
7:39
Let's peek at the result.
7:41
There we can see a cache entry was returned
7:44
with the prompt, How do I get a refund?
7:47
The full validated answer.
7:49
The vector cosine distance here at 0.25,
7:53
which is less than our cache distance threshold of 0.3.
7:56
So this is a proper hit we expected.
7:59
Lastly, before we can test this end to end
8:01
with the large language model, we're going to implement TTL.
8:04
stands for time to live.
8:06
And this is how the database knows when to evict
8:08
data that's been in the cache for a certain period of time.
8:11
It's commonly used to keep the cache fresh
8:13
when data changes or evolves in a system.
8:15
This is really easy with the RedisVL open source SDK.
8:18
You just call set_ttl and we're going to set this
8:21
time to live for a full day, in seconds.
8:26
We're going to conclude this lesson with
8:28
an end-to-end large language model example using OpenAI.
8:31
We'll use the LangChain OpenAI SDK and the ChatOpenAI wrapper
8:35
to connect with gpt-4o-mini.
8:39
The OpenAI keys for this course
8:41
should already be loaded inside your environment.
8:43
To talk with our large language model, we
8:45
have a helper function that sends a prompt.
8:48
In this particular scenario, you are a helpful customer support assistant.
8:52
And this language model is going to
8:54
answer the customer question concisely and professionally.
8:57
that's going to provide a response in one to two sentences
8:59
based on whatever the question is that you give it.
9:02
Now for our end-to-end example, we have a list of test questions.
9:06
We also have a performance evaluation class.
9:09
The performance evaluation class is built as part of the course materials
9:12
and contains the ability to do fine-grained timings
9:16
on cache checks, on LLM calls, on hits and misses.
9:20
This will allow us to compare the
9:22
results after we run this entire test.
9:25
With our performance evaluation class ready to go to take our timings,
9:30
we're going to iterate through each test question, check the cache,
9:33
and if it's not in the
9:34
cache, we'll record that as a miss
9:36
and go to our large language model.
9:40
This loop runs over our list of test
9:42
questions we saw earlier and prints out the results.
9:46
Let's take a look at the results from the test.
9:50
We can see a mix of cache hits and cache misses
9:52
over the sample questions, as expected.
9:57
Now that we finished the test, we can look at the results
10:01
between the cache hits and the cache misses
10:03
that ultimately went to the LLM.
10:06
First, we can see that
10:07
the mean time it took to run a cache hit,
10:10
which is around 65 milliseconds.
10:13
And that in contrast with the LLM, as we would expect.
10:18
took over a second on average.
10:21
This is just one form of measurement.
10:23
In the very next lesson, we'll
10:25
go into depth about measuring cache effectiveness.
10:27
This means we'll focus on all
10:29
of the core metrics between cache accuracy
10:32
and cache performance
10:33
to make sure we have a system
10:35
ready for production and up and running.
10:38
At the very end, you can clear your cache
10:40
to clear the workspace and get ready for the next piece.
