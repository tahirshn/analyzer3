# Kotlin ile Spring Boot/Hibernate Projesinden Sütun Metadata'sı Alma

Aynı işlevselliği Kotlin ile nasıl elde edebileceğinizi aşağıda bulabilirsiniz:

## 1. Hibernate Metadata API Kullanımı (Kotlin)

```kotlin
import org.hibernate.boot.Metadata
import org.hibernate.boot.MetadataSources
import org.hibernate.boot.registry.StandardServiceRegistry
import org.hibernate.boot.registry.StandardServiceRegistryBuilder
import org.hibernate.mapping.Column
import org.hibernate.mapping.PersistentClass
import org.hibernate.mapping.Property
import org.hibernate.mapping.Table

class HibernateMetadataExtractor {

    fun extractMetadata() {
        val registry: StandardServiceRegistry = StandardServiceRegistryBuilder()
            .configure() // hibernate.cfg.xml yükler
            .build()
        
        try {
            val metadata: Metadata = MetadataSources(registry)
                .metadataBuilder
                .build()
            
            // Tüm entity eşlemelerini al
            for (persistentClass in metadata.entityBindings) {
                val table: Table = persistentClass.table
                println("Tablo: ${table.name}")
                
                // Sütunları al
                for (column in table.columns) {
                    println("  Sütun: ${column.name}")
                    println("    Tip: ${column.sqlType}")
                    println("    Uzunluk: ${column.length}")
                    println("    Nullable: ${column.isNullable}")
                    // Daha fazla özellik...
                }
                
                // Özellikleri al (alanlar)
                for (property in persistentClass.propertyIterator) {
                    println("Özellik: ${property.name}")
                    // Daha fazla metadata...
                }
            }
        } finally {
            StandardServiceRegistryBuilder.destroy(registry)
        }
    }
}
```

## 2. JPA Metamodel API Kullanımı (Kotlin)

```kotlin
import javax.persistence.EntityManager
import javax.persistence.metamodel.EntityType
import javax.persistence.metamodel.Metamodel

class JpaMetadataExtractor(private val entityManager: EntityManager) {

    fun extractMetadata() {
        val metamodel: Metamodel = entityManager.metamodel
        
        for (entityType in metamodel.entities) {
            println("Entity: ${entityType.name}")
            
            entityType.attributes.forEach { attribute ->
                println("  Özellik: ${attribute.name}")
                println("    Tip: ${attribute.javaType.simpleName}")
                println("    Kalıcı özellik tipi: ${attribute.persistentAttributeType}")
                
                // Sütun-spesifik annotation'lar için reflection gerekir
            }
        }
    }
}
```

## 3. Spring Data JPA ile Reflection Kullanımı (Kotlin)

```kotlin
import org.springframework.stereotype.Component
import javax.persistence.Column
import javax.persistence.EntityManager
import kotlin.reflect.full.memberProperties

@Component
class EntityMetadataExtractor(private val entityManager: EntityManager) {

    fun extractMetadata(entityClass: Class<*>) {
        // JPA entity adını al
        val entityName = entityClass.simpleName
        println("Entity: $entityName")
        
        // Alanları ve annotation'ları al
        entityClass.kotlin.memberProperties.forEach { property ->
            property.annotations.forEach { annotation ->
                if (annotation is Column) {
                    println("  Alan: ${property.name}")
                    println("    Sütun adı: ${annotation.name}")
                    println("    Nullable: ${annotation.nullable}")
                    println("    Uzunluk: ${annotation.length}")
                    // Daha fazla sütun özelliği...
                }
            }
        }
    }
}
```

## 4. Doğrudan Database Metadata Sorgulama (Kotlin)

```kotlin
import java.sql.Connection
import java.sql.DatabaseMetaData
import javax.sql.DataSource

class DatabaseMetadataExtractor(private val dataSource: DataSource) {

    fun extractMetadata() {
        dataSource.connection.use { connection ->
            val metaData: DatabaseMetaData = connection.metaData
            
            // Tüm tabloları al
            metaData.getTables(null, null, "%", arrayOf("TABLE")).use { tables ->
                while (tables.next()) {
                    val tableName = tables.getString("TABLE_NAME")
                    println("Tablo: $tableName")
                    
                    // Bu tablonun sütunlarını al
                    metaData.getColumns(null, null, tableName, "%").use { columns ->
                        while (columns.next()) {
                            println("  Sütun: ${columns.getString("COLUMN_NAME")}")
                            println("    Tip: ${columns.getString("TYPE_NAME")}")
                            println("    Boyut: ${columns.getInt("COLUMN_SIZE")}")
                            println("    Nullable: ${columns.getInt("NULLABLE")}")
                            // Daha fazla sütun özelliği...
                        }
                    }
                }
            }
        }
    }
}
```

## Kotlin-Spesifik İyileştirmeler

Kotlin'de kodu daha idiomatik hale getirebiliriz:

```kotlin
// Extension fonksiyon ile daha temiz kullanım
fun EntityManager.extractMetadata() {
    metamodel.entities.forEach { entityType ->
        println("Entity: ${entityType.name}")
        entityType.attributes.forEach { attribute ->
            println("  ${attribute.name}: ${attribute.javaType.simpleName}")
        }
    }
}

// Kullanımı:
// entityManager.extractMetadata()
```

Kotlin'in null-safety özelliklerini kullanarak:

```kotlin
fun safeExtractColumnInfo(column: Column?) {
    column?.let {
        println("Sütun: ${it.name}")
        println("Tip: ${it.sqlType}")
    } ?: println("Sütun bilgisi bulunamadı")
}
```

Bu Kotlin implementasyonları, Java versiyonlarıyla aynı işlevselliği sağlarken Kotlin'in daha kısa ve okunabilir sözdizimini kullanır. Özellikle null safety, extension fonksiyonlar ve koleksiyon işlemleri Kotlin'de daha temiz bir kod sağlar.
