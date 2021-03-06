# 使用spring validation完成数据后端校验

参考链接([ 徐靖峰|个人博客](https://www.cnkirito.moe/))：<https://www.cnkirito.moe/spring-validation/>

>背景：项目准备将 `restful` 请求方式改造成 `graphql` 请求方式，项目是面向 B 端的，PC 端提交表单参数内容非常多，参数校验也比较多。

## `restful` 与 `graphql` 请求参数校验对比

使用 `spring mvc` 的 `restful` 请求，`spring mvc`  框架 中 `WebDataBinder`  可以将 `request` 中的参数自动绑定到 `POJO` 对象中，可以使用 `spring validation` 完成后端校验。

但是 `graphql` 请求方式，使用的是 `graphql` 官方语法定义的 `graphql-tools` 来进行参数解析和绑定。示例如下：<br/>![graphql请求参数](images/graphql请求参数.png)

![graphql解析入参](images/graphql解析入参.png)

`graphql`  将 `variables` 参数解析为 `linkedHashMap`，`graphql` 定义了`Instrumentation` 可以在 `graphql` 执行的各个阶段进行拦截处理，类似 `spring` 的 `Interceptor`，因为定义参数校验的方式可以如下：<br/>

```java
import graphql.GraphQLError;
import graphql.execution.ExecutionPath;
import graphql.execution.instrumentation.fieldvalidation.*;
import graphql.execution.instrumentation.fieldvalidation.FieldValidation;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import graphql.execution.instrumentation.Instrumentation;
import java.util.Optional;
import java.util.function.BiFunction;

/**
 *
 * @author wencheng
 * @date 2019/3/27
 */
@Configuration
public class InstrumentationConfig {

    @Bean
    public Instrumentation instrumentation() {
        // 参数校验的请求路径，这里对应于 addSeqGroup 方法
        ExecutionPath fieldPath = ExecutionPath.parse("/addSeqGroup");
        FieldValidation fieldValidation = new SimpleFieldValidation()
                .addRule(fieldPath, new BiFunction<FieldAndArguments, FieldValidationEnvironment, Optional<GraphQLError>>() {
                    @Override
                    public Optional<GraphQLError> apply(FieldAndArguments fieldAndArguments, FieldValidationEnvironment environment) {
                        // 获取到对应请求路径的方法的参数后，下面就需要自己定义参数校验规则了
                        Object input = fieldAndArguments.getArgumentValue("input");
                        return Optional.empty();
                    }
                });
        return new FieldValidationInstrumentation(fieldValidation);
    }

}
```

使用上面 `graphql` 定义的参数校验方式，很繁琐，也不容易复用。因此还是希望通过之前 `spring mvc` 的 `validation` 方式来校验。而且 `graphql` 也不关注参数校验和权限校验，只关注对应的资源获取。



## 通过 `spring aop` 方式完成 spring validation数据后端校验

### 方式一：2019年03月23日

`spring mvc` 实现基于 `validation` 的后端 `POJO` 对象参数校验，其内部还是调用 `javax.validation.Validator#validate` 方法进行校验，于是通过自己 `debug` 方式，将 `spring mvc` 中对应的方法拷贝出来，实现了对应的功能。代码如下：<br/>

```java
import com.enums.ErrorCode;
import com.exception.ServiceException;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.core.Conventions;
import org.springframework.core.MethodParameter;
import org.springframework.core.annotation.AnnotationUtils;
import org.springframework.core.annotation.Order;
import org.springframework.format.support.FormattingConversionService;
import org.springframework.stereotype.Component;
import org.springframework.validation.Errors;
import org.springframework.validation.Validator;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.WebDataBinder;
import org.springframework.web.bind.support.ConfigurableWebBindingInitializer;
import org.springframework.web.servlet.mvc.method.annotation.ExtendedServletRequestDataBinder;

import java.lang.annotation.Annotation;
import java.lang.reflect.Method;

/**
 *
 * @author wencheng
 * @date 2019/3/22
 */
@Component
@Aspect
@Order(2)//注意order优先级要比TransactionInterceptor高
@Slf4j
public class ValidatorAspect {

    @Autowired
    private FormattingConversionService mvcConversionService;

    @Autowired
    private Validator mvcValidator;

    @Pointcut("(execution(public * com.graphql.mutation.*.*(..))) ||" +
            "(execution(public * com.graphql.query.*.*(..)))")
    public void validatePointCuts() {
    }

    @Around(value = "validatePointCuts()")
    public Object validate(ProceedingJoinPoint joinPoint) throws Throwable {
        Method method = ((MethodSignature) joinPoint.getSignature()).getMethod();
        IHandlerMethod iHandlerMethod = new IHandlerMethod(joinPoint.getThis(), method);
        MethodParameter[] parameters = iHandlerMethod.getMethodParameters();
        for (MethodParameter parameter: parameters) {
            parameter = parameter.nestedIfOptional();
            ConfigurableWebBindingInitializer initializer = new ConfigurableWebBindingInitializer();
            initializer.setConversionService(mvcConversionService);
            initializer.setValidator(mvcValidator);
            initializer.setMessageCodesResolver(null);

            String name = Conventions.getVariableNameForParameter(parameter);
            WebDataBinder binder = new ExtendedServletRequestDataBinder(joinPoint.getArgs()[parameter.getParameterIndex()], name);
            if (initializer != null) {
                initializer.initBinder(binder, null);
            }

            if (joinPoint.getArgs() != null) {
                validateIfApplicable(binder, parameter);
                if (binder.getBindingResult().hasErrors() && isBindExceptionRequired(binder, parameter)) {
                    throw new ServiceException(binder.getBindingResult().getFieldError().getDefaultMessage(), ErrorCode.INVALID_ARGUMENT.getCode());
                }
            }
        }

        return joinPoint.proceed();
    }

    protected void validateIfApplicable(WebDataBinder binder, MethodParameter parameter) {
        Annotation[] annotations = parameter.getParameterAnnotations();
        for (Annotation ann : annotations) {
            Validated validatedAnn = AnnotationUtils.getAnnotation(ann, Validated.class);
            if (validatedAnn != null || ann.annotationType().getSimpleName().startsWith("Valid")) {
                Object hints = (validatedAnn != null ? validatedAnn.value() : AnnotationUtils.getValue(ann));
                Object[] validationHints = (hints instanceof Object[] ? (Object[]) hints : new Object[] {hints});
                binder.validate(validationHints);
                break;
            }
        }
    }

    protected boolean isBindExceptionRequired(WebDataBinder binder, MethodParameter parameter) {
        int i = parameter.getParameterIndex();
        Class<?>[] paramTypes = parameter.getMethod().getParameterTypes();
        boolean hasBindingResult = (paramTypes.length > (i + 1) && Errors.class.isAssignableFrom(paramTypes[i + 1]));
        return !hasBindingResult;
    }

}
```

这样做的目的：

1. 直接参考 `spring mvc` 框架实现方式，拷贝其代码，系统运行中，兼容度比较强，bug 可控。
2. 项目开发进度比较紧，通过查询资料，并不能保证一定能在较短时间内，达到项目可用且稳定的目的。

问题：

1. 直接拷贝 `spring mvc` 框架的代码，其实现比较繁杂，直接拷贝，其中包含很多无用代码。
2. 在 `spring aop` 中 抛出了 `MethodArgumentNotValidException`，其异常直接继承自 `Exception`，不是 `RuntimeException`，所以抛出了**非受检时异常**，`spring aop` 中，对于抛出的**非受检时异常**，如果没有自己处理，会将其包装成 `UndeclaredThrowableException`，因此 `@ControllerAdvice` 定义的异常处理就无法执行。在 `UndeclaredThrowableException` 中获取原始异常，还需要通过以下方式。![UndeclaredThrowableException获取原始异常](images/UndeclaredThrowableException获取原始异常.png)



### 方式二：2019年04月08日

使用 `javax.validation.Validator` 自己校验。代码如下：<br/>

```java
import com.enums.ErrorCode;
import com.exception.ServiceException;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.lang3.StringUtils;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.core.MethodParameter;
import org.springframework.core.annotation.AnnotationUtils;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;
import org.springframework.validation.annotation.Validated;

import javax.validation.ConstraintViolation;
import javax.validation.Validation;
import javax.validation.ValidatorFactory;
import java.lang.annotation.Annotation;
import java.lang.reflect.Method;
import java.util.Set;

/**
 *
 * @author wencheng
 * @date 2019/3/22
 */
@Component
@Aspect
@Order(2)
@Slf4j
public class ValidatorAspect {

    @Pointcut("(execution(public * com.graphql.mutation.*.*(..))) ||" +
            "(execution(public * com.graphql.query.*.*(..)))")
    public void validatePointCuts() {
    }

    @Around(value = "validatePointCuts()")
    public Object validate(ProceedingJoinPoint joinPoint) throws Throwable {
        Method method = ((MethodSignature) joinPoint.getSignature()).getMethod();
        IHandlerMethod iHandlerMethod = new IHandlerMethod(joinPoint.getThis(), method);
        MethodParameter[] parameters = iHandlerMethod.getMethodParameters();

        ValidatorFactory vf = Validation.buildDefaultValidatorFactory();
        javax.validation.Validator validator = vf.getValidator();
        for (MethodParameter parameter: parameters) {
            Object args = joinPoint.getArgs()[parameter.getParameterIndex()];
            Annotation[] annotations = parameter.getParameterAnnotations();
            StringBuilder errMsgBuider = new StringBuilder();
            for (Annotation ann : annotations) {
                Validated validatedAnn = AnnotationUtils.getAnnotation(ann, Validated.class);
                if (validatedAnn != null || ann.annotationType().getSimpleName().startsWith("Valid")) {
                    Class<?>[] validationHints = (validatedAnn != null ? validatedAnn.value() : new Class[0]);
                    Set<ConstraintViolation<Object>> constraintViolationSet = validator.validate(args, validationHints);
                    for (ConstraintViolation<?> constraintViolation : constraintViolationSet) {
                        errMsgBuider.append(constraintViolation.getMessage()).append(";");
                    }
                }
            }
            if (StringUtils.isNotBlank(errMsgBuider.toString())) {
                throw new ServiceException(errMsgBuider.toString(), ErrorCode.INVALID_ARGUMENT.getCode());
            }
        }

        return joinPoint.proceed();
    }

}
```

上面代码，避免直接拷贝 `spring mvc` 实现方式，简化了实现方式。