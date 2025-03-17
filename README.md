package com.example.demo.aspect;

import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.annotation.AfterReturning;
import org.aspectj.lang.annotation.AfterThrowing;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.ProceedingJoinPoint;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

@Aspect
@Component
public class LoggingAspect {

    private static final Logger logger = LoggerFactory.getLogger(LoggingAspect.class);

    // Log all method calls in the specified package
    @Around("execution(* com.example.demo..*(..))")
    public Object logMethodCall(ProceedingJoinPoint joinPoint) throws Throwable {
        String methodName = joinPoint.getSignature().getName();
        String className = joinPoint.getTarget().getClass().getSimpleName();

        logger.info("Entering method: {}.{}", className, methodName);
        logger.info("Arguments: {}", joinPoint.getArgs());

        long startTime = System.currentTimeMillis();

        try {
            Object result = joinPoint.proceed(); // Proceed with the intercepted method
            long elapsedTime = System.currentTimeMillis() - startTime;

            logger.info("Exiting method: {}.{}; Execution time: {} ms", className, methodName, elapsedTime);
            logger.info("Return Value: {}", result);

            return result;
        } catch (Exception ex) {
            logger.error("Exception in method: {}.{}; Message: {}", className, methodName, ex.getMessage(), ex);
            throw ex;
        }
    }

    // Log after successful method execution
    @AfterReturning(pointcut = "execution(* com.example.demo..*(..))", returning = "result")
    public void logAfterReturning(JoinPoint joinPoint, Object result) {
        String methodName = joinPoint.getSignature().getName();
        logger.info("Method {} executed successfully. Returned: {}", methodName, result);
    }

    // Log after exceptions are thrown
    @AfterThrowing(pointcut = "execution(* com.example.demo..*(..))", throwing = "exception")
    public void logAfterThrowing(JoinPoint joinPoint, Exception exception) {
        String methodName = joinPoint.getSignature().getName();
        logger.error("Exception thrown in method {}. Exception: {}", methodName, exception.getMessage());
    }
}

	implementation 'org.springframework.boot:spring-boot-starter-aop'
logging.level.root=INFO
logging.level.com.example.demo=DEBUG
logging.pattern.console=%d{yyyy-MM-dd HH:mm:ss} %-5level %logger{36} - %msg%n

    /**
     * Processes collections of nested objects dynamically.
     */
    private static void processCollectionFields(Set<String> includeFields, Set<String> excludeFields, Field field,
                                                String fullName, Map<String, Set<String>> filters, Set<Class<?>> visited) {
        // Get the generic type of the collection (e.g., Transaction)
        Class<?> collectionElementType = getCollectionElementType(field);
        if (collectionElementType != null) {
            processFields(includeFields, excludeFields, collectionElementType, fullName, filters, visited);
        }
    }

    /**
     * Retrieves the element type of a collection field (e.g., List<Transaction> -> Transaction).
     */
    private static Class<?> getCollectionElementType(Field field) {
        try {
            return (Class<?>) ((java.lang.reflect.ParameterizedType) field.getGenericType()).getActualTypeArguments()[0];
        } catch (Exception e) {
            logger.warn("Unable to determine collection element type for field: {}", field.getName(), e);
            return null;
        }
    }
