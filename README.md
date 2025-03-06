import com.fasterxml.jackson.databind.ser.impl.SimpleBeanPropertyFilter;
import com.fasterxml.jackson.databind.ser.impl.SimpleFilterProvider;
import com.fasterxml.jackson.databind.ser.FilterProvider;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.annotation.JsonFilter;
import org.springframework.http.converter.json.MappingJacksonValue;
import org.springframework.web.bind.annotation.*;

import java.util.*;

@RestController
@RequestMapping("/accounts")
public class AccountController {

    @GetMapping("/{accountId}")
    public MappingJacksonValue getAccount(
            @PathVariable String accountId,
            @RequestParam(value = "includeOnly", required = false) String includeOnly,
            @RequestParam(value = "excludeOnly", required = false) String excludeOnly) {

        // Sample nested data
        AccountDetails account = new AccountDetails();
        account.setAccountId("123456");
        account.setAccountName("John Doe");
        account.setAccountNumber("9876543210");
        account.setAccountSortCode("001122");

        AccountDetailsResponse response = new AccountDetailsResponse();
        response.setAccount(account);

        // Apply dynamic filtering
        MappingJacksonValue mapping = applyFiltering(response, includeOnly, excludeOnly);
        return mapping;
    }

    private MappingJacksonValue applyFiltering(Object responseData, String includeOnly, String excludeOnly) {
        SimpleFilterProvider filterProvider = new SimpleFilterProvider();

        // Convert comma-separated values to a List
        Set<String> includeFields = (includeOnly != null && !includeOnly.isEmpty()) ?
                new HashSet<>(Arrays.asList(includeOnly.split(","))) : null;

        Set<String> excludeFields = (excludeOnly != null && !excludeOnly.isEmpty()) ?
                new HashSet<>(Arrays.asList(excludeOnly.split(","))) : null;

        // Apply Include-Only or Exclude-Only Filtering
        if (includeFields != null) {
            System.out.println("Applying INCLUDE ONLY filter: " + includeFields);
            filterProvider.addFilter("dynamicFilter",
                    SimpleBeanPropertyFilter.filterOutAllExcept(includeFields));
        } else if (excludeFields != null) {
            System.out.println("Applying EXCLUDE ONLY filter: " + excludeFields);
            filterProvider.addFilter("dynamicFilter",
                    SimpleBeanPropertyFilter.serializeAllExcept(excludeFields));
        } else {
            System.out.println("No filters applied. Returning all fields.");
            filterProvider.addFilter("dynamicFilter", SimpleBeanPropertyFilter.serializeAll());
        }

        MappingJacksonValue mapping = new MappingJacksonValue(responseData);
        mapping.setFilters(filterProvider);
        return mapping;
    }
}
