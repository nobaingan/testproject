### **🔍 Issue Analysis**  
The problem you're facing is that when `includeOnly=accountName` is passed:  
- The **AccountDetailsFilter** correctly filters `accountName`.  
- However, for **other DTOs** (like `BalanceDetails`, `Address`, etc.), filtering is **not applied properly**, so they still appear in the response.  

### **🔧 Root Cause**  
- The filter logic **only processes fields that are explicitly included** (`includeOnly`).  
- But **it does not exclude fields from DTOs that are not explicitly listed** in `includeOnly`.  
- This means, when you request `accountName`, other nested objects **retain all fields** because they are not explicitly filtered.

---

## **✅ Solution: Force Exclusion of All Fields Except Those in `includeOnly`**
### **Approach**
1. **For `includeOnly` mode**:  
   - **Only allow** explicitly requested fields.  
   - **Remove all fields from nested DTOs unless they are referenced explicitly.**  

2. **For `excludeOnly` mode**:  
   - **Remove explicitly listed fields** but **keep everything else.**  

---

### **🚀 Fixed `DynamicFilterUtil`**
```java
import com.fasterxml.jackson.databind.ser.FilterProvider;
import com.fasterxml.jackson.databind.ser.impl.SimpleBeanPropertyFilter;
import com.fasterxml.jackson.databind.ser.impl.SimpleFilterProvider;

import java.util.*;

public class DynamicFilterUtil {

    public static FilterProvider createFilters(Set<String> includeFields, Set<String> excludeFields, Class<?> rootClass) {
        Map<String, Set<String>> filters = new HashMap<>();
        Set<Class<?>> visited = new HashSet<>();

        extractNestedFields(includeFields, excludeFields, rootClass, "", filters, visited);

        SimpleFilterProvider filterProvider = new SimpleFilterProvider();
        filters.forEach((filterName, fields) -> {
            SimpleBeanPropertyFilter filter;
            if (!includeFields.isEmpty()) {
                // Ensure ALL fields not in includeFields are excluded
                filter = SimpleBeanPropertyFilter.filterOutAllExcept(fields);
            } else if (!excludeFields.isEmpty()) {
                filter = SimpleBeanPropertyFilter.serializeAllExcept(excludeFields);
            } else {
                filter = SimpleBeanPropertyFilter.serializeAll();
            }
            filterProvider.addFilter(filterName, filter);
        });

        return filterProvider;
    }

    private static void extractNestedFields(Set<String> includeFields, Set<String> excludeFields,
                                            Class<?> clazz, String prefix,
                                            Map<String, Set<String>> filters, Set<Class<?>> visited) {
        if (visited.contains(clazz)) return;
        visited.add(clazz);

        String filterName = clazz.getSimpleName() + "Filter";
        Set<String> relevantFields = new HashSet<>();

        Arrays.stream(clazz.getDeclaredFields()).forEach(field -> {
            String fullName = prefix.isEmpty() ? field.getName() : prefix + "." + field.getName();

            if (!includeFields.isEmpty()) {
                // If includeOnly is used, ONLY keep explicitly requested fields
                if (includeFields.contains(fullName) || includeFields.contains(field.getName())) {
                    relevantFields.add(field.getName());
                }
            } else if (!excludeFields.isEmpty()) {
                // If excludeOnly is used, REMOVE explicitly mentioned fields
                if (!excludeFields.contains(fullName) && !excludeFields.contains(field.getName())) {
                    relevantFields.add(field.getName());
                }
            } else {
                relevantFields.add(field.getName());
            }

            // Recursively process nested DTOs
            if (!field.getType().isPrimitive() && !field.getType().equals(String.class)) {
                extractNestedFields(includeFields, excludeFields, field.getType(), fullName, filters, visited);
            }
        });

        filters.put(filterName, relevantFields);
    }
}
```

---

### **✅ Fixed `DynamicFilterInterceptor`**
```java
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.ser.FilterProvider;
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.HandlerInterceptor;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.util.*;

@Component
public class DynamicFilterInterceptor implements HandlerInterceptor {

    private final ObjectMapper objectMapper;

    public DynamicFilterInterceptor(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {

        String includeOnly = request.getParameter("includeOnly");
        String excludeOnly = request.getParameter("excludeOnly");

        Set<String> includeFields = (includeOnly != null) ? new HashSet<>(Arrays.asList(includeOnly.split(","))) : new HashSet<>();
        Set<String> excludeFields = (excludeOnly != null) ? new HashSet<>(Arrays.asList(excludeOnly.split(","))) : new HashSet<>();

        request.setAttribute("includeFields", includeFields);
        request.setAttribute("excludeFields", excludeFields);
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) {

        Object responseObject = request.getAttribute("filteredResponse");
        if (responseObject != null) {
            Set<String> includeFields = (Set<String>) request.getAttribute("includeFields");
            Set<String> excludeFields = (Set<String>) request.getAttribute("excludeFields");

            FilterProvider filters = DynamicFilterUtil.createFilters(includeFields, excludeFields, responseObject.getClass());
            objectMapper.setFilterProvider(filters);

            response.setContentType("application/json");
            response.setCharacterEncoding("UTF-8");
        }
    }
}
```

---

### **✅ Expected Behavior After Fix**
#### **Case 1: `includeOnly=accountName`**
```http
GET /accounts?includeOnly=accountName
```
**Expected Response:**
```json
{
    "primaryAccount": {
        "accountName": "John Doe"
    },
    "secondaryAccount": {
        "accountName": "Jane Smith"
    }
}
```
✅ **Fixes the issue! Now, `tmDetails` and other properties are removed.**

#### **Case 2: `excludeOnly=balanceDetails`**
```http
GET /accounts?excludeOnly=balanceDetails
```
**Expected Response:**
```json
{
    "primaryAccount": {
        "accountId": "123456",
        "accountName": "John Doe",
        "address": {
            "street": "123 Main St",
            "city": "New York",
            "country": {
                "name": "USA",
                "code": "US"
            }
        }
    },
    "secondaryAccount": {
        "accountId": "789012",
        "accountName": "Jane Smith",
        "address": {
            "street": "456 Elm St",
            "city": "Los Angeles",
            "country": {
                "name": "USA",
                "code": "US"
            }
        }
    }
}
```
✅ **Now correctly removes `balanceDetails` without affecting other fields.**

---

### **🛠 Why This Fix Works**
1. **Explicitly removes all fields in nested DTOs unless explicitly included.**  
2. **Handles both `includeOnly` and `excludeOnly` cases properly.**  
3. **Ensures filtering works at all levels of the object tree.**  
4. **Keeps filtering dynamic (no hardcoded filter names).**  

🚀 Now, the filtering **should work exactly as expected!** Let me know if you need further refinements.
