0:00
In this lesson, you will learn how to evaluate your cache
0:02
using metrics like hit rate, precision, recall, and latency
0:04
to understand its real impact. All right, let's go.
0:08
So the two ways that our cache can fail,
0:10
it can either have a Low Quality or a Poor Performance.
0:13
And the way that we're going to measure our cache performance
0:16
is similar to how we measure the performance of machine learning models.
0:20
In this lab, we're going to use this data set
0:22
where we have user queries,
0:24
their corresponding cache hits or the closest match we have,
0:28
the distance to that cache hit,
0:29
and we also have a label column,
0:31
which tells us whether this cache hit was true or not.
0:35
This last column can be obtained either by using human feedback
0:39
or labeling it with LLM as a judge.
0:42
In this slide here, we can see
0:43
diagrammatically what happens with the data set.
0:45
We have examples of the cache entries
0:47
and they are mapped in the embedding space.
0:50
Here we have a point in the embedding space.
0:52
and we also see the distance threshold.
0:55
These two user queries are mapped to the specific cache entry.
0:59
This metric measures from all of the
1:01
user queries that come to our system,
1:03
how many of those actually fall
1:05
within the bounds of the distance threshold.
1:08
In this case, we have three out of five.
1:10
Next, we'll continue with Precision.
1:12
And Precision measures how much of all of the entries
1:16
that fall within the Distance threshold are actually valid.
1:19
And this validity comes from the label column.
1:22
Recall measures how many of the
1:24
queries that should have hit actually hit.
1:26
And here in this case, we see that
1:29
one of the questions that should have hit
1:31
did not because of a too low of a distance threshold.
1:34
It is also important to see where each example falls
1:38
in our confusion matrix, which shows us all the true positives,
1:42
true negatives, false positives, and false negatives.
1:45
In the specific case of our example, we have one true negative
1:49
that doesn't map to anything.
1:52
We have one False Positive, which maps but it's wrong.
1:56
We have one False Negative that should have mapped,
1:58
but just because of the threshold was too low it didn't,
2:01
and two True Positives. And these are
2:03
all the values of our confusion matrix.
2:05
We prefer for the confusion matrix values
2:08
to be mainly on the main diagonal.
2:10
The distance threshold can help us trade off between Precision and Recall.
2:14
As we lower the distance threshold, we
2:17
can increase Precision but we lose Recall.
2:19
And as we increase it, the reverse
2:21
happen. We increase the recall of the model
2:23
and decrease the precision. And the measure that
2:26
we can use to strike a good balance
2:29
between precision and recall is the F1 score.
2:32
We'll use this technique of sweeping through the threshold
2:34
to optimize and find the best threshold for our data.
2:38
And in the next lesson, we're going to use the F1 score
2:41
to find the perfect threshold for our cache.
2:43
It is important also to measure the
2:45
speedup that the caching system can give us
2:47
and the LLM tokens that it can save us.
2:50
To measure the latency performance, we can use this formula,
2:53
which measures a metric called With Cache Latency,
2:56
which breaks down into three different variables. ACL or Average Cache Latency.
3:00
which is the average time
3:03
that our cache takes to respond.
3:06
ALL or average LLM latency,
3:08
which is how long the LLM takes to respond,
3:10
and CHR or cache hit ratio,
3:13
which is how many of the user queries actually get cache hits.
3:17
And let's look at a specific example
3:19
where the average cache latency would be 11 milliseconds,
3:22
the average LLM latency is 350 milliseconds,
3:26
which is probably an underestimate.
3:28
and the cache hit ratio is 30%.
3:31
We calculate that the with cache latency is 256 milliseconds.
3:36
which you can compare to the system's latency before being cached,
3:40
and we see the improvement.
3:42
which in this case is 26%.
3:45
And to measure the saved LLM tokens, you
3:47
can use the resource at the corresponding URL,
3:50
where you can put your cache hit rate
3:52
and expected daily queries, input and output tokens.
3:55
and it will calculate for you
3:57
an expected measure of the annual savings.
4:00
And now let's do all the cache performance measures in the code.
4:04
All right, let's start by setting up our environment.
4:10
And let's continue with loading our data that we're already familiar with.
4:14
Then we'll introduce the SemanticCacheWrapper,
4:17
that wraps around our cache and provides us some helper functions
4:21
uh that we can use to
4:22
hydrate our cache or check our cache.
4:25
and so forth. Then using the abstraction,
4:27
we can hydrate our cache using this simple helper function,
4:30
and you can also check it with questions
4:33
like this first instance of a user query that we have
4:36
in our FAQ data frame.
4:39
When we check our cache, we get
4:41
an object which we see nicely formatted here,
4:43
which shows us what our query was and all of our matches.
4:48
In this case, we have a single match.
4:51
We have already defined this test_queries list,
4:53
which contains all of the user
4:55
queries that we're going to be using.
4:56
We can also introduce the check_many helper function,
5:00
which can given a list of user
5:02
queries, give us results of the matches.
5:05
We can also configure it with
5:07
different thresholds or different number of matches.
5:11
Okay, moving on,
5:13
we can now also use an abstraction called the CacheEvaluator.
5:17
I've already executed this cell,
5:19
but as you can see, you
5:21
can provide it a number of cache_results.
5:23
And because our data set is set up in a particular way,
5:26
we're actually able to automatically label them
5:29
because the data set was automatically generated.
5:31
We can use the data_container that provides the data
5:34
to provide also labels for the data.
5:37
Because the data_container can extract the queries
5:40
and the matches,
5:42
and because we already know the
5:44
correspondences because of how the data was
5:46
generated, we can generate labels automatically.
5:50
Running the report_metrics function
5:52
also prints us this nice-looking report
5:54
where we'll see the confusion matrix.
5:56
In this case, we have a rather nice-looking confusion matrix
6:00
and all of the metrics for the particular threshold
6:03
that the cache was instantiated with.
6:05
In this case, we get a precision of 0.79
6:09
and recall of 0.79.
6:11
Another very useful high-level helper function is called get_metrics.
6:16
This function gives us access to all
6:18
of these metrics in a dictionary format,
6:20
but it also gives us something called the confusion_mask.
6:23
The confusion mask is similar to the confusion matrix,
6:26
but instead of giving you the values of the
6:29
confusion matrix as numbers.
6:31
It gives you a mask over the data set,
6:34
which you can use and extract the different values
6:37
corresponding to different categories of the confusion matrix.
6:40
In this example, we're seeing the first nine examples
6:44
of the true negatives group.
6:47
In this case, we see that the
6:49
0, 1, 2, and the third element
6:51
is a part of the true negatives group.
6:53
Another very useful method of the evaluator
6:57
is the matches data frame, matches_df. If we run it,
6:59
we get a nicely looking data frame
7:01
that gives us our queries, matches,
7:04
distance to the corresponding match, and labeling.
7:08
We can use our confusion masks and filter out
7:11
and get the corresponding groups.
7:13
In this case, we're looking at the false positives
7:15
or these five examples here of this particular example.
7:19
In this specific instance, we see that the query,
7:21
Can I get a refund if I change my mind?
7:23
is mapped to the cache entry, How do I get a refund?
7:27
It's obvious why the retrieval model got confused,
7:31
but because we generated our dataset in a particular way,
7:35
having in mind that this query
7:37
is not particular enough to be matched
7:39
to this cache entry. We have generated
7:42
the label to be false. And this so that
7:45
because of that, we're considering this example as a false positive.
7:49
So for instance, let's look at this last example in the table.
7:52
Can I schedule a specific delivery time?
7:55
is matched against Can I change
7:57
my delivery address? As you can see,
7:59
These are quite different, even though they sound similar.
8:03
The model has given them low enough distance
8:06
so that we've considered them as a match,
8:08
even though we've labeled them as
8:10
though they should be a false match.
8:12
So that's why this example falls into the bucket of false positives.
8:16
I would strongly encourage you to change this false positives modifier here
8:20
or a mask
8:22
to all of the other values like
8:24
false positives, false negatives, and true positives,
8:27
all of the other groups of the confusion matrix
8:29
so that you can explore the results there.
8:32
Now let's introduce helper functions
8:34
for evaluating the latency performance of our cache.
8:38
We'll define a simple function just
8:41
that would simulate an LLM response latency
8:44
by randomly sleeping for a period of
8:48
200 to 500 milliseconds.
8:51
Then we'll run this performance code,
8:54
which uses an abstraction called PerfEval.
8:57
which we can use to compare the performance
9:00
of two different executions simultaneously.
9:05
At the end of the execution, we'll obtain a dictionary called metrics,
9:10
which is generated by the perf_eval abstraction, which will give us
9:13
the latency both for the cache and the LLM calls.
9:17
If we look into the dictionary, we can
9:20
see that we can obtain the average latency
9:22
of the cache, which is 2.2 milliseconds,
9:25
and the average latency of the simulated LLM,
9:28
which is 361 milliseconds.
9:32
Using the perf_eval, we can also plot
9:34
all of the metrics that we have obtained.
9:37
Here, we see a breakdown
9:39
of all of the different groups of evaluations that we did.
9:42
We had 80 cache hits and 80 LLM calls.
9:45
And here we can compare their latency in a visual manner.
9:50
So this is how a 360 milliseconds on average looks like,
9:55
compared to the 2 point something milliseconds for the cache.
10:00
In the summary block here, we can
10:02
also see the cache speedup that we get,
10:04
which in this case is 161 times faster.
10:08
So what we saw now is a raw LLM latency
10:11
versus full cache latency.
10:14
which gave us this big speed
10:15
up, but it's not a fair comparison.
10:17
What we should actually do is get our raw LLM latency
10:21
and compare it. This is the cache latency.
10:24
We can assume some cache hit rate, in this case 30%.
10:27
and we can calculate using the formula from the slide,
10:30
the cached LLM latency that we can expect.
10:34
Now using that, we can calculate both
10:37
a drop in the latency
10:38
and a cached LLM speedup.
10:40
And here, for this particular example,
10:43
you see that we see 29% drop in latency.
10:47
and around 1.42x speed up of the system.
10:51
Let's move to our last section here, which is LLM-as-a-Judge,
10:55
where we were going to introduce an
10:57
automatic way to label your query cache pairs.
11:00
We're going to start over by hydrating our cache with the faq data frame.
11:05
We're going to do a full retrieval.
11:07
So we're going to use a distance_threshold of 1,
11:10
and for each test query, we're going to retrieve the closest neighbor.
11:14
no matter the threshold.
11:16
And what we get is a
11:18
list of all of the closest matches
11:20
for all of the queries.
11:21
Now since we're going to use an
11:23
LLM to label all the query cache pairs,
11:25
we have to load our key, which is already loaded.
11:30
Then we can use an abstraction called LLMEvaluator,
11:34
which here we will construct using the ChatGPT API.
11:38
It already comes with pre-configured prompt.
11:41
which help us compare pairs of queries and cache hits.
11:45
This evaluator instance has a method called predict,
11:48
which you can use to pass in our pairs,
11:51
which are the test queries and the full retrieval matches.
11:54
We can also configure a batch size.
12:01
So this call to predict gives us an object
12:04
called llm_similarity_results.
12:06
Let's inspect it.
12:10
This object has a property called data frame df,
12:13
which formats the results in a data frame format.
12:16
We can see that this is a data frame
12:19
that matches every pair with a reason and a is_similar label,
12:23
which tells us if this pair is similar or not
12:27
and the reasoning why the LLM thought that was the case.
12:30
Now we can continue and use our evaluator again,
12:33
but instead of using the data container
12:35
to label our data set.
12:38
we can use the automatic labeling that we generated here
12:42
by passing the is_similar column
12:44
and all the values as a boolean labeling.
12:48
It is important to note that we have to construct the evaluator
12:52
with this special constructor called from_full_retrieval,
12:55
which is important when we're using the full retrieval matches.
12:58
We can again use the evaluator.report_metrics function to get a report.
13:03
of the performance. As we can see,
13:05
previously we had 79% precision
13:08
and 79% recall.
13:10
Using the automatic labeling, we get 75 precision versus 90 recall,
13:15
which is relatively close in terms
13:17
of metrics to what we had previously.
13:19
which tells us that this method can
13:22
create good enough labels for our evaluation.
13:24
And now that you've learned how to evaluate your cache,
13:27
in the next lesson, you're going to learn how to improve it.
13:30
Now let's clean the cache and continue to the next lesson.