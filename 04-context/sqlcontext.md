# SqlContext的运行过程
SparkSQL有两个分支SqlContext和HiveContext，SqlContext现在只支持sql语法解析器（SQL-92语法），而HiveContext现在既支持sql语法解析器又支持hivesql语法解析器，默认为hivesql语法解析器，用户可以通过配置切换成sql语法解析器，来运行hiveql不支持的语法。

SqlContext使用sqlContext.sql(sqlText)来提交用户sql语句，SqlContext首先会调用parserSql对sqlText进行语法分析，然后返回给用户DataFrame。Dataframe继承自RDDApi[Row]。

```
  /**
   * Executes a SQL query using Spark, returning the result as a SchemaRDD.  The dialect that is
   * used for SQL parsing can be configured with 'spark.sql.dialect'.
   *
   * @group userf
   */
   def sql(sqlText: String): DataFrame = {
      if (conf.dialect == "sql") {
        DataFrame(this, parseSql(sqlText))
      } else {
        sys.error(s"Unsupported SQL dialect: ${conf.dialect}")
      }
    }

class DataFrame private[sql](
    @transient val sqlContext: SQLContext,
    @DeveloperApi @transient val queryExecution: SQLContext#QueryExecution)
  extends RDDApi[Row] with Serializable {
```
  /**
   * An internal interface defining the RDD-like methods for [[DataFrame]].
   * Please use [[DataFrame]] directly, and do NOT use this.
   */
  private[sql] trait RDDApi[T] {
    def cache(): this.type
  
    def persist(): this.type
  ...
  }
 
parseSql首先会尝试dll语法解析，如果失败则进行sql语法解析。
```
  protected[sql] def parseSql(sql: String): LogicalPlan = {
    ddlParser(sql, false).getOrElse(sqlParser(sql))
  }
```

然后调用DataFrame中的辅助构造函数，其再调用sqlContext.executePlan(logicalPlan)来生成QueryExecution对象，此类将完成关系查询的主要流程。
```
  def this(sqlContext: SQLContext, logicalPlan: LogicalPlan) = {
    this(sqlContext, {
      val qe = sqlContext.executePlan(logicalPlan)
      if (sqlContext.conf.dataFrameEagerAnalysis) {
        qe.assertAnalyzed()  // This should force analysis and throw errors if there are any
      }
      qe
    })
  }
  ...
  
  protected[sql] def executePlan(plan: LogicalPlan) = new this.QueryExecution(plan)
  
```
先看看QueryExecution的代码
```
  protected[sql] class QueryExecution(val logical: LogicalPlan) {
    def assertAnalyzed(): Unit = analyzer.checkAnalysis(analyzed)

    lazy val analyzed: LogicalPlan = analyzer(logical)
    lazy val withCachedData: LogicalPlan = {
      assertAnalyzed()
      cacheManager.useCachedData(analyzed)
    }
    lazy val optimizedPlan: LogicalPlan = optimizer(withCachedData)

    // TODO: Don't just pick the first one...
    lazy val sparkPlan: SparkPlan = {
      SparkPlan.currentContext.set(self)
      planner(optimizedPlan).next()
    }
    // executedPlan should not be used to initialize any SparkPlan. It should be
    // only used for execution.
    lazy val executedPlan: SparkPlan = prepareForExecution(sparkPlan)

    /** Internal version of the RDD. Avoids copies and has no schema */
    lazy val toRdd: RDD[Row] = executedPlan.execute()

``}

QueryExecution的执行如下
1. 使用analyzer结合数据数据字典（catalog）进行绑定，生成resolved LogicalPlan
2. 处理UDF
3. 处理Cache
4. 使用optimizer对resolved LogicalPlan进行优化，生成optimized LogicalPlan
5. 使用SparkPlan将LogicalPlan转换成PhysicalPlan
6. 使用prepareForExecution()将PhysicalPlan转换成可执行物理计划
7. 使用execute()执行可执行物理计划
8. 生成SchemaRDD

![](/images/SqlContext-Execution.png)







