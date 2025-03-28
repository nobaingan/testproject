package com.example.demo.util;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.ser.FilterProvider;
import com.fasterxml.jackson.databind.ser.impl.SimpleFilterProvider;
import com.fasterxml.jackson.annotation.JsonFilter;
import lombok.AllArgsConstructor;
import lombok.Data;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.Collections;
import java.util.HashSet;
import java.util.Set;

import static org.junit.jupiter.api.Assertions.*;

@SpringBootTest
class DynamicFilterUtilTest {

    private static final Logger logger = LoggerFactory.getLogger(DynamicFilterUtilTest.class);
    private final ObjectMapper objectMapper = new ObjectMapper();

    /**
     * Sample DTO with @JsonFilter annotation.
     */
    @Data
    @JsonFilter("AccountDetailsFilter")
    @AllArgsConstructor
    static class AccountDetails {
        private String accountId;
        private String accountType;
        private double balance;
    }

    /**
     * Test case: Include only 'accountId' field in JSON response.
     */
    @Test
    void testIncludeOnlySingleField() throws JsonProcessingException {
        AccountDetails details = new AccountDetails("12345", "Savings", 1500.75);

        Set<String> includeFields = new HashSet<>(Collections.singletonList("accountId"));
        Set<String> excludeFields = Collections.emptySet();

        FilterProvider filter = DynamicFilterUtil.createFilters(includeFields, excludeFields, AccountDetails.class);
        objectMapper.setFilterProvider(filter);

        String jsonResult = objectMapper.writeValueAsString(details);
        logger.debug("Filtered JSON (Include accountId): {}", jsonResult);

        assertTrue(jsonResult.contains("accountId"));
        assertFalse(jsonResult.contains("accountType"));
        assertFalse(jsonResult.contains("balance"));
    }

    /**
     * Test case: Exclude 'balance' field from JSON response.
     */
    @Test
    void testExcludeSingleField() throws JsonProcessingException {
        AccountDetails details = new AccountDetails("12345", "Savings", 1500.75);

        Set<String> includeFields = Collections.emptySet();
        Set<String> excludeFields = new HashSet<>(Collections.singletonList("balance"));

        FilterProvider filter = DynamicFilterUtil.createFilters(includeFields, excludeFields, AccountDetails.class);
        objectMapper.setFilterProvider(filter);

        String jsonResult = objectMapper.writeValueAsString(details);
        logger.debug("Filtered JSON (Exclude balance): {}", jsonResult);

        assertTrue(jsonResult.contains("accountId"));
        assertTrue(jsonResult.contains("accountType"));
        assertFalse(jsonResult.contains("balance"));
    }

    /**
     * Test case: Include 'accountId' and 'balance', exclude 'accountType'.
     */
    @Test
    void testIncludeAndExcludeFields() throws JsonProcessingException {
        AccountDetails details = new AccountDetails("12345", "Savings", 1500.75);

        Set<String> includeFields = new HashSet<>(Set.of("accountId", "balance"));
        Set<String> excludeFields = new HashSet<>(Collections.singletonList("accountType"));

        FilterProvider filter = DynamicFilterUtil.createFilters(includeFields, excludeFields, AccountDetails.class);
        objectMapper.setFilterProvider(filter);

        String jsonResult = objectMapper.writeValueAsString(details);
        logger.debug("Filtered JSON (Include accountId, balance; Exclude accountType): {}", jsonResult);

        assertTrue(jsonResult.contains("accountId"));
        assertFalse(jsonResult.contains("accountType"));
        assertTrue(jsonResult.contains("balance"));
    }

    /**
     * Test case: Include all fields when includeFields is empty.
     */
    @Test
    void testIncludeAllFields() throws JsonProcessingException {
        AccountDetails details = new AccountDetails("12345", "Savings", 1500.75);

        Set<String> includeFields = Collections.emptySet();
        Set<String> excludeFields = Collections.emptySet();

        FilterProvider filter = DynamicFilterUtil.createFilters(includeFields, excludeFields, AccountDetails.class);
        objectMapper.setFilterProvider(filter);

        String jsonResult = objectMapper.writeValueAsString(details);
        logger.debug("Filtered JSON (Include all fields): {}", jsonResult);

        assertTrue(jsonResult.contains("accountId"));
        assertTrue(jsonResult.contains("accountType"));
        assertTrue(jsonResult.contains("balance"));
    }

    /**
     * Test case: Exclude all fields when all are in excludeFields.
     */
    @Test
    void testExcludeAllFields() throws JsonProcessingException {
        AccountDetails details = new AccountDetails("12345", "Savings", 1500.75);

        Set<String> includeFields = Collections.emptySet();
        Set<String> excludeFields = new HashSet<>(Set.of("accountId", "accountType", "balance"));

        FilterProvider filter = DynamicFilterUtil.createFilters(includeFields, excludeFields, AccountDetails.class);
        objectMapper.setFilterProvider(filter);

        String jsonResult = objectMapper.writeValueAsString(details);
        logger.debug("Filtered JSON (Exclude all fields): {}", jsonResult);

        assertEquals("{}", jsonResult);
    }
}
