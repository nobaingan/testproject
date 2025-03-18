package com.example.demo.util;

import com.fasterxml.jackson.annotation.JsonFilter;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.ser.FilterProvider;
import com.fasterxml.jackson.databind.ser.impl.SimpleFilterProvider;
import org.junit.jupiter.api.Test;
import java.util.*;

import static org.junit.jupiter.api.Assertions.*;

public class DynamicFilterUtilTest {

    private final ObjectMapper objectMapper = new ObjectMapper();

    @JsonFilter("AccountDetailsFilter")
    public static class AccountDetails {
        public String accountNumber = "123456";
        public String accountType = "Savings";
        public List<Transaction> transactions = List.of(new Transaction());
        public Meta meta = new Meta();
    }

    @JsonFilter("TransactionFilter")
    public static class Transaction {
        public String transactionId = "txn123";
        public String transactionType = "Credit";
        public double amount = 1000.0;
    }

    @JsonFilter("MetaFilter")
    public static class Meta {
        public int totalTransactions = 5;
        public String lastUpdated = "2025-03-18";
    }

    @Test
    public void testIncludeOnly_SingleField() throws Exception {
        Set<String> includeFields = Set.of("accountNumber");
        Set<String> excludeFields = Set.of();

        FilterProvider filters = DynamicFilterUtil.createFilters(includeFields, excludeFields, AccountDetails.class);
        objectMapper.setFilterProvider(filters);

        String result = objectMapper.writeValueAsString(new AccountDetails());
        assertTrue(result.contains("accountNumber"));
        assertFalse(result.contains("accountType"));
        assertFalse(result.contains("transactions"));
        assertFalse(result.contains("meta"));
    }

    @Test
    public void testExcludeOnly_SingleField() throws Exception {
        Set<String> includeFields = Set.of();
        Set<String> excludeFields = Set.of("meta");

        FilterProvider filters = DynamicFilterUtil.createFilters(includeFields, excludeFields, AccountDetails.class);
        objectMapper.setFilterProvider(filters);

        String result = objectMapper.writeValueAsString(new AccountDetails());
        assertTrue(result.contains("accountNumber"));
        assertTrue(result.contains("accountType"));
        assertTrue(result.contains("transactions"));
        assertFalse(result.contains("meta"));
    }

    @Test
    public void testIncludeNestedField() throws Exception {
        Set<String> includeFields = Set.of("transactions.transactionId");
        Set<String> excludeFields = Set.of();

        FilterProvider filters = DynamicFilterUtil.createFilters(includeFields, excludeFields, AccountDetails.class);
        objectMapper.setFilterProvider(filters);

        String result = objectMapper.writeValueAsString(new AccountDetails());
        assertTrue(result.contains("transactions"));
        assertTrue(result.contains("transactionId"));
        assertFalse(result.contains("transactionType"));
        assertFalse(result.contains("amount"));
    }

    @Test
    public void testExcludeNestedField() throws Exception {
        Set<String> includeFields = Set.of();
        Set<String> excludeFields = Set.of("transactions.amount");

        FilterProvider filters = DynamicFilterUtil.createFilters(includeFields, excludeFields, AccountDetails.class);
        objectMapper.setFilterProvider(filters);

        String result = objectMapper.writeValueAsString(new AccountDetails());
        assertTrue(result.contains("transactions"));
        assertTrue(result.contains("transactionId"));
        assertTrue(result.contains("transactionType"));
        assertFalse(result.contains("amount"));
    }

    @Test
    public void testComplexIncludeAndExclude() throws Exception {
        Set<String> includeFields = Set.of("accountNumber", "transactions.transactionId");
        Set<String> excludeFields = Set.of("meta");

        FilterProvider filters = DynamicFilterUtil.createFilters(includeFields, excludeFields, AccountDetails.class);
        objectMapper.setFilterProvider(filters);

        String result = objectMapper.writeValueAsString(new AccountDetails());
        assertTrue(result.contains("accountNumber"));
        assertTrue(result.contains("transactions"));
        assertTrue(result.contains("transactionId"));
        assertFalse(result.contains("accountType"));
        assertFalse(result.contains("amount"));
        assertFalse(result.contains("meta"));
    }

    @Test
    public void testEmptyIncludeAndExclude() throws Exception {
        Set<String> includeFields = Set.of();
        Set<String> excludeFields = Set.of();

        FilterProvider filters = DynamicFilterUtil.createFilters(includeFields, excludeFields, AccountDetails.class);
        objectMapper.setFilterProvider(filters);

        String result = objectMapper.writeValueAsString(new AccountDetails());
        assertTrue(result.contains("accountNumber"));
        assertTrue(result.contains("accountType"));
        assertTrue(result.contains("transactions"));
        assertTrue(result.contains("meta"));
    }
}
