# ExperimentalMethods

`ExperimentalMethods` holds extra <<extraOptimizations, optimizations>> and <<extraStrategies, strategies>> that are used in [SparkOptimizer](SparkOptimizer.md#User-Provided-Optimizers) and [SparkPlanner](SparkPlanner.md), respectively.

[[attributes]]
.ExperimentalMethods' Attributes
[width="100%",cols="1m,2",options="header"]
|===
| Name
| Description

| extraOptimizations
a| [[extraOptimizations]] Collection of catalyst/Rule.md[rules] to optimize spark-sql-LogicalPlan.md[LogicalPlans] (i.e. `Rule[LogicalPlan]` objects)

[source, scala]
----
extraOptimizations: Seq[Rule[LogicalPlan]]
----

Used when `SparkOptimizer` is requested for the [User Provided Optimizers](SparkOptimizer.md#User-Provided-Optimizers)

| extraStrategies
a| [[extraStrategies]] Collection of [SparkStrategies](execution-planning-strategies/SparkStrategy.md)

[source, scala]
----
extraStrategies: Seq[Strategy]
----

Used when `SessionState` is requested for the SessionState.md#planner[SparkPlanner]
|===

`ExperimentalMethods` is available as the <<SparkSession.md#experimental, experimental>> property of a `SparkSession`.

[source, scala]
----
scala> :type spark
org.apache.spark.sql.SparkSession

scala> :type spark.experimental
org.apache.spark.sql.ExperimentalMethods
----

=== Example

[source, scala]
----
import org.apache.spark.sql.catalyst.rules.Rule
import org.apache.spark.sql.catalyst.plans.logical.LogicalPlan

object SampleRule extends Rule[LogicalPlan] {
  def apply(p: LogicalPlan): LogicalPlan = p
}

scala> :type spark
org.apache.spark.sql.SparkSession

spark.experimental.extraOptimizations = Seq(SampleRule)

// extraOptimizations is used in Spark Optimizer
val rule = spark.sessionState.optimizer.batches.flatMap(_.rules).filter(_ == SampleRule).head
scala> rule.ruleName
res0: String = SampleRule
----
