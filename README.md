package com.example.demo.util;

import com.fasterxml.jackson.annotation.JsonFilter;
import com.fasterxml.jackson.databind.ser.FilterProvider;
import com.fasterxml.jackson.databind.ser.impl.SimpleFilterProvider;
import org.junit.jupiter.api.Test;
import java.util.*;

import static org.junit.jupiter.api.Assertions.*;

public class DynamicFilterUtilTest {

    // Sample DTO classes for testing
    @JsonFilter("AccountDetailsFilter")
    public static class AccountDetails {
        public String accountNumber;
        public String accountType;
        public List<Transaction> transactions;
        public Meta meta;
    }

    @JsonFilter("TransactionFilter")
    public static class Transaction {
        public String transactionId;
        public String transactionType;
        public double amount;
    }

    @JsonFilter("MetaFilter")
    public static class Meta {
        public int totalTransactions;
        public String lastUpdated;
    }

    @Test
    public void testIncludeOnly_SingleField() {
        Set<String> includeFields = Set.of("accountNumber");
        Set<String> excludeFields = Set.of();

        FilterProvider filters = DynamicFilterUtil.createFilters(
                includeFields, excludeFields, AccountDetails.class);

        SimpleFilterProvider filterProvider = (SimpleFilterProvider) filters;
        assertTrue(filterProvider.findPropertyFilter("AccountDetailsFilter", null)
                .includeAll());
        assertTrue(filterProvider.findPropertyFilter("AccountDetailsFilter", null)
                .include("accountNumber"));
    }

    @Test
    public void testExcludeOnly_SingleField() {
        Set<String> includeFields = Set.of();
        Set<String> excludeFields = Set.of("meta");

        FilterProvider filters = DynamicFilterUtil.createFilters(
                includeFields, excludeFields, AccountDetails.class);

        SimpleFilterProvider filterProvider = (SimpleFilterProvider) filters;
        assertTrue(filterProvider.findPropertyFilter("AccountDetailsFilter", null)
                .serializeAllExcept(Set.of("meta")));
    }

    @Test
    public void testIncludeNestedField() {
        Set<String> includeFields = Set.of("transactions.transactionId");
        Set<String> excludeFields = Set.of();

        FilterProvider filters = DynamicFilterUtil.createFilters(
                includeFields, excludeFields, AccountDetails.class);

        SimpleFilterProvider filterProvider = (SimpleFilterProvider) filters;
        assertTrue(filterProvider.findPropertyFilter("TransactionFilter", null)
                .include("transactionId"));
    }

    @Test
    public void testExcludeNestedField() {
        Set<String> includeFields = Set.of();
        Set<String> excludeFields = Set.of("transactions.amount");

        FilterProvider filters = DynamicFilterUtil.createFilters(
                includeFields, excludeFields, AccountDetails.class);

        SimpleFilterProvider filterProvider = (SimpleFilterProvider) filters;
        assertTrue(filterProvider.findPropertyFilter("TransactionFilter", null)
                .serializeAllExcept(Set.of("amount")));
    }

    @Test
    public void testIncludeCollectionElements() {
        Set<String> includeFields = Set.of("transactions.transactionType");
        Set<String> excludeFields = Set.of();

        FilterProvider filters = DynamicFilterUtil.createFilters(
                includeFields, excludeFields, AccountDetails.class);

        SimpleFilterProvider filterProvider = (SimpleFilterProvider) filters;
        assertTrue(filterProvider.findPropertyFilter("TransactionFilter", null)
                .include("transactionType"));
    }

    @Test
    public void testComplexIncludeAndExclude() {
        Set<String> includeFields = Set.of("accountNumber", "transactions.transactionId");
        Set<String> excludeFields = Set.of("meta");

        FilterProvider filters = DynamicFilterUtil.createFilters(
                includeFields, excludeFields, AccountDetails.class);

        SimpleFilterProvider filterProvider = (SimpleFilterProvider) filters;
        
        // AccountDetails Filter Assertions
        assertTrue(filterProvider.findPropertyFilter("AccountDetailsFilter", null)
                .include("accountNumber"));
        assertFalse(filterProvider.findPropertyFilter("AccountDetailsFilter", null)
                .include("meta"));

        // Transaction Filter Assertions
        assertTrue(filterProvider.findPropertyFilter("TransactionFilter", null)
                .include("transactionId"));
        assertFalse(filterProvider.findPropertyFilter("TransactionFilter", null)
                .include("transactionType"));
    }

    @Test
    public void testEmptyIncludeAndExclude() {
        Set<String> includeFields = Set.of();
        Set<String> excludeFields = Set.of();

        FilterProvider filters = DynamicFilterUtil.createFilters(
                includeFields, excludeFields, AccountDetails.class);

        SimpleFilterProvider filterProvider = (SimpleFilterProvider) filters;

        // Should include all fields by default
        assertTrue(filterProvider.findPropertyFilter("AccountDetailsFilter", null)
                .includeAll());
        assertTrue(filterProvider.findPropertyFilter("TransactionFilter", null)
                .includeAll());
    }

    @Test
    public void testExcludeNonExistentField() {
        Set<String> includeFields = Set.of();
        Set<String> excludeFields = Set.of("nonExistentField");

        FilterProvider filters = DynamicFilterUtil.createFilters(
                includeFields, excludeFields, AccountDetails.class);

        SimpleFilterProvider filterProvider = (SimpleFilterProvider) filters;
        
        // Should not affect filtering since field doesn’t exist
        assertTrue(filterProvider.findPropertyFilter("AccountDetailsFilter", null)
                .includeAll());
    }
}
