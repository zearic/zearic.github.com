---
title: A REST API than can dynamically manage and connect datasoures, including JDBC and Mongo
tags: "java" "spring boot" "restful" "jdbc" "mongo"
---



```java
package com.example.datasource;

import com.mongodb.client.MongoClient;
import com.mongodb.client.MongoIterable;
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.sql.Connection;
import java.sql.DatabaseMetaData;
import java.sql.ResultSet;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;

/**
 * Generic interface for managing data store clients/pools.
 * @param <R> Resource type (e.g. HikariDataSource, MongoClient)
 */
public interface DataStoreManager<R> {
    R getOrCreate(DataStoreConfig config);
    void removeAndClose(DataStoreConfig config);
}

/** Supported data store types. */
public enum DataStoreType {
    JDBC,
    MONGO
}

/**
 * Unified configuration for any data store.
 */
public class DataStoreConfig {
    private DataStoreType type;
    private String host;
    private int port;
    private String database;
    private String username;
    private String password;

    // JDBC-specific
    private Integer maxPoolSize;
    private Long leakDetectionThresholdMs;

    // Getters and setters omitted for brevity
    // ...
}

/**
 * JDBC-specific implementation of DataStoreManager.
 */
public class JdbcDataStoreManager implements DataStoreManager<HikariDataSource> {
    private final ConcurrentMap<String, HikariDataSource> pools = new ConcurrentHashMap<>();

    @Override
    public HikariDataSource getOrCreate(DataStoreConfig cfg) {
        String key = String.format("%s:%d/%s|%s", cfg.getHost(), cfg.getPort(), cfg.getDatabase(), cfg.getUsername());
        return pools.computeIfAbsent(key, k -> createPool(cfg));
    }

    @Override
    public void removeAndClose(DataStoreConfig cfg) {
        String key = String.format("%s:%d/%s|%s", cfg.getHost(), cfg.getPort(), cfg.getDatabase(), cfg.getUsername());
        HikariDataSource ds = pools.remove(key);
        if (ds != null) ds.close();
    }

    private HikariDataSource createPool(DataStoreConfig cfg) {
        HikariConfig hc = new HikariConfig();
        hc.setJdbcUrl(String.format("jdbc:oracle:thin:@%s:%d/%s", cfg.getHost(), cfg.getPort(), cfg.getDatabase()));
        hc.setUsername(cfg.getUsername());
        hc.setPassword(cfg.getPassword());
        if (cfg.getMaxPoolSize() != null) hc.setMaximumPoolSize(cfg.getMaxPoolSize());
        if (cfg.getLeakDetectionThresholdMs() != null) hc.setLeakDetectionThreshold(cfg.getLeakDetectionThresholdMs());
        return new HikariDataSource(hc);
    }
}

/**
 * MongoDB-specific implementation of DataStoreManager.
 */
public class MongoDataStoreManager implements DataStoreManager<MongoClient> {
    private final ConcurrentMap<String, MongoClient> clients = new ConcurrentHashMap<>();

    @Override
    public MongoClient getOrCreate(DataStoreConfig cfg) {
        String key = String.format("%s:%d/%s|%s", cfg.getHost(), cfg.getPort(), cfg.getDatabase(),
                cfg.getUsername() != null ? cfg.getUsername() : "");
        return clients.computeIfAbsent(key, k -> createClient(cfg));
    }

    @Override
    public void removeAndClose(DataStoreConfig cfg) {
        String key = String.format("%s:%d/%s|%s", cfg.getHost(), cfg.getPort(), cfg.getDatabase(),
                cfg.getUsername() != null ? cfg.getUsername() : "");
        MongoClient client = clients.remove(key);
        if (client != null) client.close();
    }

    private MongoClient createClient(DataStoreConfig cfg) {
        String uri;
        if (cfg.getUsername() != null && cfg.getPassword() != null) {
            uri = String.format("mongodb://%s:%s@%s:%d/%s", cfg.getUsername(), cfg.getPassword(), cfg.getHost(), cfg.getPort(), cfg.getDatabase());
        } else {
            uri = String.format("mongodb://%s:%d/%s", cfg.getHost(), cfg.getPort(), cfg.getDatabase());
        }
        return MongoClients.create(uri);
    }
}

/**
 * Factory for obtaining the correct DataStoreManager implementation.
 */
public class DataStoreManagerFactory {
    private static final JdbcDataStoreManager JDBC_MANAGER = new JdbcDataStoreManager();
    private static final MongoDataStoreManager MONGO_MANAGER = new MongoDataStoreManager();

    @SuppressWarnings("unchecked")
    public static <R> DataStoreManager<R> getManager(DataStoreConfig cfg) {
        switch (cfg.getType()) {
            case JDBC:  return (DataStoreManager<R>) JDBC_MANAGER;
            case MONGO: return (DataStoreManager<R>) MONGO_MANAGER;
            default:    throw new IllegalArgumentException("Unsupported type: " + cfg.getType());
        }
    }
}

/**
 * Unified controller fetching schema for both JDBC and MongoDB based on config.type.
 */
@RestController
@RequestMapping("/api/schema")
public class SchemaController {
    @PostMapping
    public ResponseEntity<List<String>> fetchSchema(@RequestBody DataStoreConfig cfg) {
        DataStoreType type = cfg.getType();
        DataStoreManager<?> mgr = DataStoreManagerFactory.getManager(cfg);
        List<String> names = new ArrayList<>();

        try {
            if (type == DataStoreType.JDBC) {
                HikariDataSource ds = (HikariDataSource) mgr.getOrCreate(cfg);
                try (Connection conn = ds.getConnection()) {
                    DatabaseMetaData meta = conn.getMetaData();
                    try (ResultSet rs = meta.getTables(null, cfg.getUsername().toUpperCase(), null, new String[]{"TABLE"})) {
                        while (rs.next()) names.add(rs.getString("TABLE_NAME"));
                    }
                }
            } else if (type == DataStoreType.MONGO) {
                MongoClient client = (MongoClient) mgr.getOrCreate(cfg);
                MongoIterable<String> cols = client.getDatabase(cfg.getDatabase()).listCollectionNames();
                for (String name : cols) names.add(name);
            } else {
                return ResponseEntity.badRequest().build();
            }
        } catch (Exception e) {
            return ResponseEntity.badRequest().build();
        }

        return ResponseEntity.ok(names);
    }
}
```