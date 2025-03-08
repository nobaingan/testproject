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
        Account account = new Account();

        // Convert object to JSON tree
        ObjectNode jsonNode = objectMapper.valueToTree(account);

        // Convert include/exclude parameters to sets
        Set<String> includeFields = (includeOnly != null) ? new HashSet<>(Arrays.asList(includeOnly.split(","))) : null;
        Set<String> excludeFields = (excludeOnly != null) ? new HashSet<>(Arrays.asList(excludeOnly.split(","))) : null;

        // Apply filtering
        if (includeFields != null && !includeFields.isEmpty()) {
            retainOnlyIncludedFields(jsonNode, includeFields, "");
        }

        if (excludeFields != null && !excludeFields.isEmpty()) {
            removeExcludedFields(jsonNode, excludeFields, "");
        }

        return jsonNode.toPrettyString();
    }

    /**
     * Recursively removes all fields that are NOT in includeFields.
     */
    private void retainOnlyIncludedFields(ObjectNode node, Set<String> includeFields, String parentPath) {
        Iterator<String> fieldNames = node.fieldNames();
        List<String> fieldsToRemove = new ArrayList<>();

        while (fieldNames.hasNext()) {
            String field = fieldNames.next();
            String fullPath = parentPath.isEmpty() ? field : parentPath + "." + field;
            JsonNode childNode = node.get(field);

            // If this field is NOT in includeFields, mark for removal
            if (!matchesField(fullPath, includeFields)) {
                fieldsToRemove.add(field);
            }

            // If it's an object, recursively process it
            if (childNode.isObject()) {
                retainOnlyIncludedFields((ObjectNode) childNode, includeFields, fullPath);

                // Remove empty objects
                if (childNode.isEmpty()) {
                    fieldsToRemove.add(field);
                }
            }
        }

        // Remove non-matching fields
        fieldsToRemove.forEach(node::remove);
    }

    /**
     * Recursively removes fields specified in excludeFields.
     */
    private void removeExcludedFields(ObjectNode node, Set<String> excludeFields, String parentPath) {
        Iterator<String> fieldNames = node.fieldNames();
        List<String> fieldsToRemove = new ArrayList<>();

        while (fieldNames.hasNext()) {
            String field = fieldNames.next();
            String fullPath = parentPath.isEmpty() ? field : parentPath + "." + field;
            JsonNode childNode = node.get(field);

            // If this field is in excludeFields, mark for removal
            if (matchesField(fullPath, excludeFields)) {
                fieldsToRemove.add(field);
            }

            // If it's an object, recursively process it
            if (childNode.isObject()) {
                removeExcludedFields((ObjectNode) childNode, excludeFields, fullPath);
            }
        }

        // Remove fields marked for exclusion
        fieldsToRemove.forEach(node::remove);
    }

    /**
     * Checks if a field or its parent exists in the include/exclude set.
     */
    private boolean matchesField(String field, Set<String> fieldSet) {
        return fieldSet.contains(field) || fieldSet.stream().anyMatch(field::startsWith);
    }
}
