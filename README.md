import java.lang.reflect.Constructor;
import java.util.Set;
import java.util.HashSet;
import java.util.Collections;

public class AccountController {

    // Recursive filtering function
    private Object filterFields(Object responseObject, Set<String> includeFields, Set<String> excludeFields) {
        if (responseObject == null) {
            return null;
        }

        // Create a new instance of the class to hold the filtered response
        Object filteredObject = null;
        try {
            filteredObject = createNewInstance(responseObject.getClass());

            // Get all fields of the current object
            for (var field : responseObject.getClass().getDeclaredFields()) {
                field.setAccessible(true);
                try {
                    Object fieldValue = field.get(responseObject);

                    // If it's a nested object, apply recursive filtering
                    if (fieldValue != null && !field.getType().isPrimitive() && !field.getType().equals(String.class)) {
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
        } catch (Exception e) {
            e.printStackTrace();
        }
        return filteredObject;
    }

    private Object createNewInstance(Class<?> clazz) throws Exception {
        // Only create a new instance if it's not an abstract class or interface
        if (clazz.isInterface() || java.lang.reflect.Modifier.isAbstract(clazz.getModifiers())) {
            throw new IllegalArgumentException("Cannot create an instance of an interface or abstract class.");
        }

        try {
            // Attempt to use a no-argument constructor
            Constructor<?> constructor = clazz.getDeclaredConstructor();
            return constructor.newInstance();
        } catch (NoSuchMethodException e) {
            // Handle the case where the no-argument constructor doesn't exist
            // Try finding a constructor that accepts parameters
            Constructor<?> constructorWithParams = clazz.getDeclaredConstructor(String.class, String.class);
            return constructorWithParams.newInstance("defaultAccountId", "defaultAccountName");
        }
    }

    private Set<String> parseFields(String fields) {
        if (fields != null && !fields.isEmpty()) {
            return new HashSet<>(fields.split(","));
        }
        return Collections.emptySet();
    }

    // The controller method, as before
}
