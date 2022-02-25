---
layout:     post
title:      Flink SQL 代码自动生成之 janino
subtitle:   
date:       2021-06-26
author:     danner
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Flink
    - SQL
---

SQL 加速有两种方式：代码自动生成和向量化，本文分析Flink SQL 中代码自动生成方式。

`Janino` 是一个极小、极快的 开源Java 编译器（Janino is a super-small, super-fast Java™ compiler.）。`Janino` 不仅可以像 JAVAC 一样将 Java 源码文件编译为字节码文件，还可以编译内存中的 Java 表达式、块、类和源码文件，加载字节码并在 JVM 中直接执行。Janino 同样可以用于静态代码分析和代码操作。

首先以一个简单案例使用 `Janino`，接着以 `Group By` 和 `Lookup` 这两个语义来分析在Flink SQL 中的运用。

### HelloWorld

项目中导入 `janino`包，主体代码如下

```java
public static void main(String[] args) {
    ScriptEvaluator evaluator = new ScriptEvaluator();
    try {
        evaluator.cook("static void method() {\n" +
                "    System.out.println(\"print hello world\");\n" +
                "}\n" +
                "\n" +
                "method();");
        evaluator.evaluate(null);
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

与平常写代码一样，定义了一个函数，代码体传入`cook` 函数中，接着调用`evaluate` 就会执行代码，非常方便。

### Flink SQL

```java
// org.apache.flink.table.runtime.generated.GeneratedClass
/** Create a new instance of this generated class. */
@SuppressWarnings("unchecked")
public T newInstance(ClassLoader classLoader) {
    try {
        return compile(classLoader)
                .getConstructor(Object[].class)
                // Because Constructor.newInstance(Object... initargs), we need to load
                // references into a new Object[], otherwise it cannot be compiled.
                .newInstance(new Object[] {references});
    } catch (Exception e) {
        throw new RuntimeException(
                "Could not instantiate generated class '" + className + "'", e);
    }
}
/**
* Compiles the generated code, the compiled class will be cached in the {@link GeneratedClass}.
*/
public Class<T> compile(ClassLoader classLoader) {
    if (compiledClass == null) {
        // cache the compiled class
        compiledClass = CompileUtils.compile(classLoader, className, code);
    }
    return compiledClass;
}

// org.apache.flink.table.runtime.generated.CompileUtils
public static <T> Class<T> compile(ClassLoader cl, String name, String code) {
    try {
        Cache<ClassLoader, Class> compiledClasses =
                COMPILED_CACHE.get(
                        // "code" as a key should be sufficient as the class name
                        // is part of the Java code
                        code,
                        () ->
                                CacheBuilder.newBuilder()
                                        .maximumSize(5)
                                        .weakKeys()
                                        .softValues()
                                        .build());
        return compiledClasses.get(cl, () -> doCompile(cl, name, code));
    } catch (Exception e) {
        throw new FlinkRuntimeException(e.getMessage(), e);
    }
}
private static <T> Class<T> doCompile(ClassLoader cl, String name, String code) {
    checkNotNull(cl, "Classloader must not be null.");
    CODE_LOG.debug("Compiling: {} \n\n Code:\n{}", name, code);
    SimpleCompiler compiler = new SimpleCompiler();
    compiler.setParentClassLoader(cl);
    try {
        compiler.cook(code);
    } catch (Throwable t) {
        System.out.println(addLineNumber(code));
        throw new InvalidProgramException(
                "Table program cannot be compiled. This is a bug. Please file an issue.", t);
    }
    try {
        //noinspection unchecked
        return (Class<T>) compiler.getClassLoader().loadClass(name);
    } catch (ClassNotFoundException e) {
        throw new RuntimeException("Can not load class " + name, e);
    }
}
```

Flink SQL 整个流程可大致分为两步

- 编译Code 并加载类

- 类实例化成对象，供后续调用

至于Code 如何生成，看下面的案例

#### Group By

```java
// org.apache.flink.table.planner.codegen.agg.AggsHandlerCodeGenerator#generateAggsHandler
// Generate [[GeneratedAggsHandleFunction]] with the given function name and aggregate infos.
def generateAggsHandler(
      name: String,
      aggInfoList: AggregateInfoList): GeneratedAggsHandleFunction = {

    initialAggregateInformation(aggInfoList)

    // generates all methods body first to add necessary reuse code to context
    val createAccumulatorsCode = genCreateAccumulators()
    val getAccumulatorsCode = genGetAccumulators()
    val setAccumulatorsCode = genSetAccumulators()
    val resetAccumulatorsCode = genResetAccumulators()
    val accumulateCode = genAccumulate()
    val retractCode = genRetract()
    val mergeCode = genMerge()
    val getValueCode = genGetValue()

    val functionName = newName(name)

    val functionCode =
      j"""
        public final class $functionName implements $AGGS_HANDLER_FUNCTION {

          ${ctx.reuseMemberCode()}

          private $STATE_DATA_VIEW_STORE store;

          public $functionName(java.lang.Object[] references) throws Exception {
            ${ctx.reuseInitCode()}
          }

          private $RUNTIME_CONTEXT getRuntimeContext() {
            return store.getRuntimeContext();
          }

          @Override
          public void open($STATE_DATA_VIEW_STORE store) throws Exception {
            this.store = store;
            ${ctx.reuseOpenCode()}
          }

          @Override
          public void accumulate($ROW_DATA $ACCUMULATE_INPUT_TERM) throws Exception {
            $accumulateCode
          }

          @Override
          public void retract($ROW_DATA $RETRACT_INPUT_TERM) throws Exception {
            $retractCode
          }

          @Override
          public void merge($ROW_DATA $MERGED_ACC_TERM) throws Exception {
            $mergeCode
          }

          @Override
          public void setAccumulators($ROW_DATA $ACC_TERM) throws Exception {
            $setAccumulatorsCode
          }

          @Override
          public void resetAccumulators() throws Exception {
            $resetAccumulatorsCode
          }

          @Override
          public $ROW_DATA getAccumulators() throws Exception {
            $getAccumulatorsCode
          }

          @Override
          public $ROW_DATA createAccumulators() throws Exception {
            $createAccumulatorsCode
          }

          @Override
          public $ROW_DATA getValue() throws Exception {
            $getValueCode
          }

          @Override
          public void cleanup() throws Exception {
            ${ctx.reuseCleanupCode()}
          }

          @Override
          public void close() throws Exception {
            ${ctx.reuseCloseCode()}
          }
        }
      """.stripMargin

    new GeneratedAggsHandleFunction(functionName, functionCode, ctx.references.toArray)
  }
```

`functionCode` 就是聚合函数的代码，事先定义函数，然后生成对应的函数体。

- createAccumulatorsCode

- getAccumulatorsCode

- setAccumulatorsCode

- resetAccumulatorsCode

- accumulateCode

- retractCode

- mergeCode

- getValueCode

自动生成的代码最终结果，可以看之前分分析 group by 的文章

#### Lookup

关联维表，无法事先定义好关联条件和需要的字段。定义后SQL，自动生成代码获取维表字段。

代码生成函数`org.apache.flink.table.planner.codegen.LookupJoinCodeGenerator#generateLookupFunction`

最终结果案例

```java

public class LookupFunction$4
        extends org.apache.flink.api.common.functions.RichFlatMapFunction {

    private transient org.apache.flink.connector.jdbc.table.JdbcRowDataLookupFunction function_org$apache$flink$connector$jdbc$table$JdbcRowDataLookupFunction$3b97049b38ddb18c3b67d5e4ba11ff48;
    private TableFunctionResultConverterCollector$2 resultConverterCollector$3 = null;

    public LookupFunction$4(Object[] references) throws Exception {
        function_org$apache$flink$connector$jdbc$table$JdbcRowDataLookupFunction$3b97049b38ddb18c3b67d5e4ba11ff48 = (((org.apache.flink.connector.jdbc.table.JdbcRowDataLookupFunction) references[0]));
    }
    
    @Override
    public void open(org.apache.flink.configuration.Configuration parameters) throws Exception {
        
        function_org$apache$flink$connector$jdbc$table$JdbcRowDataLookupFunction$3b97049b38ddb18c3b67d5e4ba11ff48.open(new org.apache.flink.table.functions.FunctionContext(getRuntimeContext()));
                
        
        resultConverterCollector$3 = new TableFunctionResultConverterCollector$2();
        resultConverterCollector$3.setRuntimeContext(getRuntimeContext());
        resultConverterCollector$3.open(new org.apache.flink.configuration.Configuration());
              
        function_org$apache$flink$connector$jdbc$table$JdbcRowDataLookupFunction$3b97049b38ddb18c3b67d5e4ba11ff48.setCollector(resultConverterCollector$3);
        
    }

    @Override
    public void flatMap(Object _in1, org.apache.flink.util.Collector c) throws Exception {
        org.apache.flink.table.data.RowData in1 = (org.apache.flink.table.data.RowData) _in1;
        
        org.apache.flink.table.data.binary.BinaryStringData field$0;
        boolean isNull$0;
        isNull$0 = in1.isNullAt(0);
        field$0 = org.apache.flink.table.data.binary.BinaryStringData.EMPTY_UTF8;
        if (!isNull$0) {
        field$0 = ((org.apache.flink.table.data.binary.BinaryStringData) in1.getString(0));
        }
        
        resultConverterCollector$3.setCollector(c);
                
        if (isNull$0) {
        // skip
        } else {
            function_org$apache$flink$connector$jdbc$table$JdbcRowDataLookupFunction$3b97049b38ddb18c3b67d5e4ba11ff48.eval(isNull$0 ? null : ((org.apache.flink.table.data.binary.BinaryStringData) field$0));
        }               
    }

    @Override
    public void close() throws Exception {        
        function_org$apache$flink$connector$jdbc$table$JdbcRowDataLookupFunction$3b97049b38ddb18c3b67d5e4ba11ff48.close();                
    }


    public class TableFunctionResultConverterCollector$2 extends org.apache.flink.table.runtime.collector.WrappingCollector {
                    
            public TableFunctionResultConverterCollector$2() throws Exception {
                
            }
    
            @Override
            public void open(org.apache.flink.configuration.Configuration parameters) throws Exception {
                
            }
    
            @Override
            public void collect(Object record) throws Exception {
                org.apache.flink.table.data.RowData externalResult$1 = (org.apache.flink.table.data.RowData) record;                                
                                
                if (externalResult$1 != null) {
                outputResult(externalResult$1);
                }
                
            }
    
            @Override
            public void close() {
                try {
                
                } catch (Exception e) {
                    throw new RuntimeException(e);
                }
            }
        }        
    }

```

关联逻辑在`function_orgapacheflinkconnectorjdbctableJdbcRowDataLookupFunction$3b97049b38ddb18c3b67d5e4ba11ff48` ，由用户定义。以`JDBC`为例，就是 `JdbcRowDataLookupFunction`

自动生成的代码里，主要函数是初始化/关联

- `open`：调用`JdbcRowDataLookupFunction` 开始初始化

- `FlatMap`：调用`JdbcRowDataLookupFunction.eval` 关联外部数据库；这里解释了为什么每个function 都需要 eval 函数。



## 参考资料

[Janino框架初识与使用教程_IT- 研究者-CSDN博客](https://blog.csdn.net/inrgihc/article/details/104399439)
