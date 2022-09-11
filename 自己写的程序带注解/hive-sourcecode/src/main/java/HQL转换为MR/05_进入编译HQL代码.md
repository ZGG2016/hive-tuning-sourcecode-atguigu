
在 `03_读取HQL语句分类解析.md` 中，根据在控制台输入的内容，使用不同的解析方案。

在 `04_控制台打印信息介绍.md` 中，分析了输出的打印内容。

这里进入编译HSQL的代码

**查看 IDriver 接口的实现类 Driver**

```java
public class Driver implements IDriver {
    
    @Override
    public CommandProcessorResponse run(String command) {
        return run(command, false);
    }

    // 第二个参数表示是否已经编译完成，如果是true，就不会编译，而是直接提交任务
    public CommandProcessorResponse run(String command, boolean alreadyCompiled) {

        try {
            runInternal(command, alreadyCompiled);
            return createProcessorResponse(0);
        } catch (CommandProcessorResponse cpr) {
            //....
        }
    }
}
```

```java
public class Driver implements IDriver {
    private void runInternal(String command, boolean alreadyCompiled) throws CommandProcessorResponse {
        // ....
        try {
            // 根据编译情况，调整Driver的状态
            if (alreadyCompiled) {
                // 传来的 alreadyCompiled 是false，所以不走这部分
                if (lDrvState.driverState == DriverState.COMPILED) {
                    lDrvState.driverState = DriverState.EXECUTING;
                } else {
                    errorMessage = "FAILED: Precompiled query has been cancelled or closed.";
                    console.printError(errorMessage);
                    throw createProcessorResponse(12);
                }
            } else {
                lDrvState.driverState = DriverState.COMPILING;
            }


            try {
                //....

                PerfLogger perfLogger = null;
                if (!alreadyCompiled) {
                    // compile internal will automatically reset the perf logger  
                    // 开始编译
                    compileInternal(command, true);
                    // then we continue to use this perf logger
                    perfLogger = SessionState.getPerfLogger();
                } else {
                    // reuse existing perf logger.
                    perfLogger = SessionState.getPerfLogger();
                    // Since we're reusing the compiled plan, we need to update its start time for current run
                    plan.setQueryStartTime(perfLogger.getStartTime(PerfLogger.DRIVER_RUN));
                }
                //....
                try {
                    // 执行sql
                    execute();
                } catch (CommandProcessorResponse cpr) {
                    //....
                }
            }
        }
    }
}
```

```java
public class Driver implements IDriver {
    
    private void compileInternal(String command, boolean deferClose) throws CommandProcessorResponse {
        //....
        try {
            // 这里
            compile(command, true, deferClose);
        } catch (CommandProcessorResponse cpr) {
            //....
        }
    }
}
```

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