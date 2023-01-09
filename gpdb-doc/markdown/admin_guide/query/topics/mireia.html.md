---
title: Just-in-Time Compilation (JIT)
---

This topic provides an explanation of what Just-in-Time compilation is and how it can be configured in Greenplum Database.

**Parent topic:** [Querying Data](../../query/topics/query.html)

## <a id="topic2"></a>What is JIT compilation?

Just-in-Time (JIT) compilation is the process of turning some form of interpreted program evaluation into a native program, and doing so at run time.
For example, instead of using general-purpose code that can evaluate arbitrary SQL expressiones to evaluate a particular SQL predicate like WHERE a.col=3, it is possible to generate a function that is specific to that expression and can be natively executed by the CPU, yielding a speedup.

Greenplum Database has builtin support to perform JIT compilation using LLVM when Greenplum Database is built with --with-llvm.

### <a id="topic21"></a>JIT accelerated operations

Currently Greenplum Database's JIT implementation has support for accelerating expression evaluation and tuple deforming. Several other operations could be accelerated in the future.

Expression evaluation is used to evaluate `WHERE` clauses, target lists, aggregates and projections. It can be accelerated by generating code specific to each case.

Tuple deforming is the process of transforming an on-disk tuple into its in-memory representation. It can be accelerated by creating a function specific to the table layout and the number of columns to be extracted.

### <a id="topic22"></a>Inlining

Greenplum Database is very extensible and allows new data types, functions, operators and other database objects to be defined. In fact the built-in objects are implemented using nearly the same mechanisms. This extensibility implies some overhead, for example due to function calls. To reduce that overhead, JIT compilation can inline the bodies of small functions into the expressions using them. That allows a significant percentage of the overhead to be optimized away.

### <a id="topic23"></a>Optimization

LLVM has support for optimizing generated code. Some of the optimizations are cheap enough to be performed whenever JIT is used, while others are only beneficial for longer-running queries. See https://llvm.org/docs/Passes.html#transform-passes for more details about optimizations.

## <a id="topic3"></a>When to JIT? 

JIT compilation is beneficial primarily for long-running CPU-bound queries. Frequently these will be analytical queries. For short queries the added overhead of performing JIT compilation will often be higher than the time it can save.

To determine whether JIT compilation should be used, the total estimated cost of a query is used. The estimated cost of the query will be compared with the setting of `jit_above_cost`. If the cost is higher, JIT compilation will be performed. Two further decisions are then needed. Firstly, if the estimated cost is more than the setting of `jit_inline_above_cost`, short functions and operators used in the query will be inlined. Secondly, if the estimated cost is more than the setting of `jit_optimize_above_cost`, expensive optimizations are applied to improve the generated code. Each of these options increases the JIT compilation overhead, but can reduce query execution time considerably.

These cost-based decisions will be made at plan time, not execution time. This means that when prepared statements are in use, and a generic plan is used (see `PREPARE`), the values of the configuration parameters in effect at prepare time control the decisions, not the settings at execution time.

`EXPLAIN` can be used to see whether JIT is used or not. As an example, here is a query that is not using JIT:

```
=# EXPLAIN ANALYZE SELECT SUM(relpages) FROM pg_class;
                                                 QUERY PLAN
-------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=16.27..16.29 rows=1 width=8) (actual time=0.303..0.303 rows=1 loops=1)
   ->  Seq Scan on pg_class  (cost=0.00..15.42 rows=342 width=4) (actual time=0.017..0.111 rows=356 loops=1)
 Planning Time: 0.116 ms
 Execution Time: 0.365 ms
(4 rows)
```

Given the cost of the plan, it is entirely reasonable that no JIT was used; the cost of JIT would have been bigger than the potential savings. Adjusting the cost limits will lead to JIT use: (Note the `JIT:` lines in output is a hint of actived JIT).

```
=# SET jit_above_cost = 10;
SET
=# EXPLAIN ANALYZE SELECT SUM(relpages) FROM pg_class;
                                                 QUERY PLAN
-------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=16.27..16.29 rows=1 width=8) (actual time=6.049..6.049 rows=1 loops=1)
   ->  Seq Scan on pg_class  (cost=0.00..15.42 rows=342 width=4) (actual time=0.019..0.052 rows=356 loops=1)
 Planning Time: 0.133 ms
 JIT:
   Functions: 3
   Options: Inlining false, Optimization false, Expressions true, Deforming true
   Timing: Generation 1.259 ms, Inlining 0.000 ms, Optimization 0.797 ms, Emission 5.048 ms, Total 7.104 ms
 Execution Time: 7.416 ms
```

As visible here, JIT was used, but inlining and expensive optimization were not. If `jit_inline_above_cost` or `jit_optimize_above_cost` were also lowered, that would change.

## <a id="topic4"></a>Configuration

The configuration variable `jit` determines whether JIT compilation is enabled or disabled. If it is enabled, the configuration variables `jit_above_cost`, `jit_inline_above_cost`, and `jit_optimize_above_cost` determine whether JIT compilation is performed for a query, and how much effort is spent doing so.

The decision whether to use or not JIT is cost-based, controlled by GUCs:


jit
jit_above_cost
jit_inline_above_cost
jit_optimize_above_cost
jit_provider

`jit_provider` determines which JIT implementation is used. It is rarely required to be changed.


