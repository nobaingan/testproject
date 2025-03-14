### **🚀 Fully Dynamic Solution: Exclude All Fields When Not Explicitly Included**
Since the filter names and fields **must be determined dynamically**, we need to extract all fields **at runtime** from the DTOs.

---

## **🔹 Solution**
1. **Find all fields dynamically** for a given DTO using **reflection**.
2. **Ensure that if `includeOnly` applies to one filter, all other filters exclude all their fields.**
3. **Automatically determine filter names** using `@JsonFilter` annotations on DTOs.

---

### **🔄 Updated `createFilters` Method**
```java
public static FilterProvider createFilters(Set<String> includeFields, Set<String> excludeFields, Class<?> rootClass) {
    Map<String, Set<String>> filters = new HashMap<>();
    Set<Class<?>> visited = new HashSet<>();

    // Extract nested fields dynamically
    extractNestedFields(includeFields, excludeFields, rootClass, "", filters, visited);

    SimpleFilterProvider filterProvider = new SimpleFilterProvider();

    // Get filters that have at least one explicitly included field
    Set<String> explicitlyFiltered = filters.entrySet().stream()
            .filter(entry -> !entry.getValue().isEmpty())
            .map(Map.Entry::getKey)
            .collect(Collectors.toSet());

    // Apply filters dynamically
    filters.forEach((filterName, fields) -> {
        SimpleBeanPropertyFilter filter;

        if (!fields.isEmpty()) {
            // ✅ Include only explicitly requested fields
            filter = SimpleBeanPropertyFilter.filterOutAllExcept(fields);
        } else if (!explicitlyFiltered.isEmpty()) {
            // ❌ If any filters have explicit fields, exclude everything in others
            Set<String> allFields = getAllFieldsDynamically(filterName, rootClass);
            filter = SimpleBeanPropertyFilter.serializeAllExcept(allFields);
        } else {
            // Default: No filtering (fallback case)
            filter = SimpleBeanPropertyFilter.serializeAll();
        }

        filterProvider.addFilter(filterName, filter);
    });

    return filterProvider;
}
```

---

### **🔍 Dynamic Method to Get All Fields in a DTO**
```java
private static Set<String> getAllFieldsDynamically(String filterName, Class<?> rootClass) {
    Set<String> allFields = new HashSet<>();

    // Find the DTO class with the correct @JsonFilter annotation
    for (Field field : rootClass.getDeclaredFields()) {
        if (field.isAnnotationPresent(JsonFilter.class)) {
            JsonFilter jsonFilter = field.getAnnotation(JsonFilter.class);
            if (jsonFilter.value().equals(filterName)) {
                // Add all declared fields dynamically
                for (Field subField : field.getType().getDeclaredFields()) {
                    allFields.add(subField.getName());
                }
                break;
            }
        }
    }

    return allFields;
}
```

---

## **✅ How This Works**
- **Dynamically detects filters** based on `@JsonFilter` annotations in DTOs.
- **Extracts all fields of each DTO at runtime** via reflection.
- **Ensures that only the requested fields are included** while everything else is excluded.

---

### **🚀 Expected Behavior**
#### **Request:** `includeOnly=accountName`
**Before Fix (Incorrect Output)**  
```json
{
    "account": { "accountName": "test" },
    "tmDetails": { "proprietary": "XYZ", "masterAccountIdent": null },
    "product": { "identifier": "123", "type": "Savings" }
}
```
**After Fix (Correct Output)**  
```json
{
    "account": { "accountName": "test" }
}
```
✔️ `tmDetails` and `product` are **completely removed!**

---

## **🚀 Final Fix Summary**
✅ **Fully dynamic** – No hardcoded DTO field lists  
✅ **Uses reflection** to get fields at runtime  
✅ **Ensures only relevant filters allow data**  
✅ **Removes entire objects if not explicitly requested**  

Try it and let me know if this finally solves the issue! 🚀
public static FilterProvider createFilters(Set<String> includeFields, Set<String> excludeFields, Class<?> rootClass) {
    Map<String, Set<String>> filters = new HashMap<>();
    Set<Class<?>> visited = new HashSet<>();

    // Extract nested fields dynamically
    extractNestedFields(includeFields, excludeFields, rootClass, "", filters, visited);

    SimpleFilterProvider filterProvider = new SimpleFilterProvider();

    // If includeOnly is used, determine which filters should be empty
    boolean isIncludeOnly = !includeFields.isEmpty();

    filters.forEach((filterName, fields) -> {
        SimpleBeanPropertyFilter filter;

        if (isIncludeOnly) {
            if (!fields.isEmpty()) {
                // ✅ Include only explicitly requested fields
                filter = SimpleBeanPropertyFilter.filterOutAllExcept(fields);
            } else {
                // ❌ If nothing was explicitly included, exclude everything
                filter = SimpleBeanPropertyFilter.serializeAllExcept("*");
            }
        } else if (!excludeFields.isEmpty()) {
            // ✅ Exclude only specified fields, keep the rest
            filter = SimpleBeanPropertyFilter.serializeAllExcept(excludeFields);
        } else {
            // Default: No filtering (fallback case)
            filter = SimpleBeanPropertyFilter.serializeAll();
        }

        filterProvider.addFilter(filterName, filter);
    });

    return filterProvider;
}

