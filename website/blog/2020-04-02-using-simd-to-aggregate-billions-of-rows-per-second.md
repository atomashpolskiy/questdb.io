---
title: QuestDB - Using SIMD to aggregate billions of rows per second
author: Tancrede Collard
---

<a href="https://www.questdb.io/getstarted" target="_blank"><img class="banner-4-2" src="/blog/assets/banner-4-2.png" alt="drawing"/></a> 

<a href="https://en.wikipedia.org/wiki/SIMD" target="_blank">SIMD instructions</a> are specific CPU instruction sets for arithmetic calculations that use synthetic parallelisation. 
The parallelisation is synthetic because instead of spreading the work across CPU cores, 
SIMD performs vector operations on multiple items using a **single** CPU instruction. 
In practice, if you were to add 8 numbers together, SIMD does that in 1 operation instead of 8.
We get compounded performance improvements by combining SIMD with actual parallelisation and spanning the work across CPUs.

QuestDB 4.2 introduces SIMD instructions, which made our aggregations faster by 100x! 
QuestDB is available [open-source under Apache 2.0](https://github.com/questdb/questdb). If you like what we do, please consider <b> <a href="https://github.com/questdb/questdb"> following us on Github and starring our project <img class="yellow-star" src="/img/star-yellow.svg"/></a></b>
 
 As of now, SIMD operations are available for non-keyed aggregation queries, such as
```select sum(value) from table```. In future releases, we will extend these to keyed aggregations, for example
```select key, sum(value) from table``` (note the intentional omission of `GROUP BY`). This will also result in ultrafast 
aggregation for time-bucketed queries using `SAMPLE BY`.

### How fast is it?
To get an idea of how fast aggregations have become, we ran a benchmark against kdb+, which is one of the fastest databases out there. 
Coincidentally, their new version 4.0 (released a few days ago) introduces performance improvements through implicit parallelism. 

#### Setup
We have benchmarked QuestDB against kdb's latest version using 2 different CPUs: the [Intel 8850H](https://ark.intel.com/content/www/us/en/ark/products/134899/intel-core-i7-8850h-processor-9m-cache-up-to-4-30-ghz.html) 
and the [AMD Ryzen 3900X](https://www.amd.com/en/products/cpu/amd-ryzen-9-3900x). Both databases were running on 4 threads.

#### Queries
|Test	|Query (kdb+ 4.0)	|Query (QuestDB 4.2)|
|---|---|---|
|sum of 1Bn doubles <br/> no nulls|zz:1000000000?1000.0 <br/>\t sum zz	| create table zz as (select rnd_double() d from long_sequence(1000000000)); <br/> select sum(d) from zz;|
|sum of 1Bn ints |zz:1000000000?1000i <br/> \t sum zz |create table zz as (select rnd_int() i from long_sequence(1000000000)); <br/> select sum(i) from zz; |
|sum of 1Bn longs	|zz:1000000000?1000j <br/>\t sum zz|	create table zz as (select rnd_long() l from long_sequence(1000000000));<br/>select sum(l) from zz;|
|max of 1Bn doubles|zz:1000000000?1000.0<br/>\t max zz|	create table zz as (select rnd_double() d from long_sequence(1000000000));<br/>select max(d) from zz;|
|max of 1Bn longs |zz:1000000000?1000<br/>\t max zz|	create table zz as (select rnd_long() l from long_sequence(1000000000));<br/>select max(l) from zz;|

#### Results
![alt-text](assets/bench-kdb-8850h.png)


![alt-text](assets/bench-kdb-3900x.png)

The dataset producing the results shown above does not contain NULL values. Interestingly, as soon as the data contains NULL values, kdb+ sum() performance drops while QuestDB sum() query time is unchanged as seen on the chart below.

|Test	|Query (kdb+ 4.0)	|Query (QuestDB 4.2)|
|---|---|---|
|sum of 1Bn doubles <br/>(nulls)	|zz:1000000000?1000.0 <br/>zz:?[zz<100;0Nf;zz]<br/>\t sum zz|	create table zz as (select rnd_double(5) d from long_sequence(1000000000));<br/>select sum(d) from zz;|

![alt-text](assets/bench-kdb-8850H-sum-null.png)

#### We can improve this performance further
QuestDB's sum(int) result is 64-bit long, whereas kdb+ sum(int) returns a 32-bit integer (even if the sum overflows). 
Our approach is currently slightly more complicated as we convert each 32-bit integer to a 64-bit long to avoid overflow. 
By removing this overhead and more, there is scope left to make our implementation faster in the future.

### Perspectives on performance

The execution times outlined above become more interesting once put into context. 
This is how QuestDB compares to Postgres when doing a sum of 1 billion numbers from a given table `select sum(d) from 1G_double_nonNull`. 

![alt-text](assets/bench-pg-kdb-quest.png)

We found that our performance figures are constrained by the available memory channels. Both the 8850H and the 3900X 
have 2 memory channels, and throwing more than 4 cores at the query above does not improve the performance. 
On the other hand, if the CPU has more memory channels, then performance scales almost linearly for both kdb+ and QuestDB. 

To get an idea of the impact of memory channels, we spun off a m5.metal instance on AWS. This instance has two 
 24-core Intel 8275CL with 6 memory channels each. Here are the results compared to the 2-channel 3900X:

|cpu cores|1|2|3|4|5|6|7|8|9|10|11|12|
|---|---|---|---|---|---|---|---|---|---|---|---|---|
|8275CL|910|605|380|240|193|176|156|148|140|136|133|141|
|3900X|621|502|381|260|260|260|260|260|260|260|260|260|

We plot those results below on the left. On the right-hand side, we normalise the results for each CPU and plot the performance 
improvement of going from 1 to more cores. 

![alt-text](assets/core-scale.png)

Interestingly, the 2-channel 3900X, is much faster on 1 core than the 
8275CL. But it does not scale well and hits a performance ceiling at 4 cores. This is because it only has 2 memory channels 
that are already saturated. The 6-channel 8275CL allows QuestDB to 
scale almost linearly as we add more CPU cores and hits a performance ceiling at around 12 cores.

Unfortunately AWS CPUs are hyperthreaded. 
We could unpack even more performance if CPU were fully isolated to run the computations. 

We did not get our hands on CPUs with more memory channels for this test, but if you have easy access to 8 or 12-channel servers and would like to benchmark QuestDB, we'd love to hear the results. 
You can <a href="https://www.questdb.io/getstarted">download QuestDB</a> and leave a <a target="_blank" href="https://github.com/questdb/questdb/issues/146">comment on github</a>

### What is next?
In further releases, we will roll out this functionality to other parts of our SQL implementation. QuestDB implements SIMD in a generic fashion, which will allow us to continue adding SIMD to about everything our SQL engine does, such as keyed aggregations, indexing etc. We will also keep improving QuestDB's performance. Through some further work on assembly, we estimate that we can gain another 15% speed on these 
operations. In the meantime, if you want to know exactly how we have achieved this, all of our code is **[open-source](https://github.com/questdb/questdb)**!


### About the release: QuestDB 4.2

#### Summary
We have implemented SIMD-based vector execution of queries, such as `select sum(value) from table`.
This is ~100x faster than non-vector based execution. This is just the beginning as we will introduce vectors to more operations going forward.
Try our first implementation in this release - stay tuned for more features in the upcoming releases!

#### Important
Metadata file format has been changed to include a new flag for columns of type symbol. 
It is necessary to convert existing tables to new format. Running the following sql: `repair table myTable` will update the table metadata.

#### What is new?
- Java: vectorized sum(), avg(), min(), max() for DOUBLE, LONG, INT
- Java: select distinct symbol optimisation
- FreeBSD support
- Automatically restore data consistency and recover from partial data loss.

#### What we fixed
- SQL: NPE when parsing SQL text with malformed table name expression , for example ')', or ', blah'
- SQL: parsing 'fill' clause in sub-query context was causing unexpected syntax error (#115)
- SQL: possible internal error when ordering result of group-by or sample-by
- Data Import: Ignore byte order marks (BOM) in table names created from an imported CSV (#114)
- SQL: 'timestamp' propagation thru group-by code had issues. sum() was tripping over null values. Added last() aggregate function. (#113)
- LOG: make service log names consistent on windows (#106)
- SQL: deal with the following syntax 'select * from select ( select a from ....)'
- SQL: allow the following syntax 'case cast(x as int) when 1 then ...'
- fix(griffin): syntax check for "case"-')' overlap, e.g. "a + (case when .. ) end"
