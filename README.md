import jakarta.persistence.*
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.context.ApplicationContext
import org.springframework.data.jpa.repository.Query
import org.springframework.data.repository.CrudRepository
import org.springframework.util.ReflectionUtils
import java.lang.reflect.ParameterizedType
import java.sql.Connection
import java.sql.DatabaseMetaData
import java.sql.ResultSet
import kotlin.reflect.full.findAnnotation
import kotlin.reflect.full.memberProperties
import kotlin.reflect.jvm.kotlinProperty

@SpringBootTest
class RepositoryIndexAuditTest {

    @Autowired
    private lateinit var applicationContext: ApplicationContext

    @PersistenceContext
    private lateinit var entityManager: EntityManager

    data class IndexMetadata(
        val indexName: String?,
        val columnName: String?,
        val isUnique: Boolean,
        val isPrimary: Boolean
    )

    data class EntityField(
        val entityClassName: String,
        val tableName: String,
        val fieldName: String,
        val columnName: String,
        val isId: Boolean
    )

    @Test
    fun `should detect repository methods and missing indexes`() {
        val connection: Connection = entityManager.unwrap(Connection::class.java)
        val metaData: DatabaseMetaData = connection.metaData

        val tableIndexes = mutableMapOf<String, List<IndexMetadata>>()
        val entityFields = extractEntityFields()

        // Load DB indexes
        entityFields.map { it.tableName }.distinct().forEach { tableName ->
            tableIndexes[tableName] = fetchIndexes(metaData, tableName)
        }

        // Print entity metadata
        entityFields.forEach {
            println("ENTITY: ${it.entityClassName}, TABLE: ${it.tableName}, FIELD: ${it.fieldName}, COLUMN: ${it.columnName}, IS_ID: ${it.isId}")
        }

        // Analyze repository methods
        val repoBeans = applicationContext.getBeansOfType(CrudRepository::class.java)
        repoBeans.forEach { (name, repo) ->
            val repoClass = repo.javaClass
            val interfaceType = repoClass.genericInterfaces.firstOrNull { it is ParameterizedType } as? ParameterizedType
            val entityClass = interfaceType?.actualTypeArguments?.get(0) as? Class<*> ?: return@forEach
            val entityTableName = resolveTableName(entityClass)
            val entityFieldMap = entityFields.filter { it.tableName == entityTableName }.associateBy { it.fieldName }

            println("\nREPOSITORY: $name -> ENTITY: ${entityClass.simpleName}")

            repoClass.methods.forEach { method ->
                val queryAnnotation = method.getAnnotation(Query::class.java)
                val queryText = queryAnnotation?.value ?: ""

                println("  METHOD: ${method.name}")
                if (queryText.isNotBlank()) {
                    println("    @Query: $queryText")
                    // Rough check of where conditions
                    val whereFields = Regex("\\b([a-zA-Z0-9_]+)\\s*=").findAll(queryText)
                        .map { it.groupValues[1].substringAfter('.') }
                        .toSet()

                    whereFields.forEach { field ->
                        val fieldMeta = entityFieldMap[field]
                        if (fieldMeta != null) {
                            val hasIndex = tableIndexes[entityTableName]?.any { it.columnName.equals(fieldMeta.columnName, true) } == true
                            if (!hasIndex) {
                                println("    [MISSING INDEX] on field '${fieldMeta.fieldName}' -> column '${fieldMeta.columnName}'")
                            }
                        } else {
                            println("    [UNKNOWN FIELD] $field")
                        }
                    }
                } else {
                    // Infer method name queries e.g., findByUserId
                    val inferredFields = Regex("findBy([A-Z][a-zA-Z0-9]*)").findAll(method.name)
                        .map { it.groupValues[1].replaceFirstChar { it.lowercase() } }
                        .toSet()

                    inferredFields.forEach { field ->
                        val fieldMeta = entityFieldMap[field]
                        if (fieldMeta != null) {
                            val hasIndex = tableIndexes[entityTableName]?.any { it.columnName.equals(fieldMeta.columnName, true) } == true
                            if (!hasIndex) {
                                println("    [MISSING INDEX] on inferred field '${fieldMeta.fieldName}' -> column '${fieldMeta.columnName}'")
                            }
                        } else {
                            println("    [UNKNOWN FIELD] $field")
                        }
                    }
                }
            }
        }
    }

    private fun extractEntityFields(): List<EntityField> {
        return entityManager.metamodel.entities.mapNotNull { entityType ->
            val clazz = entityType.javaType
            val table = clazz.getAnnotation(Table::class.java)
            val tableName = table?.name ?: clazz.simpleName.toSnakeCase()

            clazz.declaredFields.mapNotNull { field ->
                val isId = field.isAnnotationPresent(Id::class.java)
                val column = field.getAnnotation(Column::class.java)
                val joinColumn = field.getAnnotation(JoinColumn::class.java)
                val columnName = column?.name ?: joinColumn?.name ?: field.name.toSnakeCase()

                EntityField(
                    entityClassName = clazz.simpleName,
                    tableName = tableName,
                    fieldName = field.name,
                    columnName = columnName,
                    isId = isId
                )
            }
        }.flatten()
    }

    private fun fetchIndexes(metaData: DatabaseMetaData, tableName: String): List<IndexMetadata> {
        val indexes = mutableListOf<IndexMetadata>()
        val rs: ResultSet = metaData.getIndexInfo(null, null, tableName, false, false)
        while (rs.next()) {
            val indexName = rs.getString("INDEX_NAME")
            val columnName = rs.getString("COLUMN_NAME")
            val nonUnique = rs.getBoolean("NON_UNIQUE")
            val isPrimary = indexName?.contains("pk", true) == true || indexName?.contains("pkey", true) == true

            indexes.add(
                IndexMetadata(
                    indexName = indexName,
                    columnName = columnName,
                    isUnique = !nonUnique,
                    isPrimary = isPrimary
                )
            )
        }
        return indexes
    }

    private fun resolveTableName(entityClass: Class<*>): String {
        return entityClass.getAnnotation(Table::class.java)?.name ?: entityClass.simpleName.toSnakeCase()
    }

    private fun String.toSnakeCase(): String =
        this.replace(Regex("([a-z])([A-Z]+)"), "$1_$2").lowercase()
}
