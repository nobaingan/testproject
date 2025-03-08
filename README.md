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

        // Sample account data
        AccountDetailsResponse response = AccountDetailsResponse.builder()
                .account(new AccountDetails("12345", "Savings", "2025-12-31"))
                .tmDetails(new TMAccountDetails(5.5, 10000))
                .build();

        // Convert object to JSON tree
        ObjectNode jsonNode = objectMapper.valueToTree(response);

        // Convert include/exclude parameters to sets
        Set<String> includeFields = (includeOnly != null) ? new HashSet<>(Arrays.asList(includeOnly.split(","))) : null;
        Set<String> excludeFields = (excludeOnly != null) ? new HashSet<>(Arrays.asList(excludeOnly.split(","))) : null;

        // Apply filtering
        if (includeFields != null && !includeFields.isEmpty()) {
            retainOnlyIncludedFields(jsonNode, includeFields);
        }

        if (excludeFields != null && !excludeFields.isEmpty()) {
            removeExcludedFields(jsonNode, excludeFields);
        }

        return jsonNode.toPrettyString();
    }

    /**
     * Retains only the fields specified in includeFields.
     */
    private void retainOnlyIncludedFields(ObjectNode node, Set<String> includeFields) {
        Iterator<String> fieldNames = node.fieldNames();
        List<String> fieldsToRemove = new ArrayList<>();

        while (fieldNames.hasNext()) {
            String field = fieldNames.next();
            JsonNode childNode = node.get(field);

            // If includeOnly contains the whole object (e.g., "account"), keep it entirely
            boolean isParentObjectIncluded = includeFields.contains(field);

            if (!isParentObjectIncluded && !hasMatchingChild(field, includeFields)) {
                fieldsToRemove.add(field);
            }

            // If it's a nested object, process it recursively
            if (childNode.isObject() && !isParentObjectIncluded) {
                retainOnlyIncludedFields((ObjectNode) childNode, includeFields);
                if (childNode.isEmpty()) {
                    fieldsToRemove.add(field);
                }
            }
        }

        // Remove non-matching fields
        fieldsToRemove.forEach(node::remove);
    }

    /**
     * Recursively removes excluded fields.
     */
    private void removeExcludedFields(ObjectNode node, Set<String> excludeFields) {
        Iterator<String> fieldNames = node.fieldNames();
        List<String> fieldsToRemove = new ArrayList<>();

        while (fieldNames.hasNext()) {
            String field = fieldNames.next();
            JsonNode childNode = node.get(field);

            if (excludeFields.contains(field)) {
                fieldsToRemove.add(field);
            }

            if (childNode.isObject()) {
                removeExcludedFields((ObjectNode) childNode, excludeFields);
            }
        }

        fieldsToRemove.forEach(node::remove);
    }

    /**
     * Checks if any child field of a parent is in the includeFields set.
     */
    private boolean hasMatchingChild(String parentField, Set<String> includeFields) {
        return includeFields.stream().anyMatch(field -> field.startsWith(parentField + "."));
    }
}
