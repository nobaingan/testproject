# **Dynamic Filtering in Spring Boot - Design Decision Document**  

## **Overview**  
APIs often need to return different fields based on client needs. Instead of creating multiple endpoints, dynamic filtering allows API consumers to specify which fields they want to include or exclude.  

This document explains five approaches for implementing dynamic filtering in Spring Boot, provides sample implementations, and includes a **detailed comparison table** to help decide the best approach.  

---

## **Why Dynamic Filtering?**  
### **Use Case Examples**  
1. **Excluding Specific Fields**:  
   - A consumer does **not** want `interest`, `deposits`, and `withdrawals` in the response.  
   - **API Request:**  
     ```
     GET /accounts/12345?excludeOnly=interest,deposits,withdrawals
     ```
   - **Response:**  
     ```json
     {
         "accountNumber": "12345",
         "accountName": "John's Savings",
         "product": "Fixed Deposit",
         "maturityDate": "2025-12-31"
     }
     ```

2. **Including Specific Fields**:  
   - A consumer only wants `accountName`, `product`, and `maturityDate`.  
   - **API Request:**  
     ```
     GET /accounts/12345?includeOnly=accountName,product,maturityDate
     ```
   - **Response:**  
     ```json
     {
         "accountName": "John's Savings",
         "product": "Fixed Deposit",
         "maturityDate": "2025-12-31"
     }
     ```

---

## **Approaches for Dynamic Filtering**  

### **1. Jackson @JsonFilter (Dynamic Filtering)**
- Uses **@JsonFilter** annotation and `MappingJacksonValue` for **runtime filtering**.
- Works well when API consumers specify `includeOnly` or `excludeOnly` fields in query parameters.

#### **Implementation**  
```java
@JsonFilter("accountFilter")
public class Account {
    private String accountNumber;
    private String accountName;
    private String product;
    private String maturityDate;
    private double interest;
    private double deposits;
    private double withdrawals;
}
```

```java
@GetMapping("/{accountNumber}")
public MappingJacksonValue getAccount(
        @PathVariable String accountNumber,
        @RequestParam(required = false) String includeOnly,
        @RequestParam(required = false) String excludeOnly) {
    
    Account account = new Account();
    Set<String> fieldsToInclude = includeOnly != null ? new HashSet<>(Arrays.asList(includeOnly.split(","))) : null;
    Set<String> fieldsToExclude = excludeOnly != null ? new HashSet<>(Arrays.asList(excludeOnly.split(","))) : null;

    SimpleBeanPropertyFilter filter = fieldsToInclude != null ? 
        SimpleBeanPropertyFilter.filterOutAllExcept(fieldsToInclude) : 
        SimpleBeanPropertyFilter.serializeAllExcept(fieldsToExclude);

    FilterProvider filters = new SimpleFilterProvider().addFilter("accountFilter", filter);
    MappingJacksonValue mapping = new MappingJacksonValue(account);
    mapping.setFilters(filters);

    return mapping;
}
```

✅ **Pros:**  
✔️ Works at **runtime**, flexible for API users.  
✔️ Allows **both include and exclude** field options.  

⚠️ **Cons:**  
❌ Requires `MappingJacksonValue`, which is **not intuitive**.  
❌ Can be **complex when managing multiple filters**.  

---

### **2. Jackson Mix-Ins**  
- Allows defining filters **outside the model class** using a separate mix-in class.
- Useful for **static filtering scenarios** (predefined fields to ignore).

#### **Implementation**  
```java
@JsonIgnoreProperties({"interest", "deposits", "withdrawals"})
public abstract class AccountMixin {}
```

```java
@Bean
public ObjectMapper objectMapper() {
    ObjectMapper objectMapper = new ObjectMapper();
    objectMapper.addMixIn(Account.class, AccountMixin.class);
    return objectMapper;
}
```

✅ **Pros:**  
✔️ Keeps filtering **separate** from the model.  
✔️ Avoids modifying the original **Account class**.  

⚠️ **Cons:**  
❌ Cannot be changed at **runtime**.  
❌ Requires configuring **ObjectMapper manually**.  

---

### **3. Jackson @JsonView**  
- Uses predefined **views** to control which fields are included in different API responses.
- Useful when **different user roles** need different views.

#### **Implementation**  
```java
public class Views {
    public static class Public {}
    public static class Private extends Public {}
}
```

```java
public class Account {
    @JsonView(Views.Public.class)
    private String accountName;

    @JsonView(Views.Private.class)
    private String product;
}
```

```java
@GetMapping("/{accountNumber}")
@JsonView(Views.Public.class)
public Account getAccount(@PathVariable String accountNumber) {
    return new Account();
}
```

✅ **Pros:**  
✔️ **Built-in Jackson feature**, no need for extra configuration.  
✔️ Good for **predefined roles** (e.g., Admin sees more fields than Users).  

⚠️ **Cons:**  
❌ **No runtime flexibility**.  
❌ Hard to manage when there are **too many views**.  

---

### **4. Squiggly Library**  
- Allows filtering using **query parameters** like `fields=accountName,product`.
- Works without modifying controllers or models.

#### **Implementation**  
```xml
<dependency>
    <groupId>com.github.bohnman</groupId>
    <artifactId>squiggly-filter-jackson</artifactId>
    <version>1.3.21</version>
</dependency>
```

```java
@Bean
public ObjectMapper objectMapper() {
    ObjectMapper objectMapper = new ObjectMapper();
    Squiggly.init(objectMapper, new RequestSquigglyContextProvider());
    return objectMapper;
}
```

✅ **Pros:**  
✔️ **Simple** to use, based on query parameters.  
✔️ No need to modify **controller or model**.  

⚠️ **Cons:**  
❌ Requires an **external library**.  
❌ May **impact performance** for complex queries.  

---

### **5. GraphQL**  
- API consumers **request only the fields** they need, no filtering logic in backend.
- Best for **modern APIs with evolving data needs**.

#### **Implementation**  
```graphql
type Account {
    accountNumber: String
    accountName: String
    product: String
    maturityDate: String
}

type Query {
    accounts: [Account]
}
```

```java
public class AccountResolver {
    public List<Account> getAccounts() {
        return List.of(new Account());
    }
}
```

✅ **Pros:**  
✔️ **Maximum flexibility** (client decides fields).  
✔️ **Efficient data fetching**, only requested fields are queried.  

⚠️ **Cons:**  
❌ Requires **GraphQL setup** and new **query language**.  
❌ May not be ideal for **simple APIs**.  

---

## **Decision Making Table**  

| Approach               | How It Works | Best For | Pros | Cons |
|-----------------------|-------------|---------|------|------|
| **Jackson @JsonFilter** | Uses `@JsonFilter` and `MappingJacksonValue` to include/exclude fields at runtime. | APIs needing **runtime filtering** via query params. | ✔️ Fully dynamic <br> ✔️ Supports both include & exclude | ❌ Needs `MappingJacksonValue` <br> ❌ More complex to set up |
| **Jackson Mix-Ins** | Defines separate classes for filtering fields | **Static filtering** (predefined rules) | ✔️ Keeps model **clean** <br> ✔️ Easy to configure | ❌ **No runtime flexibility** <br> ❌ Requires `ObjectMapper` setup |
| **Jackson @JsonView** | Uses `@JsonView` to define different field groups | **Role-based field visibility** | ✔️ **Built-in Jackson support** <br> ✔️ Good for user roles | ❌ **Not dynamic** <br> ❌ Becomes hard to manage |
| **Squiggly Library** | Uses query params like `fields=field1,field2` | **Query-based filtering** | ✔️ **Simple and intuitive** <br> ✔️ No controller changes | ❌ Requires **external library** <br> ❌ Performance impact |
| **GraphQL** | Clients request only the fields they need | **Modern APIs** with flexible data needs | ✔️ **Maximum flexibility** <br> ✔️ **Efficient data fetching** | ❌ **Requires GraphQL setup** <br> ❌ **Overkill for simple APIs** |

### **Conclusion:**  
- **For flexible filtering:** `@JsonFilter` or **Squiggly**.  
- **For static rules:** **Mix-Ins** or `@JsonView`.  
- **For modern APIs:** **GraphQL**.  

Choosing the best approach depends on the API design and business needs.

No, the current implementation using `ObjectNode.retain()` and `ObjectNode.remove()` works only for **top-level fields**. If `interest` and `withdrawals` are **nested objects**, the approach needs to be modified to handle **nested field filtering dynamically**.

---

## **Solution for Nested Fields Filtering in Jackson Mix-In**
To support **nested fields** (e.g., `interest.rate`, `withdrawals.amount`), we need to:
1. **Parse nested field names from `includeOnly` and `excludeOnly`**.
2. **Recursively process the JSON structure** to apply filtering at any depth.

---

### **Step 1: Modify the `Account` Class**
Make `interest` and `withdrawals` **nested objects** instead of simple fields.

```java
public class Account {
    private String accountNumber;
    private String accountName;
    private String product;
    private String maturityDate;
    private Interest interest;        // Nested Object
    private Transaction withdrawals;  // Nested Object
    private Transaction deposits;     // Nested Object

    // Constructor
    public Account() {
        this.accountNumber = "12345";
        this.accountName = "John's Savings";
        this.product = "Fixed Deposit";
        this.maturityDate = "2025-12-31";
        this.interest = new Interest(5.5);
        this.withdrawals = new Transaction(2000);
        this.deposits = new Transaction(10000);
    }

    // Getters and Setters
}

// Nested class for Interest
class Interest {
    private double rate;

    public Interest(double rate) {
        this.rate = rate;
    }

    public double getRate() {
        return rate;
    }
}

// Nested class for Transactions (Withdrawals/Deposits)
class Transaction {
    private double amount;

    public Transaction(double amount) {
        this.amount = amount;
    }

    public double getAmount() {
        return amount;
    }
}
```

---

### **Step 2: Modify the Controller to Handle Nested Fields**
Instead of using `ObjectNode.retain()` and `ObjectNode.remove()`, we need **recursive field filtering**.

```java
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import org.springframework.web.bind.annotation.*;

import java.util.*;

@RestController
@RequestMapping("/accounts")
public class AccountController {

    private final ObjectMapper objectMapper;

    public AccountController(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }

    @GetMapping("/{accountNumber}")
    public String getAccount(
            @PathVariable String accountNumber,
            @RequestParam(required = false) String includeOnly,
            @RequestParam(required = false) String excludeOnly) throws Exception {

        // Create an account instance
        Account account = new Account();

        // Convert object to JSON tree
        ObjectNode jsonNode = objectMapper.valueToTree(account);

        // Convert include/exclude parameters to sets
        Set<String> includeFields = includeOnly != null ? new HashSet<>(Arrays.asList(includeOnly.split(","))) : null;
        Set<String> excludeFields = excludeOnly != null ? new HashSet<>(Arrays.asList(excludeOnly.split(","))) : null;

        // Apply filtering (recursive function)
        filterJson(jsonNode, includeFields, excludeFields, "");

        return jsonNode.toPrettyString();
    }

    /**
     * Recursively filters JSON fields.
     */
    private void filterJson(ObjectNode node, Set<String> includeFields, Set<String> excludeFields, String parentPath) {
        Iterator<String> fieldNames = node.fieldNames();
        List<String> fieldsToRemove = new ArrayList<>();

        while (fieldNames.hasNext()) {
            String field = fieldNames.next();
            String fullPath = parentPath.isEmpty() ? field : parentPath + "." + field;
            JsonNode childNode = node.get(field);

            // Recursively process nested objects
            if (childNode.isObject()) {
                filterJson((ObjectNode) childNode, includeFields, excludeFields, fullPath);
            }

            // Handle includeOnly: Remove fields not in include list
            if (includeFields != null && !includeFields.isEmpty() && !matchesField(fullPath, includeFields)) {
                fieldsToRemove.add(field);
            }

            // Handle excludeOnly: Remove fields present in exclude list
            if (excludeFields != null && matchesField(fullPath, excludeFields)) {
                fieldsToRemove.add(field);
            }
        }

        // Remove collected fields
        fieldsToRemove.forEach(node::remove);
    }

    /**
     * Checks if a field or its parent exists in the include/exclude set.
     */
    private boolean matchesField(String field, Set<String> fieldSet) {
        return fieldSet.contains(field) || fieldSet.stream().anyMatch(field::startsWith);
    }
}
```

---

## **Step 3: Test Nested Field Filtering**

### **1. Default Response (Full Account Data)**
#### **Request:**
```http
GET /accounts/12345
```
#### **Response:**
```json
{
  "accountNumber": "12345",
  "accountName": "John's Savings",
  "product": "Fixed Deposit",
  "maturityDate": "2025-12-31",
  "interest": {
    "rate": 5.5
  },
  "withdrawals": {
    "amount": 2000
  },
  "deposits": {
    "amount": 10000
  }
}
```

---

### **2. Exclude Only `interest.rate` and `withdrawals.amount`**
#### **Request:**
```http
GET /accounts/12345?excludeOnly=interest.rate,withdrawals.amount
```
#### **Response:**
```json
{
  "accountNumber": "12345",
  "accountName": "John's Savings",
  "product": "Fixed Deposit",
  "maturityDate": "2025-12-31",
  "interest": {},
  "withdrawals": {},
  "deposits": {
    "amount": 10000
  }
}
```
(*Note: `interest` and `withdrawals` exist but are empty because only `rate` and `amount` were removed*)

---

### **3. Include Only `accountName`, `product`, and `interest.rate`**
#### **Request:**
```http
GET /accounts/12345?includeOnly=accountName,product,interest.rate
```
#### **Response:**
```json
{
  "accountName": "John's Savings",
  "product": "Fixed Deposit",
  "interest": {
    "rate": 5.5
  }
}
```

---

## **Final Summary**
| Feature | Description |
|---------|-------------|
| **Approach** | Uses **Jackson Mix-In** + recursive JSON filtering for **nested objects**. |
| **Include Only** | `?includeOnly=field1,field2.subfield` allows selecting **nested fields** dynamically. |
| **Exclude Only** | `?excludeOnly=field1,field2.subfield` allows **hiding nested fields** dynamically. |
| **Advantages** | - Works for **deeply nested objects**. <br> - Handles dynamic **inclusion & exclusion** in one API. <br> - Efficient and **extensible** for any model. |
| **Limitations** | - Slight performance overhead for **deeply nested filtering**. <br> - Must ensure field paths **match the JSON structure** correctly. |

---

### **Conclusion**
This updated solution **fully supports nested object filtering** using `includeOnly` and `excludeOnly`. It ensures flexibility in API responses, giving consumers control over the returned fields.

Would you like **pagination, performance optimizations, or security considerations** added to this?
