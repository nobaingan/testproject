Here are the **two complete solutions** for implementing filtering in a Spring Boot application:  

1. **Using `OncePerRequestFilter` (Filter-Based Approach)**
2. **Using `HandlerInterceptor` (Interceptor-Based Approach)**  

Both approaches support `includeOnly` and `excludeOnly` parameters for filtering response data.

---

## ✅ **Solution 1: Using `OncePerRequestFilter` (Filter-Based)**
This approach wraps the response, applies filtering, and modifies the response **before sending it to the client**.

### **1️⃣ `FilterInterceptor.java` (Filter-Based)**
```java
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.http.converter.json.MappingJacksonValue;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;
import org.springframework.web.util.ContentCachingResponseWrapper;

import javax.servlet.FilterChain;
import javax.servlet.ServletException;
import javax.servlet.ServletOutputStream;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;

@Component
public class FilterInterceptor extends OncePerRequestFilter {

    private final ObjectMapper objectMapper;

    public FilterInterceptor(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {

        ContentCachingResponseWrapper responseWrapper = new ContentCachingResponseWrapper(response);

        try {
            filterChain.doFilter(request, responseWrapper);

            byte[] responseBody = responseWrapper.getContentAsByteArray();
            if (responseBody.length > 0) {
                // Read the response body and apply filtering
                Object originalResponse = objectMapper.readValue(responseBody, Object.class);
                MappingJacksonValue filteredResponse = FilteringUtil.applyFiltering(request, originalResponse);

                // Reset response and write filtered content
                responseWrapper.resetBuffer();
                responseWrapper.setContentType("application/json");
                ServletOutputStream outputStream = responseWrapper.getOutputStream();
                objectMapper.writeValue(outputStream, filteredResponse.getValue());

                responseWrapper.copyBodyToResponse();
            }
        } catch (Exception e) {
            responseWrapper.copyBodyToResponse();
            throw e;
        }
    }
}
```

### **2️⃣ `WebConfig.java` (Registering Filter)**
```java
import org.springframework.boot.web.servlet.FilterRegistrationBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class WebConfig {

    @Bean
    public FilterRegistrationBean<FilterInterceptor> filterRegistrationBean(FilterInterceptor filterInterceptor) {
        FilterRegistrationBean<FilterInterceptor> registrationBean = new FilterRegistrationBean<>();
        registrationBean.setFilter(filterInterceptor);
        registrationBean.addUrlPatterns("/accounts/*");  // Apply filter only for account endpoints
        return registrationBean;
    }
}
```

### **3️⃣ `FilteringUtil.java` (Applying Filters)**
```java
import com.fasterxml.jackson.databind.ser.impl.SimpleBeanPropertyFilter;
import com.fasterxml.jackson.databind.ser.impl.SimpleFilterProvider;
import com.fasterxml.jackson.http.converter.json.MappingJacksonValue;

import javax.servlet.http.HttpServletRequest;
import java.util.Arrays;
import java.util.HashSet;
import java.util.Set;

public class FilteringUtil {

    public static MappingJacksonValue applyFiltering(HttpServletRequest request, Object responseData) {
        String includeOnly = request.getParameter("includeOnly");
        String excludeOnly = request.getParameter("excludeOnly");

        SimpleFilterProvider filterProvider = new SimpleFilterProvider();

        Set<String> includeFields = includeOnly != null ? new HashSet<>(Arrays.asList(includeOnly.split(","))) : null;
        Set<String> excludeFields = excludeOnly != null ? new HashSet<>(Arrays.asList(excludeOnly.split(","))) : null;

        if (includeFields != null) {
            filterProvider.addFilter("dynamicFilter", SimpleBeanPropertyFilter.filterOutAllExcept(includeFields));
        } else if (excludeFields != null) {
            filterProvider.addFilter("dynamicFilter", SimpleBeanPropertyFilter.serializeAllExcept(excludeFields));
        } else {
            filterProvider.addFilter("dynamicFilter", SimpleBeanPropertyFilter.serializeAll());
        }

        MappingJacksonValue mapping = new MappingJacksonValue(responseData);
        mapping.setFilters(filterProvider);
        return mapping;
    }
}
```

---

## ✅ **Solution 2: Using `HandlerInterceptor` (Interceptor-Based)**
This approach intercepts controller responses and **modifies the response before returning it to the client**.

### **1️⃣ `FilterInterceptor.java` (Interceptor-Based)**
```java
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.http.converter.json.MappingJacksonValue;
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.HandlerInterceptor;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

@Component
public class FilterInterceptor implements HandlerInterceptor {

    private final ObjectMapper objectMapper;

    public FilterInterceptor(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        return true;  // Continue request processing
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, org.springframework.web.servlet.ModelAndView modelAndView) throws Exception {
        Object originalResponse = request.getAttribute("filteredResponse");
        if (originalResponse != null) {
            MappingJacksonValue filteredResponse = FilteringUtil.applyFiltering(request, originalResponse);
            response.setContentType("application/json");
            objectMapper.writeValue(response.getWriter(), filteredResponse.getValue());
        }
    }
}
```

### **2️⃣ `WebConfig.java` (Registering Interceptor)**
```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Autowired
    private FilterInterceptor filterInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(filterInterceptor);
    }
}
```

### **3️⃣ `AccountController.java` (Using the Interceptor)**
```java
import org.springframework.http.converter.json.MappingJacksonValue;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/accounts")
public class AccountController {

    private final ObjectMapper objectMapper;

    public AccountController(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }

    @GetMapping("/{accountId}")
    public MappingJacksonValue getAccount(
            @PathVariable String accountId,
            @RequestParam(value = "includeOnly", required = false) String includeOnly,
            @RequestParam(value = "excludeOnly", required = false) String excludeOnly,
            HttpServletRequest request) {

        AccountDetails account = new AccountDetails("123456", "John Doe", "9876543210", "001122");
        AccountDetailsResponse response = new AccountDetailsResponse(account);

        MappingJacksonValue filteredResponse = FilteringUtil.applyFiltering(request, response);
        request.setAttribute("filteredResponse", filteredResponse);

        return filteredResponse;  // The interceptor will modify this before sending the response
    }
}
```

---

## **📌 Key Differences**
| Feature | **Filter-Based (Solution 1)** | **Interceptor-Based (Solution 2)** |
|---------|------------------------------|----------------------------------|
| **Processing Level** | Low-level (modifies HTTP response directly) | High-level (intercepts after controller processing) |
| **Best For** | Global response filtering | Controller-specific response filtering |
| **Customization** | Works for all responses | Works for API responses only |
| **Performance** | Slightly more overhead due to response caching | Lighter weight (directly modifies controller output) |
| **Complexity** | More complex due to `ContentCachingResponseWrapper` | Easier to implement |

---

## **✅ Final Recommendation**
- If you **want a global filter** that applies to all responses, use **Solution 1 (Filter-Based)**.
- If you **only need to filter controller responses**, use **Solution 2 (Interceptor-Based)**.

Both solutions properly handle `includeOnly` and `excludeOnly` dynamically. 🚀 Let me know if you need further refinements!
