package com.example.demo.util;

import com.fasterxml.jackson.databind.ser.FilterProvider;
import com.fasterxml.jackson.databind.ser.impl.SimpleBeanPropertyFilter;
import com.fasterxml.jackson.databind.ser.impl.SimpleFilterProvider;

import java.util.*;

public class DynamicFilterUtil {

    /**
     * Creates a FilterProvider based on `includeOnly` and `excludeOnly` parameters.
     *
     * @param includeFields Fields to include explicitly (`includeOnly`).
     * @param excludeFields Fields to exclude explicitly (`excludeOnly`).
     * @param rootClass     The root class to process.
     * @return A configured FilterProvider for dynamic filtering.
     */
    public static FilterProvider createFilters(Set<String> includeFields, Set<String> excludeFields, Class<?> rootClass) {
        Map<String, Set<String>> filters = new HashMap<>();
        Set<Class<?>> visited = new HashSet<>();

        // Process fields for both `includeOnly` and `excludeOnly`
        processFields(includeFields, excludeFields, rootClass, "", filters, visited);

        // Create the filter provider
        SimpleFilterProvider filterProvider = new SimpleFilterProvider();
        filters.forEach((className, fields) -> {
            SimpleBeanPropertyFilter filter = fields.isEmpty()
                    ? SimpleBeanPropertyFilter.filterOutAll() // Exclude objects with no relevant fields
                    : SimpleBeanPropertyFilter.filterOutAllExcept(fields); // Include only specified fields
            filterProvider.addFilter(className, filter);
        });

        filterProvider.setFailOnUnknownId(false); // Allow classes without explicit filters
        return filterProvider;
    }

    /**
     * Processes fields dynamically for both `includeOnly` and `excludeOnly`.
     */
    private static void processFields(Set<String> includeFields, Set<String> excludeFields, Class<?> clazz,
                                      String prefix, Map<String, Set<String>> filters, Set<Class<?>> visited) {
        if (visited.contains(clazz)) {
            return; // Prevent infinite recursion
        }
        visited.add(clazz);

        String filterName = clazz.getSimpleName() + "Filter";
        Set<String> relevantFields = new HashSet<>();

        Arrays.stream(clazz.getDeclaredFields()).forEach(field -> {
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

                // Retain parent objects if partially included or excluded
                if (shouldRetainParent(includeFields, excludeFields, fullName)) {
                    relevantFields.add(field.getName());
                }
            }
        });

        // Add the filter for this class
        filters.put(filterName, relevantFields);
    }

    /**
     * Determines if a parent object should be retained based on `includeOnly` or `excludeOnly`.
     *
     * @param includeFields The set of fields to include.
     * @param excludeFields The set of fields to exclude.
     * @param parentPath    The parent path of the nested object (e.g., "account").
     * @return True if the parent object should be retained, false otherwise.
     */
    private static boolean shouldRetainParent(Set<String> includeFields, Set<String> excludeFields, String parentPath) {
        return includeFields.stream().anyMatch(field -> field.startsWith(parentPath + ".")) ||
                excludeFields.stream().anyMatch(field -> field.startsWith(parentPath + "."));
    }
}
