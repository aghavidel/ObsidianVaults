# Improving Map-Reduce Performance

So far, we have seen:
- Distributed system for storage (CFS)
- Distributed system for compute or bandwidth aggregation (SNS)

What if **both** of them are the bottleneck? What if the data is extremely large in that it cannot be put on a single machine, but the computation cannot be done on a single machine as well? This is where things like getting like-dislike ratio for a whole fleet of YouTube videos becomes a thing.

## Background: Partition-Aggregate

One simple way to cope with computation on huge amounts of data, is the partition-aggregate scheme:
- Chop data into chunks and give it to different machines
- Let each machine finish processing
- Some how join the data on an aggregator node and output the result

![[Pasted image 20240415112840.png|500]]

Simple! Not that useful in its base form though.
There are problems here:
- Aggregator needs to be replicated, or else it is a single point of failure.
- If one of the partitions slow down, the whole thing slows down as well.
- What if the aggregator output itself is so massive it won't even fit on it? (take sorting a huge array of data), the output is as large as the input.

To address all of these challenges, **MapReduce** was proposed.

## MapReduce Intro.

Whatever input you have, make it into a set of key-value pairs. So for example if you have a massive web crawl output, key should be the URL and the value be the HTML.
We then **partition** the data into chunks:

![[Pasted image 20240415113603.png|400]]

Then, the programmer defines a **Map** function. This function will receive a key-value pair and output some other key-value pair.

Then, the pipeline will Coalesce the map output, it aggregates all data with the same key into the same machine. Finally, the reduce step will also receive a function from the user that output a single key and some value that is a function of all the values from that key.

>[!EXAMPLE] Word-Count
>You have a series of files, and you want to count the words in all of them. You map the name to the file contents, and then create a map that maps a word to `1`.
>Upon the coalesce state, we get count of each word, and then reduce them by summing the values up to get the count of all of them.

>[!EXAMPLE] PageRank
>PageRank is Google's method for ranking pages in a query. Assume we initialize all pages to a random value for their rank. We want to update it such that the rank of a page is the average rank of pages that link to it.
>The map would be like this. For every page $P$ that $W$ links to, output $(P, W)$. In the reduce state, you will group the tuples with the same key and average the rank of the second element.
>You are not done though! This is iterative, since the ranks are now changed. You must repeatedly do this until the ranks converge.

In practice though, one problem with MapReduce is that again, it gets bottlenecked with the slowest tasks:

![[Pasted image 20240415115050.png|500]]

These slow tasks, called *Stragglers*, will delay the transition from Map to Reduce. The reason is that you cannot do reduce unless all map steps are completed. MapReduce is essentially **Barriered** on the coalesce step right before reduce.
Also, if the output of a map is lost, then we need wait for it catch up as well.

>[!FAQ] Why Such Disparity Between Maps or Reduces?
>The speed of each step depends on the machine that it is running on. Note that a machine can have different hardware, and very large data center fleets are heterogenous. It is also possible that hardware could be faulty for some machine.
>
>The bigger thing though is that machines run other processes and it is possible that some of them are running under contention.

The above observation implies that we need to find a way to detect and cope with slow nodes.

## Improving MapReduce Performance

We need to detect stragglers, and reduce the runtime. To do this, we can turn to our old friend, *Speculative Execution*. How?

Well, let us go back to our previous example above. Map 2 is a straggler, and we can speculate that it is the moment that we notice that Maps 1, 3 and 4 are finished but 2 is not. With this, we can start to do other things. 
If a machine is idle, then we spawn a copy of Map 2 on it in the hope that it is done sooner since its previous Map executed pretty fast. The same also applies for Reduce tasks as well.

![[Pasted image 20240415120206.png|500]]

To implement this, each worker periodically syncs with a master node that keeps track of each worker execution. 

![[Pasted image 20240415120359.png|500]]

If needed, the master will run a copy of another workers job on a faster worker to speed things up. The master must:
- Constantly monitor the execution of other workers and compare the execution of workers with each other to see who can be a straggler.
- It should have a policy to estimate the finish time and also not be too greedy with spawning copies. It should only do that if it suspects that the straggler could *significantly* slow down the process.

There is a big question though, how to estimate the execution time anyway? It is simple to assume rate of progress is linear, but is that true?
Well no, the workload itself can be heterogenous, which means that different inputs can take very different amounts of time. If there are heavy computations, then just a mathematical function can have very different execution speed (take exponents for example). 
Things become much more complicated if at any point of a process, IO gets involved. If we need to copy something, then things can quickly become unpredictable.

>[!FAQ] Why Not Run 2 Copies of Each Task?
>We are kind of assuming stragglers are rare, so why not run 2 of them?
>Well, it's just not efficient, and if you speculation of something being slow turned out to be wrong, you would look very stupid!

The scheduler developed in this paper is called the LATE scheduler and its main principles can be summarized as:
1. Identify stragglers early, so do it as soon as you notice something is an outlier.
2. Prioritize execution based on finish times
3. Run tasks on fast nodes, and identify fast nodes based on previous executions
4. Put a high cap on the total number of task copies

