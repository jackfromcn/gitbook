# Q5_Graphql starter 4.0.0 升级到 5.0.2 中的问题

> 背景：项目中引入了 `graphql starter`，其不光前端会使用，在某些逻辑处理上，本系统内也调用了。



## 问题

### schema 定义循环依赖问题

**GraphQL** 执行引擎会检测其执行引擎循环依赖。即，当前 **GraphQLSchemaProvider** 初始化时，同时也依赖了自己的 **GraphQLSchemaProvider**，就会提示循环依赖。



### spring 依赖中的问题

**Spring IOC** 解决了各 **Bean** 之间的依赖问题，但是即使如此，如果项目内部某个 **Service** 依赖了 **GraphQL** 定义，其相互依赖时就会出现问题。

具体错误见：[Q7_EnableWebMvcConfiguration No ServletContext set](Q7_EnableWebMvcConfiguration_No_ServletContext_set.md)



### API 定义修改

`graphql starter` 版本依赖以的 `graphql-java-tools` 版本有所不同。因此 **API** 定义也有些修改。

具体体现到系统内通过执行引擎直接调用 **GrapQL** 定义的资源。

#### 4.0.0 版本

`pom.xml` 依赖：<br/>

```xml
<dependency>
    <groupId>com.graphql-java</groupId>
    <artifactId>graphql-spring-boot-starter</artifactId>
    <version>4.0.0</version>
</dependency>
<dependency>
    <groupId>com.graphql-java</groupId>
    <artifactId>graphql-java-tools</artifactId>
    <version>4.3.0</version>
</dependency>
```

实现内部服务调用的 **Java** 代码：<br/>

```java
import graphql.*;
import graphql.execution.instrumentation.Instrumentation;
import graphql.execution.instrumentation.NoOpInstrumentation;
import graphql.execution.preparsed.NoOpPreparsedDocumentProvider;
import graphql.execution.preparsed.PreparsedDocumentProvider;
import graphql.schema.GraphQLSchema;
import graphql.servlet.*;
import org.apache.commons.lang3.StringUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import org.springframework.util.CollectionUtils;
import org.springframework.util.PathMatcher;

import java.util.*;
import java.util.stream.Collectors;

/**
 *
 * @author wencheng
 * @date 2019/3/28
 */
@Component
public class IGraphQL {

    @Autowired
    private PathMatcher pathMatcher;

    @Autowired
    ExecutionStrategyProvider executionStrategyProvider;

    @Autowired(required = false)
    private Instrumentation instrumentation;

    @Autowired(required = false)
    private GraphQLContextBuilder contextBuilder;

    @Autowired(required = false)
    private GraphQLRootObjectBuilder rootObjectBuilder;

    @Autowired(required = false)
    private PreparsedDocumentProvider preparsedDocumentProvider;

    private GraphQLSchema schema;

    private Object rootObject;

    private GraphQLContext context;

    public ExecutionResult execute(String query, String operationName, Map<String, Object> variables) {
        return graphQL().execute(new ExecutionInput(query, operationName, context, rootObject, transformVariables(schema, query, variables)));
    }

    /**
     * 执行查询,抛出运行中的异常
     *  注意: 抛出的异常 —— errorPath 为 null 或者 忽略指定属性字段外的字段异常
     * @param query
     * @param operationName
     * @param variables
     * @param ignorePatterns
     * @return
     */
    public ExecutionResult executeThrowsEAndIgnorePatterns(String query, String operationName, Map<String, Object> variables, Set<String> ignorePatterns) {
        ExecutionResult executionResult = graphQL().execute(new ExecutionInput(query, operationName, context, rootObject, transformVariables(schema, query, variables)));
        List<GraphQLError> errors = executionResult.getErrors();
        if (!CollectionUtils.isEmpty(errors)) {
            errors = errors.stream()
                    .filter(e -> {
                        String errorPath = errorPath(e);
                        return Objects.nonNull(errorPath) && !matchPath(ignorePatterns, errorPath);
                    }).collect(Collectors.toList());
            String message = errors.stream()
                    .map(this::extractException)
                    .filter(Optional::isPresent)
                    .map(Optional::get)
                    .map(Throwable::getMessage)
                    .collect(Collectors.joining(";"));
            if (StringUtils.isNotBlank(message)) {
                throw new ServiceException(message);
            }
        }
        return executionResult;
    }

    /**
     * 执行查询,抛出运行中的异常
     *  注意: 抛出的异常 —— errorPath 为 null 或者 为指定需要的属性字段
     * @param query
     * @param operationName
     * @param variables
     * @param requiredPatterns
     * @return
     */
    public ExecutionResult executeThrowsEByRequired(String query, String operationName, Map<String, Object> variables, Set<String> requiredPatterns) {
        ExecutionResult executionResult = graphQL().execute(new ExecutionInput(query, operationName, context, rootObject, transformVariables(schema, query, variables)));
        List<GraphQLError> errors = executionResult.getErrors();
        if (!CollectionUtils.isEmpty(errors)) {
            errors = errors.stream()
                    .filter(e -> {
                        String errorPath = errorPath(e);
                        return Objects.isNull(errorPath) || matchPath(requiredPatterns, errorPath);
                    }).collect(Collectors.toList());
            String message = errors.stream()
                    .map(this::extractException)
                    .filter(Optional::isPresent)
                    .map(Optional::get)
                    .map(Throwable::getMessage)
                    .collect(Collectors.joining(";"));
            if (StringUtils.isNotBlank(message)) {
                throw new ServiceException(message);
            }
        }
        return executionResult;
    }

    private GraphQL graphQL() {
        GraphQLSchemaProvider schemaProvider = SpringContextUtil.getBean(GraphQLSchemaProvider.class);
        if (instrumentation == null) {
            this.instrumentation = NoOpInstrumentation.INSTANCE;
        }

        if(contextBuilder == null) {
            this.contextBuilder = new DefaultGraphQLContextBuilder();
        }

        if(rootObjectBuilder == null) {
            this.rootObjectBuilder = new DefaultGraphQLRootObjectBuilder();
        }

        if(preparsedDocumentProvider == null) {
            this.preparsedDocumentProvider = NoOpPreparsedDocumentProvider.INSTANCE;
        }

        schema = schemaProvider.getReadOnlySchema(null);
        context = contextBuilder.build(Optional.empty(), Optional.empty());
        rootObject = rootObjectBuilder.build(Optional.empty(), Optional.empty());
        GraphQL graphQL = GraphQL.newGraphQL(schema)
                .queryExecutionStrategy(this.executionStrategyProvider.getQueryExecutionStrategy())
                .mutationExecutionStrategy(this.executionStrategyProvider.getMutationExecutionStrategy())
                .subscriptionExecutionStrategy(this.executionStrategyProvider.getSubscriptionExecutionStrategy())
                .instrumentation(this.instrumentation)
                .preparsedDocumentProvider(this.preparsedDocumentProvider)
                .build();
        return graphQL;
    }

    protected Map<String, Object> transformVariables(GraphQLSchema schema, String query, Map<String, Object> variables) {
        return variables;
    }

    private Optional<Throwable> extractException(GraphQLError error) {
        if (error instanceof ExceptionWhileDataFetching) {
            return Optional.of(((ExceptionWhileDataFetching) error).getException());
        } else if (error instanceof SerializationError) {
            return Optional.of(((SerializationError) error).getException());
        } else if (error instanceof GraphQLException) {
            return Optional.of((GraphQLException) error);
        }
        return Optional.of(new ServiceException(error.getMessage()));
    }

    private String errorPath(GraphQLError error) {
        if (CollectionUtils.isEmpty(error.getPath())) {
            return null;
        }
        if (error instanceof ExceptionWhileDataFetching) {
            // format("Exception while fetching data (%s) : %s", path, exception.getMessage())
            int startIndex = error.getMessage().indexOf("(");
            int endIndex = error.getMessage().indexOf(")");
            return error.getMessage().substring(startIndex + 1, endIndex);
        }
        StringBuilder builder = new StringBuilder("/");
        for (Object path: error.getPath()) {
            if (path instanceof String) {
                builder.append(path).append("/");
            } else if (path instanceof Integer) {
                builder.delete(builder.length() - 1, builder.length())
                        .append("[")
                        .append(path)
                        .append("]")
                        .append("/");
            }
        }
        return builder.deleteCharAt(builder.length()-1).toString();
    }

    private Boolean matchPath(Set<String> patterns, String errorPath) {
        for (String ignorePath: patterns) {
            if (pathMatcher.match(ignorePath, errorPath)) {
                return true;
            }
        }
        return false;
    }
}
```



#### 版本 5.0.2

`pom.xml` 依赖：<br/>

```xml
<dependency>
    <groupId>com.graphql-java</groupId>
    <artifactId>graphql-spring-boot-starter</artifactId>
    <version>5.0.2</version>
</dependency>
<dependency>
    <groupId>com.graphql-java</groupId>
    <artifactId>graphql-java-tools</artifactId>
    <version>5.2.4</version>
</dependency>
```



实现内部服务调用的 **Java** 代码：<br/>

```java
import graphql.*;
import graphql.servlet.GraphQLInvocationInputFactory;
import graphql.servlet.GraphQLQueryInvoker;
import graphql.servlet.GraphQLSingleInvocationInput;
import graphql.servlet.internal.GraphQLRequest;
import org.apache.commons.lang3.StringUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import org.springframework.util.CollectionUtils;
import org.springframework.util.PathMatcher;

import java.util.*;
import java.util.stream.Collectors;

/**
 *
 * @author wencheng
 * @date 2019/3/28
 */
@Component
public class IGraphQL {

    @Autowired
    private PathMatcher pathMatcher;

    @Autowired
    private GraphQLQueryInvoker queryInvoker;

    @Autowired
    private GraphQLInvocationInputFactory invocationInputFactory;

    /**
     * 执行查询,抛出运行中的异常
     *  注意: 抛出的异常 —— errorPath 为 null 或者 忽略指定属性字段外的字段异常
     * @param query
     * @param operationName
     * @param variables
     * @param ignorePatterns
     * @return
     */
    public ExecutionResult executeThrowsEAndIgnorePatterns(String query, String operationName, Map<String, Object> variables, Set<String> ignorePatterns) {
        ExecutionResult executionResult = execute(query, operationName, variables);
        List<GraphQLError> errors = executionResult.getErrors();
        if (!CollectionUtils.isEmpty(errors)) {
            errors = errors.stream()
                    .filter(e -> {
                        String errorPath = errorPath(e);
                        return Objects.nonNull(errorPath) && !matchPath(ignorePatterns, errorPath);
                    }).collect(Collectors.toList());
            String message = errors.stream()
                    .map(this::extractException)
                    .filter(Optional::isPresent)
                    .map(Optional::get)
                    .map(Throwable::getMessage)
                    .collect(Collectors.joining(";"));
            if (StringUtils.isNotBlank(message)) {
                throw new ServiceException(message);
            }
        }
        return executionResult;
    }

    /**
     * 执行查询,抛出运行中的异常
     *  注意: 抛出的异常 —— errorPath 为 null 或者 为指定需要的属性字段
     * @param query
     * @param operationName
     * @param variables
     * @param requiredPatterns
     * @return
     */
    public ExecutionResult executeThrowsEByRequired(String query, String operationName, Map<String, Object> variables, Set<String> requiredPatterns) {
        ExecutionResult executionResult = execute(query, operationName, variables);
        List<GraphQLError> errors = executionResult.getErrors();
        if (!CollectionUtils.isEmpty(errors)) {
            errors = errors.stream()
                    .filter(e -> {
                        String errorPath = errorPath(e);
                        return Objects.isNull(errorPath) || matchPath(requiredPatterns, errorPath);
                    }).collect(Collectors.toList());
            String message = errors.stream()
                    .map(this::extractException)
                    .filter(Optional::isPresent)
                    .map(Optional::get)
                    .map(Throwable::getMessage)
                    .collect(Collectors.joining(";"));
            if (StringUtils.isNotBlank(message)) {
                throw new ServiceException(message);
            }
        }
        return executionResult;
    }

    /**
     * 核心方法
     * @param query
     * @param operationName
     * @param variables
     * @return
     */
    public ExecutionResult execute(String query, String operationName, Map<String, Object> variables) {
        GraphQLSingleInvocationInput invocationInput = invocationInputFactory.create(new GraphQLRequest(query, variables, operationName));
        return queryInvoker.query(invocationInput);
    }

    private Optional<Throwable> extractException(GraphQLError error) {
        if (error instanceof ExceptionWhileDataFetching) {
            return Optional.of(((ExceptionWhileDataFetching) error).getException());
        } else if (error instanceof SerializationError) {
            return Optional.of(((SerializationError) error).getException());
        } else if (error instanceof GraphQLException) {
            return Optional.of((GraphQLException) error);
        }
        return Optional.of(new ServiceException(error.getMessage()));
    }

    private String errorPath(GraphQLError error) {
        if (CollectionUtils.isEmpty(error.getPath())) {
            return null;
        }
        if (error instanceof ExceptionWhileDataFetching) {
            // format("Exception while fetching data (%s) : %s", path, exception.getMessage())
            int startIndex = error.getMessage().indexOf("(");
            int endIndex = error.getMessage().indexOf(")");
            return error.getMessage().substring(startIndex + 1, endIndex);
        }
        StringBuilder builder = new StringBuilder("/");
        for (Object path: error.getPath()) {
            if (path instanceof String) {
                builder.append(path).append("/");
            } else if (path instanceof Integer) {
                builder.delete(builder.length() - 1, builder.length())
                        .append("[")
                        .append(path)
                        .append("]")
                        .append("/");
            }
        }
        return builder.deleteCharAt(builder.length()-1).toString();
    }

    private Boolean matchPath(Set<String> patterns, String errorPath) {
        for (String ignorePath: patterns) {
            if (pathMatcher.match(ignorePath, errorPath)) {
                return true;
            }
        }
        return false;
    }

}
```

