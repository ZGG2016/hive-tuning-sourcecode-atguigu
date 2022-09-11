
在 `02_参数解析与HQL的读取.md` 中，解析并设置了在执行 `bin/hive` 时指定的参数，也读取到了sql语句。

下面开始处理sql语句。

```java
public class CliDriver {
    
        public int processLine(String line) {
            return processLine(line, false);
        }

        /**
         * Processes a line of semicolon separated commands
         * 处理分号分隔的命令行
         *
         * @param line
         *          The commands to process
         * @param allowInterrupting
         *          When true the function will handle SIG_INT (Ctrl+C) by interrupting the processing and
         *          returning -1
         *          如果可以使用 (Ctrl+C) 终止命令，就返回true
         * @return 0 if ok
         */
        /*
                根据在控制台输入的内容，使用不同的解析方案:
                  - exit quit
                  - sql文件
                  - shell命令
                  - 普通的sql语句：会按分号切分成一个个独立的sql语句，对每条sql语句，调用processCmd执行
         */
        public int processLine(String line, boolean allowInterrupting) {
            SignalHandler oldSignal = null;
            Signal interruptSignal = null;

            // 如果在sql执行过程中，使用了 (Ctrl+C) ，那么就终止执行sql语句
            if (allowInterrupting) {
                // Remember all threads that were running at the time we started line processing.
                // Hook up the custom Ctrl+C handler while processing this line
                interruptSignal = new Signal("INT");
                oldSignal = Signal.handle(interruptSignal, new SignalHandler() {
                    private boolean interruptRequested;

                    @Override
                    public void handle(Signal signal) {
                        boolean initialRequest = !interruptRequested;
                        interruptRequested = true;

                        // Kill the VM on second ctrl+c
                        if (!initialRequest) {
                            console.printInfo("Exiting the JVM");
                            System.exit(127);
                        }

                        // Interrupt the CLI thread to stop the current statement and return
                        // to prompt
                        console.printInfo("Interrupting... Be patient, this might take some time.");
                        console.printInfo("Press Ctrl+C again to kill JVM");

                        // First, kill any running MR jobs
                        HadoopJobExecHelper.killRunningJobs();
                        TezJobExecHelper.killRunningJobs();
                        HiveInterruptUtils.interrupt();
                    }
                });
            }

            try {
                int lastRet = 0, ret = 0;

                // 按分号，切分成一个个独立的sql语句
                // we can not use "split" function directly as ";" may be quoted
                List<String> commands = splitSemiColon(line);

                String command = "";
                // 处理每条sql语句
                for (String oneCmd : commands) {
                    // 赋给一个新的 command 变量
                    if (StringUtils.endsWith(oneCmd, "\\")) {
                        command += StringUtils.chop(oneCmd) + ";";
                        continue;
                    } else {
                        command += oneCmd;
                    }
                    // 空语句就跳过
                    if (StringUtils.isBlank(command)) {
                        continue;
                    }
                    // 开始处理sql语句
                    ret = processCmd(command);
                    command = "";
                    lastRet = ret;
                    boolean ignoreErrors = HiveConf.getBoolVar(conf, HiveConf.ConfVars.CLIIGNOREERRORS);
                    if (ret != 0 && !ignoreErrors) {
                        return ret;
                    }
                }
                return lastRet;
            } finally {
                //...
            }
        }
}
```

```java
public class CliDriver {
    
    public int processCmd(String cmd) {
        CliSessionState ss = (CliSessionState) SessionState.get();
        ss.setLastCommand(cmd);
    
        ss.updateThreadName();
    
        // Flush the print stream, so it doesn't include output from the last command
        ss.err.flush();
        // 去掉语句里的注释
        String cmd_trimmed = HiveStringUtils.removeComments(cmd).trim();
        // 按空格切分语句，select id from test ->  [select,id,from,test]
        String[] tokens = tokenizeCmd(cmd_trimmed);
        int ret = 0;
        
        // 如果语句是 quit 或 exit，就退出客户端
        if (cmd_trimmed.toLowerCase().equals("quit") || cmd_trimmed.toLowerCase().equals("exit")) {
    
          // if we have come this far - either the previous commands
          // are all successful or this is command line. in either case
          // this counts as a successful run
          ss.close();
          System.exit(0);
    
        } 
        // 如果执行的是 `source /root/test.sql;` 这种语句
        else if (tokens[0].equalsIgnoreCase("source")) {
          String cmd_1 = getFirstCmd(cmd_trimmed, tokens[0].length());
          // 变量替换，就是使用具体的值替换在sql中的变量
          cmd_1 = new VariableSubstitution(new HiveVariableSource() {
            @Override
            public Map<String, String> getHiveVariable() {
              return SessionState.get().getHiveVariables();
            }
          }).substitute(ss.getConf(), cmd_1);
    
          File sourceFile = new File(cmd_1);
          if (! sourceFile.isFile()){
            console.printError("File: "+ cmd_1 + " is not a file.");
            ret = 1;
          } else {
            try {
                // 处理文件中的sql语句
              ret = processFile(cmd_1);
            } catch (IOException e) {
              console.printError("Failed processing file "+ cmd_1 +" "+ e.getLocalizedMessage(),
                stringifyException(e));
              ret = 1;
            }
          }
        } 
        // 如果执行的是 shell 命令 ，比如， hive (default)> !echo "aa";
        else if (cmd_trimmed.startsWith("!")) {
          // for shell commands, use unstripped command
            // 截取出来 shell 命令
          String shell_cmd = cmd.trim().substring(1);
            // 变量替换
          shell_cmd = new VariableSubstitution(new HiveVariableSource() {
            @Override
            public Map<String, String> getHiveVariable() {
              return SessionState.get().getHiveVariables();
            }
          }).substitute(ss.getConf(), shell_cmd);
    
          // shell_cmd = "/bin/bash -c \'" + shell_cmd + "\'";
          try {
              // 这里的 ss.out, ss.err 就是在 run 方法中定义的输入输出流
            ShellCmdExecutor executor = new ShellCmdExecutor(shell_cmd, ss.out, ss.err);
            ret = executor.execute();
            if (ret != 0) {
              console.printError("Command failed with exit code = " + ret);
            }
          } catch (Exception e) {
            console.printError("Exception raised from Shell command " + e.getLocalizedMessage(),
                stringifyException(e));
            ret = 1;
          }
        }  
        // 就是普通的sql语句
        else { // local mode
          try {
            try (CommandProcessor proc = CommandProcessorFactory.get(tokens, (HiveConf) conf)) {
              if (proc instanceof IDriver) {
                // Let Driver strip comments using sql parser 
                  // 这里
                ret = processLocalCmd(cmd, proc, ss);
              } else {
                ret = processLocalCmd(cmd_trimmed, proc, ss);
              }
            }
          } catch (SQLException e) {
            console.printError("Failed processing command " + tokens[0] + " " + e.getLocalizedMessage(),
              org.apache.hadoop.util.StringUtils.stringifyException(e));
            ret = 1;
          }
          catch (Exception e) {
            throw new RuntimeException(e);
          }
        }
    
        ss.resetThreadName();
        return ret;
      }
}
```