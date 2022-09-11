找到 `CliDriver` 这个类的 `main` 方法

```java
public class CliDriver {
    public static void main(String[] args) throws Exception {
        int ret = new CliDriver().run(args);
        System.exit(ret);
    }

    /*
         在 `01_程序入口.md` 中，启动了 bin/hive 了，CliDriver 的 main 方法会随着命令行的输入，
         会在 run 方法中执行如下内容：   
             1. 解析并配置在执行 `bin/hive` 时，指定的参数，比如 --hiveconf  -f 等，
             2. 定义标准输入输出流
             3. 然后执行 executeDriver 方法
     */
    public int run(String[] args) throws Exception {

        OptionsProcessor oproc = new OptionsProcessor();
        // 解析 --hiveconf --hivevar 参数，把 --hiveconf 设置成系统属性，把 --hivevar 参数添加到一个hashmap里
        if (!oproc.process_stage1(args)) {
            return 1;
        }

        // 初始化日志相关
        // NOTE: It is critical to do this here so that log4j is reinitialized
        // before any of the other core hive classes are loaded
        boolean logInitFailed = false;
        String logInitDetailMessage;
        try {
            logInitDetailMessage = LogUtils.initHiveLog4j();
        } catch (LogInitializationException e) {
            logInitFailed = true;
            logInitDetailMessage = e.getMessage();
        }

        CliSessionState ss = new CliSessionState(new HiveConf(SessionState.class));
        // 标准输入输出以及错误输出流的定义，后续需要输入 HQL 以及打印控制台信息（比如，执行信息）
        ss.in = System.in;
        try {
            ss.out = new PrintStream(System.out, true, "UTF-8");
            ss.info = new PrintStream(System.err, true, "UTF-8");
            ss.err = new CachingPrintStream(System.err, true, "UTF-8");
        } catch (UnsupportedEncodingException e) {
            return 3;
        }

        // 解析像 -e -f 参数，分别赋给 CliSessionState 类中的对应属性，
        // 也会再次解析 --hiveconf 参数，赋给 CliSessionState 类中的 cmdProperties 属性
        if (!oproc.process_stage2(ss)) {
            return 2;
        }

        if (!ss.getIsSilent()) {
            if (logInitFailed) {
                System.err.println(logInitDetailMessage);
            } else {
                SessionState.getConsole().printInfo(logInitDetailMessage);
            }
        }

        // 设置 --hiveconf 参数
        // set all properties specified via command line
        HiveConf conf = ss.getConf();
        for (Map.Entry<Object, Object> item : ss.cmdProperties.entrySet()) {
            conf.set((String) item.getKey(), (String) item.getValue());
            ss.getOverriddenConfigurations().put((String) item.getKey(), (String) item.getValue());
        }

        // read prompt configuration and substitute variables.
        // CLIPROMPT("hive.cli.prompt", "hive") 
        // 这个就是 `hive (default)> desc emp;` 中的 hive
        prompt = conf.getVar(HiveConf.ConfVars.CLIPROMPT);
        prompt = new VariableSubstitution(new HiveVariableSource() {
            @Override
            public Map<String, String> getHiveVariable() {
                return SessionState.get().getHiveVariables();
            }
        }).substitute(conf, prompt);
        // 和 hive 一样长的空格（没写任何语句，换行时）
        prompt2 = spacesForString(prompt);

        if (HiveConf.getBoolVar(conf, ConfVars.HIVE_CLI_TEZ_SESSION_ASYNC)) {
            // Start the session in a fire-and-forget manner. When the asynchronously initialized parts of
            // the session are needed, the corresponding getters and other methods will wait as needed.
            SessionState.beginStart(ss, console);
        } else {
            SessionState.start(ss);
        }

        ss.updateThreadName();

        // Create views registry
        HiveMaterializedViewsRegistry.get().init();

        // execute cli driver work
        try {
            // 这里
            return executeDriver(ss, conf, oproc);
        } finally {
            ss.resetThreadName();
            ss.close();
        }
    }
}
```

```java
public class CliDriver {
    /**
     * Execute the cli work
     * @param ss CliSessionState of the CLI driver
     * @param conf HiveConf for the driver session
     * @param oproc Operation processor of the CLI invocation
     * @return status of the CLI command execution
     * @throws Exception
     */
    /*
           判断执行的是不是 hive -e 或 hive -f ，如果是就执行对应的sql语句或sql文件，
            否则就解析从命令行读取的sql语句，并调用 processLine 方法执行。 
     */
    private int executeDriver(CliSessionState ss, HiveConf conf, OptionsProcessor oproc)
            throws Exception {

        CliDriver cli = new CliDriver();
        // 设置通过 --hivevar 或 --define 定义的变量
        cli.setHiveVariables(oproc.getHiveVariables());

        // 先取出来使用的数据库名字，再执行 use 命令，使用这个数据库
        // use the specified database if specified
        cli.processSelectDatabase(ss);

        // Execute -i init files (always in silent mode)
        cli.processInitFiles(ss);

        // 处理 hive -e 指定的sql，不可以使用 Ctrl+C 终止sql执行
        // execString: -e 参数指定的sql
        if (ss.execString != null) {
            int cmdProcessStatus = cli.processLine(ss.execString);
            return cmdProcessStatus;
        }

        // 处理 hive -f 指定的sql文件
        // 读取文件内容后，还是调用 processLine 执行sql
        try {
            if (ss.fileName != null) {
                return cli.processFile(ss.fileName);
            }
        } catch (FileNotFoundException e) {
            System.err.println("Could not open input file for reading. (" + e.getMessage() + ")");
            return 3;
        }
        // HIVE_EXECUTION_ENGINE("hive.execution.engine", "mr")
        // 判断如果执行引擎是mr，会打出弃用警告 Hive-on-MR is deprecated in Hive 2.....
        if ("mr".equals(HiveConf.getVar(conf, ConfVars.HIVE_EXECUTION_ENGINE))) {
            console.printInfo(HiveConf.generateMrDeprecationWarning());
        }

        // 设置一个读取控制台内容的读取器
        setupConsoleReader();

        String line;
        int ret = 0;
        String prefix = "";
        // 默认就是 `hive (default)> desc emp;` 中的 default
        String curDB = getFormattedDb(conf, ss);
        // curPrompt 就是 hive (default)
        String curPrompt = prompt + curDB;
        // 当前数据库长度的空格
        String dbSpaces = spacesForString(curDB);

        // 开始读取sql语句
        // protected ConsoleReader reader;
        while ((line = reader.readLine(curPrompt + "> ")) != null) {
            if (!prefix.equals("")) {
                prefix += '\n';
            }
            // sql中的注释，就跳过
            if (line.trim().startsWith("--")) {
                continue;
            }
            if (line.trim().endsWith(";") && !line.trim().endsWith("\\;")) {
                // 追加当前行的sql语句
                line = prefix + line;
                // 处理sql, 可以使用 Ctrl+C 终止sql执行
                ret = cli.processLine(line, true);
                // 处理完这些sql后，重置这些属性
                prefix = "";
                curDB = getFormattedDb(conf, ss);
                curPrompt = prompt + curDB;
                dbSpaces = dbSpaces.length() == curDB.length() ? dbSpaces : spacesForString(curDB);
            }
            // 还没有到语句的结尾，也就是还没遇到分号
            else {
                // 追加当前行的sql语句
                prefix = prefix + line;
                /*
                   输入一行sql语句后，不输分号，而是使用换行，那么下行的头， 也就是这种形式 ：
                   hive (default)> select *
                                 > 
                 */
                curPrompt = prompt2 + dbSpaces;
                continue;
            }
        }

        return ret;
    }
}
```