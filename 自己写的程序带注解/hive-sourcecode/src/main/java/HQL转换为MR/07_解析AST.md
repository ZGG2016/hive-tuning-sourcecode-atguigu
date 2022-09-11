
在 `05_进入编译HQL代码.md` 中，进入到了编译代码部分，

在 `06_解析器完成AST` 中，得到了AST树，接下来开始解析


```java
public class Driver implements IDriver {

    private void compile(String command, boolean resetTaskIds, boolean deferClose) throws CommandProcessorResponse {

        ASTNode tree;
        try {
            // 生成AST树
            tree = ParseUtils.parse(command, ctx);
        } catch (ParseException e){
            //...
        }

        // Do semantic analysis and plan generation
        BaseSemanticAnalyzer sem = SemanticAnalyzerFactory.get(queryState, tree);
        // 解析AST树
        sem.analyze(tree, ctx);
    }
}
```

```java
public abstract class BaseSemanticAnalyzer {
    public void analyze(ASTNode ast, Context ctx) throws SemanticException {
        initCtx(ctx);
        init(true);
        analyzeInternal(ast);
    }
}
```

查看 BaseSemanticAnalyzer 类的子类 SemanticAnalyzer

```java
public class SemanticAnalyzer extends BaseSemanticAnalyzer {
    @Override
    @SuppressWarnings("nls")
    public void analyzeInternal(ASTNode ast) throws SemanticException {
        analyzeInternal(ast, new PlannerContextFactory() {
            @Override
            public PlannerContext create() {
                return new PlannerContext();
            }
        });
    }
}
```

```java
public class SemanticAnalyzer extends BaseSemanticAnalyzer {
    void analyzeInternal(ASTNode ast, PlannerContextFactory pcf) throws SemanticException {
        LOG.info("Starting Semantic Analysis");
        // 1. 语法树 转成 解析树 (query block)
        // 1. Generate Resolved Parse tree from syntax tree
        boolean needsTransform = needsTransform();
        //change the location of position alias process here
        processPositionAlias(ast);
        PlannerContext plannerCtx = pcf.create();
        if (!genResolvedParseTree(ast, plannerCtx)) { // 转换
            return;
        }
        //...
        // 2. 解析树 转成 操作树 （query block --> 逻辑执行计划）
        // 2. Gen OP Tree from resolved Parse Tree
        Operator sinkOp = genOPTree(ast, plannerCtx);

        //...

        // 3. 推断结果集的schema
        // 3. Deduce Resultset Schema
        if (createVwDesc != null && !this.ctx.isCboSucceeded()) {
          resultSchema = convertRowSchemaToViewSchema(opParseCtx.get(sinkOp).getRowResolver());
        } else {
          // resultSchema will be null if
          // (1) cbo is disabled;
          // (2) or cbo is enabled with AST return path (whether succeeded or not,
          // resultSchema will be re-initialized)
          // It will only be not null if cbo is enabled with new return path and it
          // succeeds.
          if (resultSchema == null) {
            resultSchema = convertRowSchemaToResultSetSchema(opParseCtx.get(sinkOp).getRowResolver(),
                HiveConf.getBoolVar(conf, HiveConf.ConfVars.HIVE_RESULTSET_USE_UNIQUE_COLUMN_NAMES));
          }
        }

        // 4. 为优化器和物理编译器，生成的上下文
        // 4. Generate Parse Context for Optimizer & Physical compiler
        copyInfoToQueryProperties(queryProperties);
        ParseContext pCtx = new ParseContext(queryState, opToPartPruner, opToPartList, topOps,
            new HashSet<JoinOperator>(joinContext.keySet()),
            new HashSet<SMBMapJoinOperator>(smbMapJoinContext.keySet()),
            loadTableWork, loadFileWork, columnStatsAutoGatherContexts, ctx, idToTableNameMap, destTableId, uCtx,
            listMapJoinOpsNoReducer, prunedPartitions, tabNameToTabObject, opToSamplePruner,
            globalLimitCtx, nameToSplitSample, inputs, rootTasks, opToPartToSkewedPruner,
            viewAliasToInput, reduceSinkOperatorsAddedByEnforceBucketingSorting,
            analyzeRewrite, tableDesc, createVwDesc, materializedViewUpdateDesc,
            queryProperties, viewProjectToTableSchema, acidFileSinks);

        // Set the semijoin hints in parse context
        pCtx.setSemiJoinHints(parseSemiJoinHint(getQB().getParseInfo().getHintList()));
        // Set the mapjoin hint if it needs to be disabled.
        pCtx.setDisableMapJoin(disableMapJoinWithHint(getQB().getParseInfo().getHintList()));
        }

        // 5. Take care of view creation
        // 6. Generate table access stats if required

        // 7. 执行逻辑优化
        // 7. Perform Logical optimization
        if (LOG.isDebugEnabled()) {
          LOG.debug("Before logical optimization\n" + Operator.toString(pCtx.getTopOps().values()));
        }
        Optimizer optm = new Optimizer();
        optm.setPctx(pCtx);
        optm.initialize(conf);
        // 1.1节  优化 
        pCtx = optm.optimize();
        if (pCtx.getColumnAccessInfo() != null) {
          // set ColumnAccessInfo for view column authorization
          setColumnAccessInfo(pCtx.getColumnAccessInfo());
        }
        if (LOG.isDebugEnabled()) {
          LOG.debug("After logical optimization\n" + Operator.toString(pCtx.getTopOps().values()));
        }

        // 8. Generate column access stats if required - wait until column pruning takes place during optimization


        // 9. 优化物理操作树，转换目标执行引擎  
        // 9. Optimize Physical op tree & Translate to target execution engine (MR,TEZ..)
        if (!ctx.getExplainLogical()) {
          TaskCompiler compiler = TaskCompilerFactory.getCompiler(conf, pCtx);
          compiler.init(queryState, console, db);
          // 1.2节 逻辑计划转物理计划，并优化
          compiler.compile(pCtx, rootTasks, inputs, outputs);
          fetchTask = pCtx.getFetchTask();
        }
        //find all Acid FileSinkOperatorS
        QueryPlanPostProcessor qp = new QueryPlanPostProcessor(rootTasks, acidFileSinks, ctx.getExecutionId());

        // 10. Attach CTAS/Insert-Commit-hooks for Storage Handlers
        
        LOG.info("Completed plan generation");

        // 11. put accessed columns to readEntity
        if (HiveConf.getBoolVar(this.conf, HiveConf.ConfVars.HIVE_STATS_COLLECT_SCANCOLS)) {
          putAccessedColumnsToReadEntity(inputs, columnAccessInfo);
        }
}
```

## 1.1 pCtx = optm.optimize()

逻辑优化

```java
public class SemanticAnalyzer extends BaseSemanticAnalyzer {

    private List<Transform> transformations;
    public void initialize(HiveConf hiveConf) {
        transformations = new ArrayList<Transform>();

        //...
        transformations.add(new PredicatePushDown());
        // 还有很多类似的优化策略，被这样添加进去...
    }

  /**
   * Invoke all the transformations one-by-one, and alter the query plan.
   *
   * @return ParseContext
   * @throws SemanticException
   */
  public ParseContext optimize() throws SemanticException {
    // transformations 就是一个个的优化策略
    for (Transform t : transformations) {
      t.beginPerfLogging();
      // 逐个使用这些优化策略，作用在 pctx 上，再返回它  （那么pctx就包含了各种优化策略）
      pctx = t.transform(pctx);
      t.endPerfLogging(t.toString());
    }
    return pctx;
  }
}
```

这里查看谓词下推的优化策略

```java
public class PredicatePushDown extends Transform {
    @Override
  public ParseContext transform(ParseContext pctx) throws SemanticException {
    pGraphContext = pctx;

    // create a the context for walking operators
    OpWalkerInfo opWalkerInfo = new OpWalkerInfo(pGraphContext);

    Map<Rule, NodeProcessor> opRules = new LinkedHashMap<Rule, NodeProcessor>();
    opRules.put(new RuleRegExp("R1",
      FilterOperator.getOperatorName() + "%"),
      OpProcFactory.getFilterProc());
    //...

    // The dispatcher fires the processor corresponding to the closest matching
    // rule and passes the context along
    Dispatcher disp = new DefaultRuleDispatcher(OpProcFactory.getDefaultProc(),
        opRules, opWalkerInfo);
    GraphWalker ogw = new DefaultGraphWalker(disp);

    // Create a list of topop nodes
    ArrayList<Node> topNodes = new ArrayList<Node>();
    // 把前面放进去的规则都放进 topNodes
    topNodes.addAll(pGraphContext.getTopOps().values());
    // 会遍历、排序规则，判断是
    ogw.startWalking(topNodes, null);

    if (LOG.isDebugEnabled()) {
      LOG.debug("After PPD:\n" + Operator.toString(pctx.getTopOps().values()));
    }
    return pGraphContext;
  }
}
```

## 1.2 compiler.compile(pCtx, rootTasks, inputs, outputs)

物理计划生成、优化

```java
/**
 * TaskCompiler is a the base class for classes that compile
 * operator pipelines into tasks.
 */
public abstract class TaskCompiler {
    public void compile(final ParseContext pCtx,
      final List<Task<? extends Serializable>> rootTasks,
      final HashSet<ReadEntity> inputs, final HashSet<WriteEntity> outputs) throws SemanticException {

        optimizeOperatorPlan(pCtx, inputs, outputs);

//....

        generateTaskTree(rootTasks, pCtx, mvTask, inputs, outputs);
//....
        optimizeTaskPlan(rootTasks, pCtx, ctx);
//....
    }
}
```

```java
public abstract class TaskCompiler {
  /*
   * Called at the beginning of the compile phase to have another chance to optimize the operator plan
   * 在编译阶段的开始调用它，为了有机会能优化操作计划（逻辑执行计划）
   */
  protected void optimizeOperatorPlan(ParseContext pCtxSet, Set<ReadEntity> inputs,
      Set<WriteEntity> outputs) throws SemanticException {
  }

  /*
   * Called after the tasks have been generated to run another round of optimization
   * 在生成任务之后被调用，运行另一轮优化
   */
  protected abstract void optimizeTaskPlan(List<Task<? extends Serializable>> rootTasks,
      ParseContext pCtx, Context ctx) throws SemanticException;

  /*
   * Called to generate the taks tree from the parse context/operator tree
   * 上下文/操作树 转换成 任务树
   */
  protected abstract void generateTaskTree(List<Task<? extends Serializable>> rootTasks, ParseContext pCtx,
      List<Task<MoveWork>> mvTask, Set<ReadEntity> inputs, Set<WriteEntity> outputs) throws SemanticException;
}
```

TaskCompiler 抽象类有三个子类，分别是 MapReduceCompiler\SparkCompiler\TezCompiler,

MapReduceCompiler 没有 optimizeOperatorPlan 的重写方法，而 SparkCompiler\TezCompiler 均有这三个方法的重写方法。

这里查看 MapReduceCompiler 子类

```java
public class MapReduceCompiler extends TaskCompiler {

  @Override
  protected void generateTaskTree(List<Task<? extends Serializable>> rootTasks, ParseContext pCtx,
      List<Task<MoveWork>> mvTask, Set<ReadEntity> inputs, Set<WriteEntity> outputs) throws SemanticException {

    // generate map reduce plans
    ParseContext tempParseContext = getParseContext(pCtx, rootTasks);
    GenMRProcContext procCtx = new GenMRProcContext(
        conf,
        // Must be deterministic order map for consistent q-test output across Java versions
        new LinkedHashMap<Operator<? extends OperatorDesc>, Task<? extends Serializable>>(),
        tempParseContext, mvTask, rootTasks,
        new LinkedHashMap<Operator<? extends OperatorDesc>, GenMapRedCtx>(),
        inputs, outputs);

    // create a walker which walks the tree in a DFS manner while maintaining
    // the operator stack.
    // The dispatcher generates the plan from the operator tree
    Map<Rule, NodeProcessor> opRules = new LinkedHashMap<Rule, NodeProcessor>();
    opRules.put(new RuleRegExp(new String("R1"),
        TableScanOperator.getOperatorName() + "%"),
        new GenMRTableScan1());
    opRules.put(new RuleRegExp(new String("R2"),
        TableScanOperator.getOperatorName() + "%.*" + ReduceSinkOperator.getOperatorName() + "%"),
        new GenMRRedSink1());
    opRules.put(new RuleRegExp(new String("R3"),
        ReduceSinkOperator.getOperatorName() + "%.*" + ReduceSinkOperator.getOperatorName() + "%"),
        new GenMRRedSink2());
    opRules.put(new RuleRegExp(new String("R4"),
        FileSinkOperator.getOperatorName() + "%"),
        new GenMRFileSink1());
    opRules.put(new RuleRegExp(new String("R5"),
        UnionOperator.getOperatorName() + "%"),
        new GenMRUnion1());
    opRules.put(new RuleRegExp(new String("R6"),
        UnionOperator.getOperatorName() + "%.*" + ReduceSinkOperator.getOperatorName() + "%"),
        new GenMRRedSink3());
    opRules.put(new RuleRegExp(new String("R7"),
        MapJoinOperator.getOperatorName() + "%"),
        MapJoinFactory.getTableScanMapJoin());

    // The dispatcher fires the processor corresponding to the closest matching
    // rule and passes the context along
    Dispatcher disp = new DefaultRuleDispatcher(new GenMROperator(), opRules,
        procCtx);

    GraphWalker ogw = new GenMapRedWalker(disp);
    ArrayList<Node> topNodes = new ArrayList<Node>();
    topNodes.addAll(pCtx.getTopOps().values());
    ogw.startWalking(topNodes, null);
  }
}
```


```java
public class MapReduceCompiler extends TaskCompiler {
      @Override
      protected void optimizeTaskPlan(List<Task<? extends Serializable>> rootTasks,
          ParseContext pCtx, Context ctx) throws SemanticException {
        // reduce sink does not have any kids - since the plan by now has been
        // broken up into multiple
        // tasks, iterate over all tasks.
        // For each task, go over all operators recursively
        for (Task<? extends Serializable> rootTask : rootTasks) {
          breakTaskTree(rootTask);
        }


        PhysicalContext physicalContext = new PhysicalContext(conf,
            getParseContext(pCtx, rootTasks), ctx, rootTasks, pCtx.getFetchTask());
        PhysicalOptimizer physicalOptimizer = new PhysicalOptimizer(
            physicalContext, conf);
        physicalOptimizer.optimize();

      }
  }
```

```java
/**
 * A hierarchy physical optimizer, which contains a list of
 * PhysicalPlanResolver. Each resolver has its own set of optimization rule.
 */
public class PhysicalOptimizer {
 /**
   * invoke all the resolvers one-by-one, and alter the physical plan.
   *
   * @return PhysicalContext
   * @throws HiveException
   */
  public PhysicalContext optimize() throws SemanticException {
    // resolvers 包含了各种优化规则
    for (PhysicalPlanResolver r : resolvers) {
      // 逐个应用这些优化规则
      pctx = r.resolve(pctx);
    }
    return pctx;
  }
}
```