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


import org.hibernate.SessionFactory
import org.hibernate.boot.Metadata
import org.hibernate.boot.MetadataSources
import org.hibernate.boot.registry.StandardServiceRegistry
import org.hibernate.boot.registry.StandardServiceRegistryBuilder
import org.springframework.stereotype.Component
import javax.persistence.EntityManager
import javax.persistence.metamodel.EntityType
import javax.sql.DataSource
import java.sql.DatabaseMetaData

@Component
class MetadataExtractor(
    private val entityManager: EntityManager,
    private val dataSource: DataSource
) {
    
    // 1. Hibernate Metadata API ile
    fun extractUsingHibernateMetadata(): Map<String, List<ColumnInfo>> {
        val sessionFactory = entityManager.entityManagerFactory.unwrap(SessionFactory::class.java)
        val registry: StandardServiceRegistry = StandardServiceRegistryBuilder()
            .applySettings(sessionFactory.properties)
            .build()
        
        return try {
            val metadata: Metadata = MetadataSources(registry)
                .metadataBuilder
                .build()
            
            metadata.entityBindings.associate { persistentClass ->
                val table = persistentClass.table
                table.name to table.columns.map { column ->
                    ColumnInfo(
                        name = column.name,
                        type = column.sqlType,
                        length = column.length,
                        nullable = column.isNullable,
                        unique = column.isUnique,
                        table = table.name
                    )
                }
            }
        } finally {
            StandardServiceRegistryBuilder.destroy(registry)
        }
    }
    
    // 2. JPA Metamodel API ile
    fun extractUsingJpaMetamodel(): Map<String, List<ColumnInfo>> {
        return entityManager.metamodel.entities.associate { entityType ->
            entityType.name to entityType.attributes.map { attribute ->
                ColumnInfo(
                    name = attribute.name,
                    type = attribute.javaType.simpleName,
                    table = entityType.name
                )
            }
        }
    }
    
    // 3. Database Metadata ile
    fun extractUsingDatabaseMetadata(): Map<String, List<ColumnInfo>> {
        dataSource.connection.use { connection ->
            val metaData: DatabaseMetaData = connection.metaData
            val result = mutableMapOf<String, MutableList<ColumnInfo>>()
            
            metaData.getTables(null, null, "%", arrayOf("TABLE")).use { tables ->
                while (tables.next()) {
                    val tableName = tables.getString("TABLE_NAME")
                    
                    metaData.getColumns(null, null, tableName, "%").use { columns ->
                        while (columns.next()) {
                            val columnInfo = ColumnInfo(
                                name = columns.getString("COLUMN_NAME"),
                                type = columns.getString("TYPE_NAME"),
                                length = columns.getInt("COLUMN_SIZE"),
                                nullable = columns.getInt("NULLABLE") == 1,
                                table = tableName
                            )
                            
                            result.getOrPut(tableName) { mutableListOf() }.add(columnInfo)
                        }
                    }
                }
            }
            return result
        }
    }
    
    data class ColumnInfo(
        val name: String,
        val type: String,
        val length: Int? = null,
        val nullable: Boolean? = null,
        val unique: Boolean? = null,
        val table: String
    )
}
-------------------------------------------------------


import org.hibernate.SessionFactory
import org.springframework.stereotype.Component
import javax.persistence.EntityManager
import javax.sql.DataSource
import java.sql.DatabaseMetaData

@Component
class TableIndexExtractor(
    private val entityManager: EntityManager,
    private val dataSource: DataSource
) {
    
    data class TableInfo(
        val name: String,
        val type: String,
        val remarks: String? = null
    )
    
    data class IndexInfo(
        val name: String,
        val columnName: String,
        val ordinalPosition: Int,
        val isUnique: Boolean,
        val isPrimaryKey: Boolean
    )
    
    // Tüm tabloları ve index bilgilerini getir
    fun extractAllTablesWithIndexes(): Map<TableInfo, List<IndexInfo>> {
        return dataSource.connection.use { connection ->
            val metaData = connection.metaData
            val result = mutableMapOf<TableInfo, List<IndexInfo>>()
            
            // Tüm tabloları al
            metaData.getTables(null, null, "%", arrayOf("TABLE")).use { tables ->
                while (tables.next()) {
                    val tableInfo = TableInfo(
                        name = tables.getString("TABLE_NAME"),
                        type = tables.getString("TABLE_TYPE"),
                        remarks = tables.getString("REMARKS")
                    )
                    
                    // Tablonun index bilgilerini al
                    val indexes = getIndexesForTable(metaData, tableInfo.name)
                    result[tableInfo] = indexes
                }
            }
            result
        }
    }
    
    // Belirli bir tablonun index bilgilerini getir
    fun getIndexesForTable(tableName: String): List<IndexInfo> {
        return dataSource.connection.use { connection ->
            getIndexesForTable(connection.metaData, tableName)
        }
    }
    
    private fun getIndexesForTable(metaData: DatabaseMetaData, tableName: String): List<IndexInfo> {
        val indexes = mutableListOf<IndexInfo>()
        
        // Primary Key bilgilerini al
        metaData.getPrimaryKeys(null, null, tableName).use { primaryKeys ->
            while (primaryKeys.next()) {
                indexes.add(IndexInfo(
                    name = "PRIMARY_KEY",
                    columnName = primaryKeys.getString("COLUMN_NAME"),
                    ordinalPosition = primaryKeys.getInt("KEY_SEQ"),
                    isUnique = true,
                    isPrimaryKey = true
                ))
            }
        }
        
        // Diğer index bilgilerini al (non-PK)
        metaData.getIndexInfo(null, null, tableName, false, true).use { indexInfo ->
            while (indexInfo.next()) {
                // 0 = tableIndexStatistic, 1 = tableIndexClustered, 2 = tableIndexHashed, 3 = tableIndexOther
                if (indexInfo.getShort("TYPE") != DatabaseMetaData.tableIndexStatistic) {
                    indexes.add(IndexInfo(
                        name = indexInfo.getString("INDEX_NAME"),
                        columnName = indexInfo.getString("COLUMN_NAME"),
                        ordinalPosition = indexInfo.getInt("ORDINAL_POSITION"),
                        isUnique = !indexInfo.getBoolean("NON_UNIQUE"),
                        isPrimaryKey = false
                    ))
                }
            }
        }
        
        return indexes.sortedBy { it.name }
    }
    
    // Hibernate'den entity tablo eşleşmelerini al
    fun getHibernateTableMappings(): Map<String, String> {
        val sessionFactory = entityManager.entityManagerFactory.unwrap(SessionFactory::class.java)
        val metadata = sessionFactory.metamodel
        return metadata.entities.associate { entity ->
            entity.name to (entity.javaxEntity?.let { 
                it.getAnnotation(javax.persistence.Table::class.java)?.name ?: entity.name.toLowerCase()
            } ?: entity.name.toLowerCase())
        }
    }
}

import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.PathVariable
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RestController

@RestController
@RequestMapping("/api/tables")
class TableIndexController(private val tableIndexExtractor: TableIndexExtractor) {
    
    @GetMapping
    fun getAllTablesWithIndexes() = tableIndexExtractor.extractAllTablesWithIndexes()
    
    @GetMapping("/{tableName}/indexes")
    fun getIndexesForTable(@PathVariable tableName: String) = 
        tableIndexExtractor.getIndexesForTable(tableName)
    
    @GetMapping("/mappings")
    fun getHibernateMappings() = tableIndexExtractor.getHibernateTableMappings()
}

import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest
import javax.persistence.Entity
import javax.persistence.Id
import javax.persistence.Index
import javax.persistence.Table
import org.assertj.core.api.Assertions.assertThat

@DataJpaTest
class TableIndexExtractorTest {
    
    @Autowired
    private lateinit var tableIndexExtractor: TableIndexExtractor
    
    @Test
    fun `should extract all tables with indexes`() {
        val result = tableIndexExtractor.extractAllTablesWithIndexes()
        
        assertThat(result).isNotEmpty
        assertThat(result.keys).anyMatch { it.name == "test_entity" }
        
        val testEntityIndexes = result.entries.first { it.key.name == "test_entity" }.value
        assertThat(testEntityIndexes).anyMatch { it.isPrimaryKey && it.columnName == "id" }
    }
    
    @Test
    fun `should get indexes for specific table`() {
        val indexes = tableIndexExtractor.getIndexesForTable("test_entity")
        
        assertThat(indexes).anyMatch { 
            it.isPrimaryKey && it.columnName == "id" && it.name == "PRIMARY_KEY" 
        }
        
        assertThat(indexes).anyMatch { 
            !it.isPrimaryKey && it.columnName == "name" && it.name == "IDX_TEST_ENTITY_NAME" 
        }
    }
    
    @Test
    fun `should get hibernate table mappings`() {
        val mappings = tableIndexExtractor.getHibernateTableMappings()
        
        assertThat(mappings).containsEntry("TestEntity", "test_entity")
    }
}

@Entity
@Table(name = "test_entity", indexes = [
    Index(name = "IDX_TEST_ENTITY_NAME", columnList = "name")
])
class TestEntity(
    @Id
    val id: Long = 0,
    
    val name: String = ""
)

@Service
class DatabaseHealthService(
    private val tableIndexExtractor: TableIndexExtractor
) {
    fun checkMissingIndexes(): List<String> {
        val allTables = tableIndexExtractor.extractAllTablesWithIndexes()
        val missingIndexes = mutableListOf<String>()
        
        allTables.forEach { (table, indexes) ->
            if (indexes.none { !it.isPrimaryKey }) {
                missingIndexes.add("Table ${table.name} has no secondary indexes")
            }
        }
        
        return missingIndexes
    }
}

fun generateIndexCreationScripts(): String {
    val allTables = tableIndexExtractor.extractAllTablesWithIndexes()
    
    return buildString {
        appendLine("-- Index Creation Scripts")
        allTables.forEach { (table, indexes) ->
            indexes.filterNot { it.isPrimaryKey }.forEach { index ->
                appendLine(
                    "CREATE ${if (index.isUnique) "UNIQUE " else ""}INDEX ${index.name} " +
                    "ON ${table.name} (${index.columnName});"
                )
            }
        }
    }
}


###################################
import org.hibernate.SessionFactory
import org.hibernate.engine.spi.SessionFactoryImplementor
import org.hibernate.hql.internal.ast.QueryTranslatorImpl
import org.springframework.data.jpa.repository.Query
import org.springframework.data.repository.core.support.RepositoryFactorySupport
import org.springframework.stereotype.Component
import javax.persistence.EntityManager
import javax.persistence.PersistenceContext
import kotlin.reflect.KClass
import kotlin.reflect.full.declaredFunctions
import kotlin.reflect.full.findAnnotation
import kotlin.reflect.jvm.javaMethod

@Component
class JpaQueryExtractor(
    @PersistenceContext private val entityManager: EntityManager
) {
    private val sessionFactory: SessionFactoryImplementor by lazy {
        entityManager.entityManagerFactory.unwrap(SessionFactory::class.java) as SessionFactoryImplementor
    }

    data class RepositoryQueryInfo(
        val repositoryName: String,
        val methodName: String,
        val querySource: QuerySource,
        val rawQuery: String,
        val isNative: Boolean
    )

    enum class QuerySource {
        ANNOTATION, METHOD_NAME, UNKNOWN
    }

    /**
     * Tüm repository'lerdeki query'leri çıkarır
     */
    fun extractAllQueries(repositories: List<KClass<*>>): List<RepositoryQueryInfo> {
        return repositories.flatMap { extractQueriesFromRepository(it) }
    }

    /**
     * Tek bir repository'deki query'leri çıkarır
     */
    fun extractQueriesFromRepository(repositoryClass: KClass<*>): List<RepositoryQueryInfo> {
        return repositoryClass.declaredFunctions.mapNotNull { function ->
            val queryAnnotation = function.findAnnotation<Query>()
            val method = function.javaMethod ?: return@mapNotNull null

            when {
                queryAnnotation != null -> {
                    // @Query ile tanımlanmış query
                    val query = queryAnnotation.value
                    val isNative = queryAnnotation.nativeQuery
                    val translatedQuery = if (!isNative) translateHqlToSql(query) else query

                    RepositoryQueryInfo(
                        repositoryName = repositoryClass.simpleName!!,
                        methodName = function.name,
                        querySource = QuerySource.ANNOTATION,
                        rawQuery = translatedQuery,
                        isNative = isNative
                    )
                }
                isQueryMethod(method.name) -> {
                    // Method name inference ile oluşturulmuş query
                    val query = generateQueryFromMethodName(repositoryClass, function)
                    RepositoryQueryInfo(
                        repositoryName = repositoryClass.simpleName!!,
                        methodName = function.name,
                        querySource = QuerySource.METHOD_NAME,
                        rawQuery = query,
                        isNative = false
                    )
                }
                else -> null
            }
        }
    }

    private fun isQueryMethod(methodName: String): Boolean {
        val queryKeywords = listOf("find", "read", "get", "query", "search", "stream", "count", "exists", "delete", "remove")
        return queryKeywords.any { methodName.startsWith(it, ignoreCase = true) }
    }

    private fun generateQueryFromMethodName(repositoryClass: KClass<*>, function: KFunction<*>): String {
        // Basit bir implementasyon - gerçekte Spring'in query creation mekanizması daha karmaşıktır
        val entityClass = repositoryClass.supertypes
            .firstOrNull { it.toString().contains("JpaRepository") }
            ?.arguments?.get(0)?.type?.classifier as? KClass<*>
            ?: return "Could not determine entity class"

        val entityName = entityClass.simpleName!!
        val methodName = function.name

        return when {
            methodName.startsWith("findBy") -> {
                val property = methodName.removePrefix("findBy").decapitalize()
                "SELECT e FROM $entityName e WHERE e.$property = ?1"
            }
            methodName.startsWith("countBy") -> {
                val property = methodName.removePrefix("countBy").decapitalize()
                "SELECT COUNT(e) FROM $entityName e WHERE e.$property = ?1"
            }
            methodName.startsWith("deleteBy") -> {
                val property = methodName.removePrefix("deleteBy").decapitalize()
                "DELETE FROM $entityName e WHERE e.$property = ?1"
            }
            else -> "Generated query for $methodName on $entityName"
        }.let { translateHqlToSql(it) }
    }

    private fun translateHqlToSql(hql: String): String {
        return try {
            val translators = QueryTranslatorImpl(
                queryIdentifier = hql,
                query = hql,
                enabledFilters = emptyMap(),
                factory = sessionFactory,
                replacements = QueryTranslatorImpl.QUERY_TRANSLATOR
            )
            translators.compile(emptyMap(), false)
            translators.sqlString
        } catch (e: Exception) {
            "Error translating HQL to SQL: ${e.message}"
        }
    }
}

@Service
class QueryExtractionService(
    private val queryExtractor: JpaQueryExtractor,
    private val repositoryFactory: RepositoryFactorySupport
) {
    fun extractAndLogAllQueries() {
        val repositories = listOf(
            UserRepository::class,
            ProductRepository::class
        )

        val allQueries = queryExtractor.extractAllQueries(repositories)
        
        allQueries.forEach { queryInfo ->
            println("""
                Repository: ${queryInfo.repositoryName}
                Method: ${queryInfo.methodName}
                Source: ${queryInfo.querySource}
                Native: ${queryInfo.isNative}
                Query: ${queryInfo.rawQuery}
                ${"-".repeat(80)}
            """.trimIndent())
        }
    }
}

@Component
class RepositoryScanner(
    private val applicationContext: ApplicationContext
) {
    fun findAllRepositoryClasses(): List<KClass<*>> {
        return applicationContext.getBeansWithAnnotation(Repository::class.java)
            .values
            .map { it::class }
            .distinct()
    }
}

// Kullanım
val allRepositories = repositoryScanner.findAllRepositoryClasses()
val allQueries = queryExtractor.extractAllQueries(allRepositories)
