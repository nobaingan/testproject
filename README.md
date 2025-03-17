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

private static void processFields(Set<String> includeFields, Set<String> excludeFields, Class<?> clazz,
                                  String prefix, Map<String, Set<String>> filters, Set<Class<?>> visited) {
    if (visited.contains(clazz)) {
        return; // Prevent infinite recursion
    }
    visited.add(clazz);

    String filterName = clazz.getSimpleName() + "Filter";
    Set<String> relevantFields = new HashSet<>();

    logger.debug("Processing class: {}, Prefix: {}", clazz.getSimpleName(), prefix);

    for (Field field : clazz.getDeclaredFields()) {
        String fullName = prefix.isEmpty() ? field.getName() : prefix + "." + field.getName();

        // Include fields explicitly listed
        if (includeFields.isEmpty() || includeFields.contains(fullName) || includeFields.contains(field.getName())) {
            logger.debug("Including field: {}", fullName);
            relevantFields.add(field.getName());
        }

        // Exclude explicitly listed fields
        if (excludeFields.contains(fullName) || excludeFields.contains(field.getName())) {
            logger.debug("Excluding field: {}", fullName);
            relevantFields.remove(field.getName());
        }

        // Handle nested objects and collections
        if (!field.getType().isPrimitive() && !field.getType().equals(String.class)) {
            if (Collection.class.isAssignableFrom(field.getType())) {
                // Handle collections (e.g., List<Transaction>)
                if (includeFields.contains(fullName) || includeFields.contains(field.getName())) {
                    logger.debug("Including collection field: {}", fullName);
                    relevantFields.add(field.getName());
                }
                processCollectionFields(includeFields, excludeFields, field, fullName, filters, visited);
            } else {
                // Handle nested objects
                processFields(includeFields, excludeFields, field.getType(), fullName, filters, visited);
                if (shouldRetainParent(includeFields, excludeFields, fullName)) {
                    relevantFields.add(field.getName());
                }
            }
        }
    }

    filters.put(filterName, relevantFields);
    logger.info("Filter for class {}: {}", clazz.getSimpleName(), relevantFields);
}
