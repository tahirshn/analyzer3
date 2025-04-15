data class EntityMetadata(
    val className: String,
    val tableName: String,
    val fields: List<FieldMetadata>
)

data class FieldMetadata(
    val fieldName: String,
    val columnName: String,
    val isId: Boolean
)
object NamingUtils {
    fun toKebabCase(input: String): String {
        return input.replace(Regex("([a-z])([A-Z])"), "$1-$2").lowercase()
    }
}

import jakarta.persistence.*
import org.springframework.stereotype.Component
import java.lang.reflect.Field
import javax.persistence.metamodel.EntityType

@Component
class EntityMetadataExtractor {

    fun extract(entityTypes: Set<EntityType<*>>): List<EntityMetadata> {
        return entityTypes.map { entityType ->
            val clazz = entityType.javaType
            val tableName = resolveTableName(clazz)
            val fields = clazz.declaredFields.map { resolveFieldMetadata(it) }

            EntityMetadata(
                className = clazz.simpleName,
                tableName = tableName,
                fields = fields
            )
        }
    }

    private fun resolveTableName(clazz: Class<*>): String {
        return clazz.getAnnotation(Table::class.java)?.name
            ?.takeIf { it.isNotBlank() }
            ?: NamingUtils.toKebabCase(clazz.simpleName)
    }

    private fun resolveFieldMetadata(field: Field): FieldMetadata {
        val columnAnnotation = field.getAnnotation(Column::class.java)
        val joinColumnAnnotation = field.getAnnotation(JoinColumn::class.java)

        val columnName = when {
            columnAnnotation?.name?.isNotBlank() == true -> columnAnnotation.name
            joinColumnAnnotation?.name?.isNotBlank() == true -> joinColumnAnnotation.name
            else -> NamingUtils.toKebabCase(field.name)
        }

        val isId = field.isAnnotationPresent(Id::class.java)

        return FieldMetadata(
            fieldName = field.name,
            columnName = columnName,
            isId = isId
        )
    }
}

import jakarta.persistence.EntityManager
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.context.SpringBootTest

@SpringBootTest
class EntityMetadataExtractorTest {

    @Autowired
    lateinit var entityManager: EntityManager

    @Autowired
    lateinit var extractor: EntityMetadataExtractor

    @Test
    fun `should extract metadata for all JPA entities`() {
        val metadataList = extractor.extract(entityManager.metamodel.entities)

        metadataList.forEach { entity ->
            println("Entity: ${entity.className}")
            println("  Table: ${entity.tableName}")
            entity.fields.forEach { field ->
                println("    Field: ${field.fieldName}")
                println("    Column: ${field.columnName}")
                if (field.isId) println("    >> This is ID field")
            }
            println("--------------------------------------------------")
        }
    }
}

