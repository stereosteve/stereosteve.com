---
title: "Minhash"
date: 2023-02-25T05:50:07-05:00
Summary: "Quickly approximate set similarity!"
---

[Minhash](https://en.wikipedia.org/wiki/MinHash) is a probabilistic data structure to cheaply approximate [Jaccard set similarity](https://en.wikipedia.org/wiki/Jaccard_index).

At [Audius](https://audius.co/) we used it to speed up the computation of related artist recommendations.

## Set Similarity

At [Octopart](https://octopart.com/) we used set similarity in a partially automated Entity Resolution process.  We received data feeds from hundreds of different distributors and wanted to normalize and collate the data.

A number of different spelling and naming variations would exist in the raw data, for instance Texas Instruments might appear as
* `TI`
* `TEX. INST`
* `TEXAS INSTR. INC`
* etc, etc

When NXP acquires Freescale, some vendors continue to use the Freescale name, while other list Freescale's parts under the new NXP name.

We would look for two manufacturer names that had many common MPNs (manufacturer part numbers).
A high degree of MPN overlap suggests that two different manufacturer names should be associated.

Our version did some simple sampling from the SQL tables, but while prototyping a new data processing pipeline, I wrote a version that used the [datasketch minhash](https://ekzhu.com/datasketch/minhash.html) library for this task.

## Minhash at Audius

One of the first things I worked on after joining Audius was database performance.
One of the top 5 queries was from the related artist background job.

The code was doing an "all pairs" computation to find similar artists based on mutual followers.
There was a queue and background worker to compute related artists one at a time, but even still postgres was spending lots of CPU on the mutual followers query.

I thought this would be a good opportunity to use the [Minhash](https://en.wikipedia.org/wiki/MinHash) algorithm to do the approximate all pairs computation in a single pass and offload this work from postgres.


I won't attempt to explain how it works here lest I embarrass myself, but I like to think of it as a cool variation of a bloom filter.
Where a bloom filter focuses on per-item membership testing, minhash is a variation that combines all set members into a meaningful key using LSH (location sensitive hashing).

The [datasketch Forrest](https://ekzhu.com/datasketch/lshforest.html) API makes it easy to precompute a follower "minhash" for every artist and then do an all-pairs top-K search to find similar artists based on mutual followers.

Building the index can be a simple postgres query to scan the whole followers table, and then a second loop does the all pairs search, writing the results back to a table.  You can see the code in the [PR](https://github.com/AudiusProject/audius-protocol/pull/3330/files#diff-1a7565761bebb13e8ff4c95ea66d5c3b6513a7311851ebe5d8f4b68177b3d330)

With this code we could recompute the entire related artist table in ~10 minutes.  This is nice because if you want to change some aspect of the formula, you can quickly re-materialize the `related_artists` table and compare it to prior versions, instead of waiting days or weeks for a background task.

Also gone is the queue + worker and the need to reenqueue artists for recomputation based on age + followers changes.  Now we could simply re-compute from scratch every 12 hours with a simple cron task.  And lots of busy work was taken off of postgres' plate.

### Other Options

Having recently introduced ElasticSearch to the Audius stack for search, I was keen to experiment with it for other use cases.

The [More Like This](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-mlt-query.html) query would allow us to improve results by adding additional signals like `genre` similarity.  It is easy to adjust weights a query time to experiment with different recommendations without re-indexing data.

There is a lot of conceptual overlap (pun intended) between Set Similarity and TF/IDF which ElasticSearch uses for relevancy scoring.  ElasticSearch is essentially a set intersection machine!

But the minhash version was done...  we shipped the minhash version and the ElasticSearch version remained an experiment...

_say la v_
