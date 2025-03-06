Got it! You want to dynamically filter fields in your response using `includeOnly` and `excludeOnly` parameters in the controller. Let's achieve this using **Squiggly**, which provides an easy way to include and exclude specific fields.

---

## **🚀 Steps to Implement Dynamic Filtering with `includeOnly` & `excludeOnly`**
We will:
1. Add Squiggly to filter fields dynamically.
2. Modify the controller to accept **`includeOnly`** and **`excludeOnly`** query parameters.
3. Apply filtering based on these parameters.

---

### **1️⃣ Add Squiggly Dependency**
If you haven't already, add this to `pom.xml`:
```xml
<dependency>
    <groupId>com.github.bohnman</groupId>
    <artifactId>squiggly-filter-jackson</artifactId>
    <version>1.3.23</version>
</dependency>
```

For **Gradle**:
```gradle
implementation 'com.github.bohnman:squiggly-filter-jackson:1.3.23'
```

---

### **2️⃣ Configure Squiggly in `WebConfig.java`**
We need a filter to integrate Squiggly into Spring Boot.

```java
package com.example.demo.config;

import com.github.bohnman.squiggly.Squiggly;
import com.github.bohnman.squiggly.web.SquigglyRequestFilter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import javax.servlet.Filter;

@Configuration
public class WebConfig {
    
    @Bean
    public Filter squigglyRequestFilter() {
        return new SquigglyRequestFilter();
    }
}
```

---

### **3️⃣ Modify Controller to Use `includeOnly` & `excludeOnly`**
Now, we modify the controller to accept both **`includeOnly`** and **`excludeOnly`** as query parameters.

```java
package com.example.demo.controller;

import com.example.demo.model.dto.AccountDetailsResponse;
import com.example.demo.service.AccountService;
import com.github.bohnman.squiggly.Squiggly;
import com.github.bohnman.squiggly.parser.SquigglyParser;
import org.springframework.http.converter.json.MappingJacksonValue;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/accounts")
public class AccountController {

    private final AccountService accountService;

    public AccountController(AccountService accountService) {
        this.accountService = accountService;
    }

    @GetMapping("/{accountId}")
    public MappingJacksonValue getAccount(
            @PathVariable String accountId,
            @RequestParam(value = "includeOnly", required = false) String includeOnly,
            @RequestParam(value = "excludeOnly", required = false) String excludeOnly) {

        AccountDetailsResponse account = accountService.getAccountById(accountId);
        MappingJacksonValue mapping = new MappingJacksonValue(account);

        // Determine the Squiggly filter expression
        String filterExpression = buildSquigglyExpression(includeOnly, excludeOnly);

        if (!filterExpression.isEmpty()) {
            Squiggly.init(mapping, filterExpression);
        }

        return mapping;
    }

    private String buildSquigglyExpression(String includeOnly, String excludeOnly) {
        StringBuilder expression = new StringBuilder();

        if (includeOnly != null && !includeOnly.isEmpty()) {
            expression.append(includeOnly);
        }

        if (excludeOnly != null && !excludeOnly.isEmpty()) {
            String[] fieldsToExclude = excludeOnly.split(",");
            for (String field : fieldsToExclude) {
                if (!expression.isEmpty()) {
                    expression.append(",");
                }
                expression.append("-").append(field);
            }
        }

        return expression.toString();
    }
}
```

---

### **4️⃣ Example DTO Classes**
#### **AccountDetailsResponse.java**
```java
package com.example.demo.model.dto;

import com.fasterxml.jackson.annotation.JsonFilter;
import lombok.Builder;
import lombok.Value;

@Value
@Builder
@JsonFilter("dynamicFilter") // Required for filtering
public class AccountDetailsResponse {
    AccountDetails account;
}
```

#### **AccountDetails.java**
```java
package com.example.demo.model.dto;

import com.fasterxml.jackson.annotation.JsonFilter;
import lombok.Getter;
import lombok.Setter;
import java.util.List;

@Getter
@Setter
@JsonFilter("dynamicFilter")  // Required for filtering
public class AccountDetails {
    private String accountId;
    private String accountName;
    private String accountNumber;
    private String accountSortCode;
    private Product product;  
    private List<Transaction> transactions;
}
```

---

### **5️⃣ Test Cases**
**✅ Get Full Response**
```http
GET /accounts/12345
```
**Response:**
```json
{
    "account": {
        "accountId": "123456",
        "accountName": "Holiday",
        "accountNumber": "9876543210",
        "accountSortCode": "12345678",
        "product": {
            "productId": "P123",
            "productName": "Fixed Deposit"
        }
    }
}
```

---

**✅ Include Only `accountNumber` & `accountId`**
```http
GET /accounts/12345?includeOnly=account.accountNumber,account.accountId
```
**Response:**
```json
{
    "account": {
        "accountNumber": "9876543210",
        "accountId": "123456"
    }
}
```

---

**✅ Exclude Only `accountName`**
```http
GET /accounts/12345?excludeOnly=account.accountName
```
**Response:**
```json
{
    "account": {
        "accountId": "123456",
        "accountNumber": "9876543210",
        "accountSortCode": "12345678",
        "product": {
            "productId": "P123",
            "productName": "Fixed Deposit"
        }
    }
}
```
*(Notice `accountName` is missing)*

---

**✅ Include `accountNumber` and Exclude `accountSortCode`**
```http
GET /accounts/12345?includeOnly=account.accountNumber,account.accountId&excludeOnly=account.accountSortCode
```
**Response:**
```json
{
    "account": {
        "accountNumber": "9876543210",
        "accountId": "123456"
    }
}
```
---

### **🎯 Summary: Why Use This Approach?**
✅ **Supports both `includeOnly` and `excludeOnly`**  
✅ **Nested filtering works (`account.accountNumber`)**  
✅ **No need for complex custom Jackson filters**  
✅ **Easy to extend and maintain**  

🚀 **This is the best way to dynamically filter fields using Squiggly in Spring Boot!** 🎉 Let me know if you need any improvements! 🚀
