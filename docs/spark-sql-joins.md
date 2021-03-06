title: Joins

# Dataset Join Operators

From PostgreSQL's https://www.postgresql.org/docs/current/static/tutorial-join.html[2.6. Joins Between Tables]:

> Queries can access multiple tables at once, or access the same table in such a way that multiple rows of the table are being processed at the same time. A query that accesses multiple rows of the same or different tables at one time is called a *join query*.

You can join two datasets using the <<join-operators, join operators>> with an optional <<join-condition, join condition>>.

[[join-operators]]
.Join Operators
[width="100%",cols="1,1,2",options="header"]
|===
| Operator
| Return Type
| Description

| <<crossJoin, crossJoin>>
| spark-sql-DataFrame.md[DataFrame]
| Untyped ``Row``-based cross join

| <<join, join>>
| spark-sql-DataFrame.md[DataFrame]
| Untyped ``Row``-based join

| <<joinWith, joinWith>>
| Dataset.md[Dataset]
| Used for a type-preserving join with two output columns for records for which a join condition holds
|===

You can also use SparkSession.md#sql[SQL mode] to join datasets using _good ol'_ SQL.

[source, scala]
----
val spark: SparkSession = ...
spark.sql("select * from t1, t2 where t1.id = t2.id")
----

[[join-condition]]
You can specify a *join condition* (aka _join expression_) as part of join operators or using spark-sql-dataset-operators.md#where[where] or spark-sql-dataset-operators.md#filter[filter] operators.

[source, scala]
----
df1.join(df2, $"df1Key" === $"df2Key")
df1.join(df2).where($"df1Key" === $"df2Key")
df1.join(df2).filter($"df1Key" === $"df2Key")
----

You can specify the <<join-types, join type>> as part of join operators (using `joinType` optional parameter).

[source, scala]
----
df1.join(df2, $"df1Key" === $"df2Key", "inner")
----

[[join-types]]
.Join Types
[cols="1,1,1",options="header",width="100%"]
|===
| SQL
| Name (joinType)
| JoinType

| [[CROSS]] `CROSS`
| [[cross]] `cross`
| [[Cross]] `Cross`

| [[INNER]] `INNER`
| [[inner]] `inner`
| [[Inner]] `Inner`

| [[FullOuter]][[FULL_OUTER]] `FULL OUTER`
| `outer`, `full`, `fullouter`
| `FullOuter`

| [[LEFT_ANTI]] `LEFT ANTI`
| `leftanti`
| [[LeftAnti]] `LeftAnti`

| [[LEFT_OUTER]] `LEFT OUTER`
| `leftouter`, `left`
| [[LeftOuter]] `LeftOuter`

| [[LEFT_SEMI]] `LEFT SEMI`
| `leftsemi`
| [[LeftSemi]] `LeftSemi`

| [[RIGHT_OUTER]] `RIGHT OUTER`
| `rightouter`, `right`
| [[RightOuter]] `RightOuter`

| [[NATURAL]] `NATURAL`
| Special case for `Inner`, `LeftOuter`, `RightOuter`, `FullOuter`
| `NaturalJoin`

| [[using]][[USING]] `USING`
| Special case for `Inner`, `LeftOuter`, `LeftSemi`, `RightOuter`, `FullOuter`, `LeftAnti`
| [[UsingJoin]] `UsingJoin`
|===

[[ExistenceJoin]]
`ExistenceJoin` is an artifical join type used to express an existential sub-query, that is often referred to as *existential join*.

NOTE: <<LeftAnti, LeftAnti>> and <<ExistenceJoin, ExistenceJoin>> are special cases of <<LeftOuter, LeftOuter>>.

You can also find that Spark SQL uses the following two families of joins:

* [[InnerLike]] `InnerLike` with <<Inner, Inner>> and <<Cross, Cross>>

* [[LeftExistence]] `LeftExistence` with <<LeftSemi, LeftSemi>>, <<LeftAnti, LeftAnti>> and <<ExistenceJoin, ExistenceJoin>>

TIP: Name are case-insensitive and can use the underscore (`_`) at any position, i.e. `left_anti` and `LEFT_ANTI` are equivalent.

!!! note
    Spark SQL offers different [join strategies](execution-planning-strategies/JoinSelection.md#join-selection-requirements) with [Broadcast Joins (aka Map-Side Joins)](spark-sql-joins-broadcast.md) among them that are supposed to optimize your join queries over large distributed datasets.

=== [[join]] `join` Operators

[source, scala]
----
join(right: Dataset[_]): DataFrame // <1>
join(right: Dataset[_], usingColumn: String): DataFrame // <2>
join(right: Dataset[_], usingColumns: Seq[String]): DataFrame // <3>
join(right: Dataset[_], usingColumns: Seq[String], joinType: String): DataFrame // <4>
join(right: Dataset[_], joinExprs: Column): DataFrame // <5>
join(right: Dataset[_], joinExprs: Column, joinType: String): DataFrame // <6>
----
<1> Condition-less inner join
<2> Inner join with a single column that exists on both sides
<3> Inner join with columns that exist on both sides
<4> Equi-join with explicit join type
<5> Inner join
<6> Join with explicit join type. Self-joins are acceptable.

`join` joins two ``Dataset``s.

[source, scala]
----
val left = Seq((0, "zero"), (1, "one")).toDF("id", "left")
val right = Seq((0, "zero"), (2, "two"), (3, "three")).toDF("id", "right")

// Inner join
scala> left.join(right, "id").show
+---+----+-----+
| id|left|right|
+---+----+-----+
|  0|zero| zero|
+---+----+-----+

scala> left.join(right, "id").explain
== Physical Plan ==
*Project [id#50, left#51, right#61]
+- *BroadcastHashJoin [id#50], [id#60], Inner, BuildRight
   :- LocalTableScan [id#50, left#51]
   +- BroadcastExchange HashedRelationBroadcastMode(List(cast(input[0, int, false] as bigint)))
      +- LocalTableScan [id#60, right#61]

// Full outer
scala> left.join(right, Seq("id"), "fullouter").show
+---+----+-----+
| id|left|right|
+---+----+-----+
|  1| one| null|
|  3|null|three|
|  2|null|  two|
|  0|zero| zero|
+---+----+-----+

scala> left.join(right, Seq("id"), "fullouter").explain
== Physical Plan ==
*Project [coalesce(id#50, id#60) AS id#85, left#51, right#61]
+- SortMergeJoin [id#50], [id#60], FullOuter
   :- *Sort [id#50 ASC NULLS FIRST], false, 0
   :  +- Exchange hashpartitioning(id#50, 200)
   :     +- LocalTableScan [id#50, left#51]
   +- *Sort [id#60 ASC NULLS FIRST], false, 0
      +- Exchange hashpartitioning(id#60, 200)
         +- LocalTableScan [id#60, right#61]

// Left anti
scala> left.join(right, Seq("id"), "leftanti").show
+---+----+
| id|left|
+---+----+
|  1| one|
+---+----+

scala> left.join(right, Seq("id"), "leftanti").explain
== Physical Plan ==
*BroadcastHashJoin [id#50], [id#60], LeftAnti, BuildRight
:- LocalTableScan [id#50, left#51]
+- BroadcastExchange HashedRelationBroadcastMode(List(cast(input[0, int, false] as bigint)))
   +- LocalTableScan [id#60]
----

Internally, `join(right: Dataset[_])` Dataset.md#ofRows[creates a DataFrame] with a condition-less spark-sql-LogicalPlan-Join.md[Join] logical operator (in the current SparkSession.md[SparkSession]).

NOTE: `join(right: Dataset[_])` creates a spark-sql-LogicalPlan.md[logical plan] with a condition-less spark-sql-LogicalPlan-Join.md[Join] operator with two child logical plans of the both sides of the join.

NOTE: `join(right: Dataset[_], usingColumns: Seq[String], joinType: String)` creates a spark-sql-LogicalPlan.md[logical plan] with a condition-less spark-sql-LogicalPlan-Join.md[Join] operator with <<UsingJoin, UsingJoin>> join type.

[NOTE]
====
`join(right: Dataset[_], joinExprs: Column, joinType: String)` accepts self-joins where `joinExprs` is of the form:

```
df("key") === df("key")
```

That is usually considered a trivially true condition and refused as acceptable.

With spark-sql-properties.md#spark.sql.selfJoinAutoResolveAmbiguity[spark.sql.selfJoinAutoResolveAmbiguity] option enabled (which it is by default), `join` will automatically resolve ambiguous join conditions into ones that might make sense.

See https://issues.apache.org/jira/browse/SPARK-6231[[SPARK-6231\] Join on two tables (generated from same one) is broken].
====

=== [[crossJoin]] `crossJoin` Method

[source, scala]
----
crossJoin(right: Dataset[_]): DataFrame
----

`crossJoin` joins two Dataset.md[Datasets] using <<cross, Cross>> join type with no condition.

NOTE: `crossJoin` creates an explicit cartesian join that can be very expensive without an extra filter (that can be pushed down).

=== [[joinWith]] Type-Preserving Joins -- `joinWith` Operators

[source, scala]
----
joinWith[U](other: Dataset[U], condition: Column): Dataset[(T, U)]  // <1>
joinWith[U](other: Dataset[U], condition: Column, joinType: String): Dataset[(T, U)]
----
<1> inner equi-join

`joinWith` creates a Dataset.md[Dataset] with two columns `_1` and `_2` that each contain records for which `condition` holds.

[source, scala]
----
case class Person(id: Long, name: String, cityId: Long)
case class City(id: Long, name: String)
val family = Seq(
  Person(0, "Agata", 0),
  Person(1, "Iweta", 0),
  Person(2, "Patryk", 2),
  Person(3, "Maksym", 0)).toDS
val cities = Seq(
  City(0, "Warsaw"),
  City(1, "Washington"),
  City(2, "Sopot")).toDS

val joined = family.joinWith(cities, family("cityId") === cities("id"))
scala> joined.printSchema
root
 |-- _1: struct (nullable = false)
 |    |-- id: long (nullable = false)
 |    |-- name: string (nullable = true)
 |    |-- cityId: long (nullable = false)
 |-- _2: struct (nullable = false)
 |    |-- id: long (nullable = false)
 |    |-- name: string (nullable = true)
scala> joined.show
+------------+----------+
|          _1|        _2|
+------------+----------+
| [0,Agata,0]|[0,Warsaw]|
| [1,Iweta,0]|[0,Warsaw]|
|[2,Patryk,2]| [2,Sopot]|
|[3,Maksym,0]|[0,Warsaw]|
+------------+----------+
----

NOTE: `joinWith` preserves type-safety with the original object types.

NOTE: `joinWith` creates a `Dataset` with spark-sql-LogicalPlan-Join.md[Join] logical plan.
