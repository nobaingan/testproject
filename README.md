## **Confluence Page: Implementing Dynamic Filtering for Account Service API**

### **Background**
The Account service API provides detailed information about an account, including key attributes such as the **account balance**, **deposits**, **interest**, **withdrawals**, and other account-related data. 

To optimize performance, reduce payload sizes, and improve client-server communication efficiency, we want to enable filtering functionality in our API. This will allow consumers to **include or exclude** specific fields dynamically. For example, consumers may only want the **account details** without the associated **deposits**, **interest**, and **withdrawals**.

---

### **Why We Should Implement This Feature**
Implementing dynamic field selection allows us to achieve several key benefits:

- **Reduced Payload Size**: Only relevant data is returned, reducing the amount of data transferred.
- **Improved Performance**: Smaller responses result in faster response times for API calls.
- **Minimized Bandwidth Usage**: Sending only the necessary fields minimizes network usage.
- **Simplified Processing**: Both client-side and server-side logic is simplified as consumers only receive the necessary data.

---

### **Scope of the Spike**
The goal is to investigate potential approaches for implementing **dynamic filtering** in the **Account service API** to return only the selected fields based on **query parameters** or **header parameters**.

---

### **Use Case Examples**
1. **Excluding Specific Fields**: 
   - A consumer wants to exclude the **interest**, **deposits**, and **withdrawals** from the response.
   - **API Request**:  
     ```
     GET /accounts/{accountNumber}?excludeOnly=interest,deposits,withdrawals
     ```

2. **Including Specific Fields**:
   - A consumer only wants specific fields such as **accountName**, **product**, and **maturityDate**.
   - **API Request**:
     ```
     GET /accounts/{accountNumber}?includeOnly=accountName,product,maturityDate
     ```

---

### **Approaches to Implement Dynamic Filtering**
Several methods can be used to implement dynamic filtering in Spring Boot. Below are some potential options:

---

### **Option 1: Jackson `@JsonFilter` (Dynamic Filtering)**
One common approach is to use **Jackson's `@JsonFilter`** annotation, which allows for dynamic filtering at runtime.

#### **How It Works**:
- The `@JsonFilter` annotation is used to create a **dynamic filter** on specific fields or objects.
- The controller will read query parameters (such as `includeOnly` or `excludeOnly`), apply the filter, and return the data.

#### **Implementation**:

##### **Step 1: Annotate DTOs with `@JsonFilter`**
Ensure both **parent and nested DTO classes** are annotated with `@JsonFilter`:

```java
import com.fasterxml.jackson.annotation.JsonFilter;
import lombok.Getter;
import lombok.Setter;

@Getter
@Setter
@JsonFilter("dynamicFilter")  // Apply dynamic filtering
public class AccountDetails {
    private String accountId;
    private String accountName;
    private String accountNumber;
    private String accountSortCode;
    private String product;
    private List<Deposit> deposits;
    private List<Withdrawal> withdrawals;
    private String interestRate;
    private String balance;
    private String openedDate;
}
```

##### **Step 2: Controller Logic to Apply Filters**
The controller should handle the dynamic filtering based on request parameters such as `includeOnly` or `excludeOnly`.

```java
import com.fasterxml.jackson.databind.ser.impl.SimpleBeanPropertyFilter;
import com.fasterxml.jackson.databind.ser.impl.SimpleFilterProvider;
import com.fasterxml.jackson.databind.ser.FilterProvider;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.http.converter.json.MappingJacksonValue;
import org.springframework.web.bind.annotation.*;

import java.util.*;

@RestController
@RequestMapping("/accounts")
public class AccountController {

    @GetMapping("/{accountNumber}")
    public MappingJacksonValue getAccount(
            @PathVariable String accountNumber,
            @RequestParam(value = "includeOnly", required = false) String includeOnly,
            @RequestParam(value = "excludeOnly", required = false) String excludeOnly) {

        // Sample data for response
        AccountDetails account = new AccountDetails();
        account.setAccountId("123456");
        account.setAccountName("John Doe");
        account.setAccountNumber(accountNumber);
        account.setProduct("Savings");
        account.setBalance("10000.00");

        // Apply dynamic filtering
        MappingJacksonValue mapping = applyFiltering(account, includeOnly, excludeOnly);
        return mapping;
    }

    private MappingJacksonValue applyFiltering(Object responseData, String includeOnly, String excludeOnly) {
        SimpleFilterProvider filterProvider = new SimpleFilterProvider();

        // Convert comma-separated values to a Set
        Set<String> includeFields = (includeOnly != null && !includeOnly.isEmpty()) ?
                new HashSet<>(Arrays.asList(includeOnly.split(","))) : null;

        Set<String> excludeFields = (excludeOnly != null && !excludeOnly.isEmpty()) ?
                new HashSet<>(Arrays.asList(excludeOnly.split(","))) : null;

        // Apply Include-Only or Exclude-Only Filtering
        if (includeFields != null) {
            filterProvider.addFilter("dynamicFilter",
                    SimpleBeanPropertyFilter.filterOutAllExcept(includeFields));
        } else if (excludeFields != null) {
            filterProvider.addFilter("dynamicFilter",
                    SimpleBeanPropertyFilter.serializeAllExcept(excludeFields));
        } else {
            filterProvider.addFilter("dynamicFilter", SimpleBeanPropertyFilter.serializeAll());
        }

        MappingJacksonValue mapping = new MappingJacksonValue(responseData);
        mapping.setFilters(filterProvider);
        return mapping;
    }
}
```

##### **Step 3: Test the API**
1. **GET** `/accounts/{accountNumber}?excludeOnly=interest,deposits,withdrawals`
   - **Response**: Excludes interest, deposits, and withdrawals fields, and returns the rest of the account details.

2. **GET** `/accounts/{accountNumber}?includeOnly=accountName,product,maturityDate`
   - **Response**: Only includes the account name, product, and maturity date.

---

### **Option 2: Using Libraries Like **Squiggly**
Squiggly is a third-party library that provides **dynamic filtering** of JSON objects based on query parameters. It is an easier-to-implement solution compared to custom filtering.

#### **How It Works**:
- Squiggly parses request parameters and filters out fields accordingly.
- It allows you to specify query parameters in the format `includeOnly=field1,field2` or `excludeOnly=field1,field2` directly in the URL.

#### **Implementation**:
1. Add Squiggly dependency to your `pom.xml`:

```xml
<dependency>
    <groupId>org.squiggly</groupId>
    <artifactId>squiggly-annotations</artifactId>
    <version>1.3.2</version>
</dependency>
```

2. Configure **Squiggly** in your Spring Boot app:

```java
@Configuration
public class SquigglyConfig extends ObjectMapperConfigurer {
    @Override
    public void configure(ObjectMapper objectMapper) {
        objectMapper.registerModule(new SquigglyModule());
    }
}
```

3. Use the `@JsonSquiggly` annotation in your DTOs to define which fields can be filtered dynamically:

```java
@JsonSquiggly
public class AccountDetails {
    private String accountId;
    private String accountName;
    private String accountNumber;
    private String balance;
    private String deposits;
    private String withdrawals;
    private String interestRate;
}
```

---

### **Recommendation for the Proposed Approach**

Based on the complexity and flexibility required for dynamic filtering, we recommend the **Jackson `@JsonFilter`** approach for the following reasons:

- **Flexibility**: It allows for easy integration into existing Spring Boot applications with minimal configuration.
- **Dynamic Filtering**: Filters can be dynamically applied based on `includeOnly` or `excludeOnly` parameters, without hardcoding field names.
- **No Additional Dependencies**: Unlike libraries like **Squiggly**, **Jackson** comes out-of-the-box with Spring Boot and doesn’t require third-party dependencies.

However, if you’re looking for an even simpler solution and **less customization** is required, the **Squiggly library** can provide a more concise implementation with automatic query parameter parsing.

---

### **Conclusion**
- **Jackson Filters** allow for **dynamic filtering** of both **top-level** and **nested** fields, ensuring we can apply `includeOnly` or `excludeOnly` query parameters.
- This approach can significantly reduce payload size, minimize bandwidth, and improve the performance of the Account service API.
  
