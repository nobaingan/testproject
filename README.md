package com.example.demo.util;

import com.fasterxml.jackson.annotation.JsonFilter;
import com.fasterxml.jackson.databind.ser.FilterProvider;
import com.fasterxml.jackson.databind.ser.impl.SimpleBeanPropertyFilter;
import com.fasterxml.jackson.databind.ser.impl.SimpleFilterProvider;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.lang.reflect.Field;
import java.util.*;

public class DynamicFilterUtil {

    private static final Logger logger = LoggerFactory.getLogger(DynamicFilterUtil.class);

    /**
     * Creates a FilterProvider based on `includeOnly` and `excludeOnly` parameters.
     *
     * @param includeFields Fields to include explicitly (`includeOnly`).
     * @param excludeFields Fields to exclude explicitly (`excludeOnly`).
     * @param rootClass     The root class to process.
     * @return A configured FilterProvider for dynamic filtering.
     */
    public static FilterProvider createFilters(Set<String> includeFields, Set<String> excludeFields, Class<?> rootClass) {
        logger.info("Creating filters for root class: {}", rootClass.getSimpleName());
        logger.info("Include fields: {}", includeFields);
        logger.info("Exclude fields: {}", excludeFields);

        Map<String, Set<String>> filters = new HashMap<>();
        Set<Class<?>> visited = new HashSet<>();

        // Process fields with constraints
        processFields(includeFields, excludeFields, rootClass, "", filters, visited);

        logger.info("Generated filters: {}", filters);
        return buildFilterProvider(filters);
    }

    /**
     * Processes fields dynamically, restricting to classes with `@JsonFilter` annotation or specific packages (e.g., `.dto`).
     */
    private static void processFields(Set<String> includeFields, Set<String> excludeFields, Class<?> clazz,
                                      String prefix, Map<String, Set<String>> filters, Set<Class<?>> visited) {
        if (visited.contains(clazz)) {
            return; // Prevent infinite recursion
        }
        visited.add(clazz);

        // Skip classes without @JsonFilter or outside the desired package
        if (!isRelevantClass(clazz)) {
            logger.debug("Skipping irrelevant class: {}", clazz.getName());
            return;
        }

        String filterName = clazz.getSimpleName() + "Filter";
        Set<String> relevantFields = new HashSet<>();
        logger.debug("Processing class: {}, Prefix: {}", clazz.getSimpleName(), prefix);

        for (Field field : clazz.getDeclaredFields()) {
            String fullName = prefix.isEmpty() ? field.getName() : prefix + "." + field.getName();

            // Handle includeOnly: Add fields explicitly listed
            if (includeFields.isEmpty() || includeFields.contains(fullName) || includeFields.contains(prefix)) {
                relevantFields.add(field.getName());
            }

            // Handle excludeOnly: Remove fields explicitly listed
            if (excludeFields.contains(fullName) || excludeFields.contains(field.getName())) {
                relevantFields.remove(field.getName());
            }

            // Recursively process nested fields
            if (!field.getType().isPrimitive() && !field.getType().equals(String.class)) {
                processFields(includeFields, excludeFields, field.getType(), fullName, filters, visited);

                // Retain parent objects if they are partially included or excluded
                if (shouldRetainParent(includeFields, excludeFields, fullName)) {
                    relevantFields.add(field.getName());
                }
            }
        }

        filters.put(filterName, relevantFields);
        logger.info("Filter for class {}: {}", clazz.getSimpleName(), relevantFields);
    }

    /**
     * Determines if a class is relevant based on the presence of `@JsonFilter` or its package.
     *
     * @param clazz The class to evaluate.
     * @return True if the class is relevant, false otherwise.
     */
    private static boolean isRelevantClass(Class<?> clazz) {
        // Check for @JsonFilter annotation
        if (clazz.isAnnotationPresent(JsonFilter.class)) {
            return true;
        }

        // Restrict to classes in the `.dto` package
        String packageName = clazz.getPackage().getName();
        return packageName.contains(".dto");
    }

    /**
     * Determines if a parent object should be retained based on `includeOnly` or `excludeOnly`.
     */
    private static boolean shouldRetainParent(Set<String> includeFields, Set<String> excludeFields, String parentPath) {
        return includeFields.stream().anyMatch(field -> field.startsWith(parentPath + ".")) ||
                excludeFields.stream().anyMatch(field -> field.startsWith(parentPath + "."));
    }

    /**
     * Builds the FilterProvider from the filters map.
     */
    private static FilterProvider buildFilterProvider(Map<String, Set<String>> filters) {
        SimpleFilterProvider filterProvider = new SimpleFilterProvider();
        filters.forEach((className, fields) -> {
            SimpleBeanPropertyFilter filter = fields.isEmpty()
                    ? SimpleBeanPropertyFilter.filterOutAll()
                    : SimpleBeanPropertyFilter.filterOutAllExcept(fields);
            logger.debug("Adding filter for class: {}, Fields: {}", className, fields);
            filterProvider.addFilter(className, filter);
        });

        filterProvider.setFailOnUnknownId(false);
        return filterProvider;
    }
}
