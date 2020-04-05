---
title: Fast aggregations using SIMD instructions
author: Tancrede Collard
---

<img class="banner-4-2" src="/blog/assets/banner-4-2.png" alt="drawing"/> 

<a href="https://en.wikipedia.org/wiki/SIMD" target="_blank">SIMD instructions</a> are specific CPU instruction sets for arithmetic calculations that use synthetic parallelisation. 
The parallelisation is synthetic because instead of spreading the work across CPU cores, 
SIMD performs vector operations on multiple items using a **single** CPU instruction. 
In practice, if you were to to add 8 numbers together, SIMD does that in 1 operation instead of 8.
We get compounded performance improvements by combining SIMD with actual parallelisation and spanning the work across CPUs.


QuestDB 4.2 introduces SIMD instructions, which made our aggregations faster by 100x! 
QuestDB is available [open-source under Apache 2.0](https://github.com/questdb/questdb).
 
 As of now, SIMD operations are available for non-keyed aggregation queries, such as
```select sum(value) from table```. In further releases, we will extend these to keyed aggregations, for example
```select key, sum(value) from table``` (note the intentional omission of `GROUP BY`). This will also result in ultrafast 
aggregation for time-bucketed queries using `SAMPLE BY`.

If you like what we do, please consider <b> <a href="https://github.com/questdb/questdb"> following us on Github and starring our project <img class="yellow-star" src="/img/star-yellow.svg"/></a></b>


### How fast is it?
To get an idea of how fast aggregations have become, we ran a benchmark against kdb+, which is one of the fastest databases out there. Coincidentally, their new version 4.0 (released a few days ago) introduces performance improvements through implicit parallelism. 

#### Setup
We have benchmarked QuestDB against kdb's latest version using 2 different CPUs: the [Intel 8850H](https://ark.intel.com/content/www/us/en/ark/products/134899/intel-core-i7-8850h-processor-9m-cache-up-to-4-30-ghz.html) 
and the [AMD Ryzen 3900X](https://www.amd.com/en/products/cpu/amd-ryzen-9-3900x). Both databases were running on 4 threads.

#### Queries
|Test	|Query (kdb+ 4.0)	|Query (QuestDB 4.2)|
|---|---|---|
|sum of 1Bn doubles <br/> no nulls|zz:1000000000?1000.0 <br/>\t sum zz	|create table 1G_double_nonNull as (select rnd_double() d from long_sequence(1000000000)); <br/> select sum(d) from 1G_double_nonNull;|
|sum of 1Bn ints |zz:1000000000?1000i <br/> \t sum zz         |create table 1G_int_nonNull as (select rnd_int() i from long_sequence(1000000000)); <br/> select sum(i) from 1G_int_nonNull; |
|sum of 1Bn longs	|zz:1000000000?1000j <br/>\t sum zz|	create table 1G_long_nonNull as (select rnd_long() l from long_sequence(1000000000));<br/>select sum(l) from 1G_long_nonNull;|
|max of 1Bn doubles|zz:1000000000?1000.0<br/>\t max zz|	create table 1G_double_nonNull as (select rnd_double() d from long_sequence(1000000000));<br/>select max(d) from 1G_double_nonNull;|
|max of 1Bn longs |zz:1000000000?1000<br/>\t max zz|	create table 1G_long_nonNull as (select rnd_long() l from long_sequence(1000000000));<br/>select max(l) from 1G_long_nonNull;|

#### Results
![alt-text](assets/bench-kdb-8850h.png)
![alt-text](assets/bench-kdb-3900x.png)

The synthetic test above does not generate NULL values. What is interestin that as soon as data contains NULL values KDB+ sum() performance drops while QuestDB sum() does not. All other aggregate function in both KDB+ and QuestDB are unaffected.

|Test	|Query (kdb+ 4.0)	|Query (QuestDB 4.2)|
|---|---|---|
|sum of 1G doubles <br/>(nulls)	|zz:1000000000?1000.0 <br/>zz:?[zz<100;0Nf;zz]<br/>\t sum zz|	create table 1G_double as (select rnd_double(5) d from long_sequence(1000000000));<br/>select sum(d) from 1G_double;|

![alt-text](assets/bench-kdb-8850H-sum-null.png)

QuestDB's sum(int) result is 64-bit long sum, wheras KDB sum(int) overflows 32-bit sum. There is a little bit of scope left to make our implementation faster in the future.

### Perspectives on performance

The execution times outlined above become more interesting once put into context. This is how QuestDB compares to Postgres to sum 1 billion numbers from a given table `select sum(d) from 1G_double_nonNull`. 

![alt-text](assets/bench-pg-kdb-quest.png)

We found that our performance figures to be constained by available memory channels on CPU. If CPU has two memory channels, like in above examples, throwing more cores at the problem does not change outcome at all. This is applicable to both KDB+ and QuestDB. On other hand if CPU has more memory channels, pefromance scales almost lineraly. This is an example of `sum(doule)` on Amazon c5.metal instance using 16 threads:

todo: show chart comparing 260ms and 100ms (on c5.metal)

Amazon c5.metal uses two 24-core Intel 8275CL with 6 memory channels each. Unfortunately CPUs are hyperthreaded. It is possible that performance could be even higher if CPU are be fully isolated to do the computations.

If you have easy access to 8- or 12-channel servers and would like to benchmark QuestDB we'd love to hear the results. You can <a href="https://www.questdb.io/getstarted">download QuestDB here</a> and please don't forget to <b> <a href="https://github.com/questdb/questdb"> follow us on Github and star our project <img class="yellow-star" src="/img/star-yellow.svg"/></a></b>

### What is next?
In further releases, we will roll out this model to other parts of our SQL. QuestDB implements SIMD in a generic 
fashion. This allows us to translate this model into about everything our SQL engine does, such as keyed aggregations, 
indexing etc. We will also keep improving QuestDB's performance. Through some further work on assembly, we estimate that we can gain another 15% speed on these 
operations. In the meantime, if you want to know exactly how we have achieved this, 100% of our code is **[open-source](https://github.com/questdb/questdb)**!


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