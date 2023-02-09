---
title: Just-in-Time Compilation (JIT)
---

This topic provides an explanation of what Just-in-Time compilation is and how to configure it in Greenplum Database.

**Parent topic:** [Querying Data](../../query/topics/query.html)

## <a id="topic2"></a>What is JIT compilation?

Just-in-Time (JIT) compilation is the process of turning some form of interpreted program evaluation into a native program, and doing so at run time. For example, instead of using general-purpose code that can evaluate arbitrary SQL expressiones to evaluate a particular SQL predicate like `WHERE a.col=3`, it is possible to generate a function that is specific to that expression and can be natively executed by the CPU, resulting in faster execution.

JIT compilation support is enabled with all RPM distributions of Greenplum Database. If you build Greenplum Database from source, you must include the `--with-llvm` build option to include JIT compilation support.

It is possible to use JIT with both Postgres Planner and GPORCA. Since GPORCA and Postgres Planner use different algorithms and the values for the calculated costs are different, you must tune the JIT thresholds according to your usage. See [Configuration](#topic4) for more information.

### <a id="topic21"></a>JIT accelerated operations

Currently Greenplum Database's JIT implementation supports for accelerating expression evaluation and tuple deforming.

Expression evaluation is used to evaluate `WHERE` clauses, target lists, aggregates and projections. It can be accelerated by generating code specific to each case.

Tuple deforming is the process of transforming an on-disk tuple into its in-memory representation. It can be accelerated by creating a function specific to the table layout and the number of columns to be extracted.

### <a id="topic22"></a>In-line compilation (inlining)

Greenplum Database is very extensible and allows new data types, functions, operators and other database objects to be defined. In fact, the built-in objects are implemented using nearly the same mechanisms. This extensibility implies some overhead, for example due to function calls. To reduce that overhead, JIT can use in-line compilation to fit the bodies of small functions into the expressions using them. That allows a significant percentage of the overhead to be optimized away.

### <a id="topic23"></a>Optimization

LLVM has support for optimizing generated code. Some of the optimizations are cheap enough to be performed whenever JIT is used, while others are only beneficial for longer-running queries. See the [LLVM Documentation](https://llvm.org/docs/Passes.html#transform-passes) for more details about optimizations.

## <a id="topic3"></a>When to JIT? 

JIT compilation is beneficial primarily for long-running CPU-bound queries. Frequently these are analytical queries. For short queries the added overhead of performing JIT compilation will often be higher than the time it can save.

To determine whether Greenplum Database should use JIT compilation, it uses the total estimated cost of a query. It compares the estimated cost of the query with the setting of `jit_above_cost`. If the cost is higher, it performs JIT compilation. Two further decisions remain. Firstly, if the estimated cost is more than the setting of `jit_inline_above_cost`, it compiles short functions and operators used in the query using in-line compilation. Secondly, if the estimated cost is more than the setting of `jit_optimize_above_cost`, it applies expensive optimizations to improve the generated code. Each of these options increases the JIT compilation overhead, but can reduce query execution time considerably.

JIT compilation makes these cost-based decisions at plan time, not execution time. This means that when it is using prepared statements, and a generic plan (see [PREPARE](../../../ref_guide/sql_commands/PREPARE.html)), the values of the configuration parameters in effect at prepare time control the decisions, not the settings at execution time.

The Greenplum Database configuration parameter [gp_explain_jit](../../../ref_guide/config_params/guc-list.html#gp_explain_jit) enables the display of summarized JIT information from all query executions when JIT compilation is enabled. `EXPLAIN` can be used to see whether JIT is used or not. For example:

With Postgres Planner:

```
EXPLAIN (ANALYZE) SELECT * FROM jit_explain_output LIMIT 10;
QUERY PLAN
Limit  (cost=35.50..35.67 rows=10 width=4) (actual time=13.199..13.310 rows=10 loops=1)
  ->  Gather Motion 3:1  (slice1; segments: 3)  (cost=35.50..36.01 rows=30 width=4) (actual time=11.848..11.890 rows=10 loops=1)
        ->  Limit  (cost=35.50..35.61 rows=10 width=4) (actual time=0.861..0.971 rows=10 loops=1)
              ->  Seq Scan on jit_explain_output  (cost=0.00..355.00 rows=32100 width=4) (actual time=0.029..0.070 rows=10 loops=1)
Optimizer: Postgres query optimizer
Planning Time: 0.158 ms
  (slice0)    Executor memory: 37K bytes.
  (slice1)    Executor memory: 36K bytes avg x 3 workers, 36K bytes max (seg0).
Memory used:  128000kB
JIT:
  Options: Inlining false, Optimization false, Expressions true, Deforming true.
  (slice0): Functions: 2.00. Timing: 1.381 ms total.
  (slice1): Functions: 1.00 avg x 3 workers, 1.00 max (seg0). Timing: 0.830 ms avg x 3 workers, 0.854 ms max (seg1).
Execution Time: 24.023 ms
(14 rows)
```

With GPORCA:

```
EXPLAIN (ANALYZE) SELECT * FROM jit_explain_output LIMIT 10;
QUERY PLAN
Limit  (cost=0.00..431.00 rows=1 width=4) (actual time=1.103..1.107 rows=10 loops=1)
  ->  Gather Motion 3:1  (slice1; segments: 3)  (cost=0.00..431.00 rows=1 width=4) (actual time=0.013..0.014 rows=10 loops=1)
        ->  Seq Scan on jit_explain_output  (cost=0.00..431.00 rows=1 width=4) (actual time=0.025..0.030 rows=38 loops=1)
Optimizer: Pivotal Optimizer (GPORCA)
Planning Time: 5.824 ms
  (slice0)    Executor memory: 37K bytes.
  (slice1)    Executor memory: 36K bytes avg x 3 workers, 36K bytes max (seg0).
Memory used:  128000kB
JIT:
  Options: Inlining false, Optimization false, Expressions true, Deforming true.
  (slice0): Functions: 2.00. Timing: 1.137 ms total.
Execution Time: 1.597 ms
(12 rows)
```

In the above examples, the configuration parameter `jit_above_cost` was modified so it would trigger JIT compilation. The use of JIT affects the cost of the plan, which can or cannot be bigger than the potential savings. JIT was used, but inlining and expensive optimization were not. If `jit_inline_above_cost` or `jit_optimize_above_cost` were also lowered, they could be triggered.

**Note**: You must turn off the configuration parameter `gp_explain_jit` when running regression tests.

## <a id="topic4"></a>Configuration

The configuration parameter [jit](../../../ref_guide/config_params/guc-list.html#jit) determines whether JIT compilation is enabled or disabled. If it is enabled, the decision whether or not to use JIT is cost-based is controlled by the configuration parameters:

- [jit_above_cost](../../../ref_guide/config_params/guc-list.html#jit_above_cost): All queries with a higher total cost trigger JIT compilation, *without* optimization (expensive part). This commonly already results in significant speedups if expression/deforming is a bottleneck (removing dynamic branches mostly).
- [jit_optimize_above_cost](../../../ref_guide/config_params/guc-list.html#jit_optimize_above_cost): all queries with a higher total cost trigger JIT, *with* optimization (expensive part).
- [jit_inline_above_cost](../../../ref_guide/config_params/guc-list.html#jit_inline_above_cost): inlining is tried if the query has higher cost.

Whenever the total cost of a query is above these limits, JIT compilation takes place.

Note that setting the JIT cost parameters to ‘0’ will force all queries to be JIT-compiled and, as a result, slow down queries. Setting the JIT cost to a negative value will disable JIT.

You should tune these configuration parameters when you enable or disable GPORCA, as the meaning of cost is different for GPORCA and Postgres Planner.

