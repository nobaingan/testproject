import com.fasterxml.jackson.databind.ObjectMapper;
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
    public MappingJacksonValue getAccountDetails(@PathVariable String accountNumber,
                                                @RequestParam(value = "includeOnly", required = false) String includeOnly,
                                                @RequestParam(value = "excludeOnly", required = false) String excludeOnly) throws Exception {

        // Sample data - replace with actual account fetching logic
        AccountDetailsResponse accountDetails = new AccountDetailsResponse();
        accountDetails.setAccountId("123456");
        accountDetails.setAccountName("John Doe");
        accountDetails.setAccountNumber("9876543210");
        accountDetails.setAccountSortCode("001122");
        accountDetails.setStatus("active");

        AccountDetails account = new AccountDetails();
        account.setAccountId("123456");
        account.setAccountName("John Doe");
        accountDetails.setAccount(account);

        // Determine the appropriate view based on include/exclude parameters
        Set<String> includeFields = parseFields(includeOnly);
        Set<String> excludeFields = parseFields(excludeOnly);

        // Apply dynamic filtering and map to corresponding view
        Object filteredResponse = applyDynamicView(accountDetails, includeFields, excludeFields);

        // Convert filtered response to JSON
        MappingJacksonValue mappingJacksonValue = new MappingJacksonValue(filteredResponse);
        if (!includeFields.isEmpty()) {
            mappingJacksonValue.setSerializationView(AccountView.Basic.class); // You can change this view based on your include/exclude logic
        } else {
            mappingJacksonValue.setSerializationView(AccountView.Extended.class); // Default view (can change as needed)
        }

        return mappingJacksonValue;
    }

    private Set<String> parseFields(String fields) {
        if (fields != null && !fields.isEmpty()) {
            return new HashSet<>(Arrays.asList(fields.split(",")));
        }
        return Collections.emptySet();
    }

    private Object applyDynamicView(Object responseObject, Set<String> includeFields, Set<String> excludeFields) {
        if (responseObject == null) {
            return null;
        }

        // Check if the object has fields that need filtering
        return filterFields(responseObject, includeFields, excludeFields);
    }

    private Object filterFields(Object responseObject, Set<String> includeFields, Set<String> excludeFields) {
        if (responseObject == null) {
            return null;
        }

        // Create a new instance of the class to hold the filtered response
        Object filteredObject;
        try {
            filteredObject = responseObject.getClass().getDeclaredConstructor().newInstance();
        } catch (Exception e) {
            throw new RuntimeException("Error creating filtered object", e);
        }

        // Get all fields of the current object
        Field[] fields = responseObject.getClass().getDeclaredFields();

        for (Field field : fields) {
            field.setAccessible(true);
            try {
                Object fieldValue = field.get(responseObject);

                // If it's a nested object, apply recursive filtering
                if (fieldValue != null && !field.getType().isPrimitive()) {
                    // Recursively filter nested objects
                    field.set(filteredObject, filterFields(fieldValue, includeFields, excludeFields));
                } else {
                    // Apply include/exclude logic based on the field name
                    if ((includeFields.isEmpty() || includeFields.contains(field.getName())) &&
                        (excludeFields.isEmpty() || !excludeFields.contains(field.getName()))) {
                        field.set(filteredObject, fieldValue);
                    }
                }
            } catch (IllegalAccessException e) {
                e.printStackTrace();
            }
        }

        return filteredObject;
    }
}
