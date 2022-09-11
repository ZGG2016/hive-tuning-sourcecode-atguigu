
在 `05_进入编译HQL代码.md` 中，进入到了编译代码部分，这里开始分析解析器解析sql语句

```java
public class Driver implements IDriver {
    private Context ctx;

    private void compile(String command, boolean resetTaskIds, boolean deferClose) throws CommandProcessorResponse {

        if (ctx == null) {
            ctx = new Context(conf);
            setTriggerContext(queryId);
        }

        ctx.setHiveTxnManager(queryTxnMgr);
        ctx.setStatsSource(statsSource);
        ctx.setCmd(command);
        ctx.setHDFSCleanup(true);
        
        ASTNode tree;
        try {
            // 生成AST树
            tree = ParseUtils.parse(command, ctx);
        } catch (ParseException e){
            //...
        }
        //...
    }
}
```

```java
public final class ParseUtils {
    /** Parses the Hive query. */
    public static ASTNode parse(String command, Context ctx) throws ParseException {
        return parse(command, ctx, null);
    }

    /** Parses the Hive query. */
    public static ASTNode parse(
            String command, Context ctx, String viewFullyQualifiedName) throws ParseException {
        ParseDriver pd = new ParseDriver();
        ASTNode tree = pd.parse(command, ctx, viewFullyQualifiedName);
        tree = findRootNonNullToken(tree);
        handleSetColRefs(tree);
        return tree;
    }
}
```

```java
public class ParseDriver {
    /**
     * Parses a command, optionally assigning the parser's token stream to the
     * given context.
     *
     * @param command
     *          command to parse
     *
     * @param ctx
     *          context with which to associate this parser's token stream, or
     *          null if either no context is available or the context already has
     *          an existing stream
     *
     * @return parsed AST
     */
    public ASTNode parse(String command, Context ctx, String viewFullyQualifiedName)
            throws ParseException {
        if (LOG.isDebugEnabled()) {
            LOG.debug("Parsing command: " + command);
        }
        /*
                Hive 使用 Antlr 实现 SQL 的词法和语法解析，
                编写语法文件，定义词法和语法替换规则，然后就使用这个文件解析，
                Antlr 完成词法分析、语法分析、语义分析、中间代码生成的过程。
         */
        // 构建词法解析器 
        HiveLexerX lexer = new HiveLexerX(new ANTLRNoCaseStringStream(command));
        // 将 HQL 中的关键词替换为 Token 
        TokenRewriteStream tokens = new TokenRewriteStream(lexer);
        if (ctx != null) {
            if (viewFullyQualifiedName == null) {
                // Top level query
                ctx.setTokenRewriteStream(tokens);
            } else {
                // It is a view
                ctx.addViewTokenRewriteStream(viewFullyQualifiedName, tokens);
            }
            lexer.setHiveConf(ctx.getConf());
        }
        // 构建语法解析器
        HiveParser parser = new HiveParser(tokens);
        if (ctx != null) {
            parser.setHiveConf(ctx.getConf());
        }
        parser.setTreeAdaptor(adaptor);
        HiveParser.statement_return r = null;
        try {
            // 进行语法解析，生成最终的 AST
            r = parser.statement();
        } catch (RecognitionException e) {
            e.printStackTrace();
            throw new ParseException(parser.errors);
        }

        if (lexer.getErrors().size() == 0 && parser.errors.size() == 0) {
            LOG.debug("Parse Completed");
        } else if (lexer.getErrors().size() != 0) {
            throw new ParseException(lexer.getErrors());
        } else {
            throw new ParseException(parser.errors);
        }
        
        // 得到AST树
        ASTNode tree = (ASTNode) r.getTree();
        tree.setUnknownTokenBoundaries();
        return tree;
    }
}
```