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

            // Handle includeOnly: Remove fields not in the include list
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
