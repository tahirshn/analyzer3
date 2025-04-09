# Obtaining Column Metadata from a Spring Boot/Hibernate Project

To extract column metadata (like column names, types, constraints, etc.) from a Spring Boot application using Hibernate, you have several options:

## 1. Using Hibernate's Metadata API

```java
import org.hibernate.boot.Metadata;
import org.hibernate.boot.MetadataSources;
import org.hibernate.boot.registry.StandardServiceRegistry;
import org.hibernate.boot.registry.StandardServiceRegistryBuilder;
import org.hibernate.mapping.Column;
import org.hibernate.mapping.PersistentClass;
import org.hibernate.mapping.Property;
import org.hibernate.mapping.Table;

public class HibernateMetadataExtractor {

    public void extractMetadata() {
        StandardServiceRegistry registry = new StandardServiceRegistryBuilder()
            .configure() // loads hibernate.cfg.xml
            .build();
        
        try {
            Metadata metadata = new MetadataSources(registry)
                .getMetadataBuilder()
                .build();
            
            // Get all entity mappings
            for (PersistentClass persistentClass : metadata.getEntityBindings()) {
                Table table = persistentClass.getTable();
                System.out.println("Table: " + table.getName());
                
                // Get columns
                for (Column column : table.getColumns()) {
                    System.out.println("  Column: " + column.getName());
                    System.out.println("    Type: " + column.getSqlType());
                    System.out.println("    Length: " + column.getLength());
                    System.out.println("    Nullable: " + column.isNullable());
                    // More properties available...
                }
                
                // Get properties (fields)
                for (Property property : persistentClass.getProperties()) {
                    System.out.println("Property: " + property.getName());
                    // Access more property metadata...
                }
            }
        } finally {
            StandardServiceRegistryBuilder.destroy(registry);
        }
    }
}
```

## 2. Using JPA's Metamodel API

```java
import javax.persistence.EntityManager;
import javax.persistence.metamodel.EntityType;
import javax.persistence.metamodel.Metamodel;

public class JpaMetadataExtractor {

    private final EntityManager entityManager;

    public JpaMetadataExtractor(EntityManager entityManager) {
        this.entityManager = entityManager;
    }

    public void extractMetadata() {
        Metamodel metamodel = entityManager.getMetamodel();
        
        for (EntityType<?> entityType : metamodel.getEntities()) {
            System.out.println("Entity: " + entityType.getName());
            
            entityType.getAttributes().forEach(attribute -> {
                System.out.println("  Attribute: " + attribute.getName());
                System.out.println("    Type: " + attribute.getJavaType().getSimpleName());
                System.out.println("    Persistent type: " + attribute.getPersistentAttributeType());
                
                // For column-specific annotations you'd need reflection
            });
        }
    }
}
```

## 3. Using Spring Data JPA with Reflection

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import javax.persistence.Column;
import javax.persistence.EntityManager;
import java.lang.reflect.Field;

@Component
public class EntityMetadataExtractor {

    @Autowired
    private EntityManager entityManager;

    public void extractMetadata(Class<?> entityClass) {
        // Get JPA entity name
        String entityName = entityClass.getSimpleName();
        System.out.println("Entity: " + entityName);
        
        // Get fields and annotations
        for (Field field : entityClass.getDeclaredFields()) {
            Column columnAnnotation = field.getAnnotation(Column.class);
            if (columnAnnotation != null) {
                System.out.println("  Field: " + field.getName());
                System.out.println("    Column name: " + columnAnnotation.name());
                System.out.println("    Nullable: " + columnAnnotation.nullable());
                System.out.println("    Length: " + columnAnnotation.length());
                // More column properties...
            }
        }
    }
}
```

## 4. Querying Database Metadata Directly

```java
import java.sql.Connection;
import java.sql.DatabaseMetaData;
import java.sql.ResultSet;
import java.sql.SQLException;
import javax.sql.DataSource;

public class DatabaseMetadataExtractor {

    private final DataSource dataSource;

    public DatabaseMetadataExtractor(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    public void extractMetadata() throws SQLException {
        try (Connection connection = dataSource.getConnection()) {
            DatabaseMetaData metaData = connection.getMetaData();
            
            // Get all tables
            ResultSet tables = metaData.getTables(null, null, "%", new String[]{"TABLE"});
            while (tables.next()) {
                String tableName = tables.getString("TABLE_NAME");
                System.out.println("Table: " + tableName);
                
                // Get columns for this table
                ResultSet columns = metaData.getColumns(null, null, tableName, "%");
                while (columns.next()) {
                    System.out.println("  Column: " + columns.getString("COLUMN_NAME"));
                    System.out.println("    Type: " + columns.getString("TYPE_NAME"));
                    System.out.println("    Size: " + columns.getInt("COLUMN_SIZE"));
                    System.out.println("    Nullable: " + columns.getInt("NULLABLE"));
                    // More column properties...
                }
                columns.close();
            }
            tables.close();
        }
    }
}
```

## Best Approach for Real Implementation

For a production application, I recommend:

1. **For entity/column mapping info**: Use Hibernate's Metadata API (option 1) as it provides the most complete picture of how your entities map to database structures.

2. **For runtime metadata**: Use JPA's Metamodel API (option 2) as it's standardized and works with any JPA provider.

3. **For actual database schema**: Use DatabaseMetaData (option 4) if you need to verify what's actually in the database.

Remember that in Spring Boot, you can autowire the `EntityManager` or `SessionFactory` to access these APIs.
