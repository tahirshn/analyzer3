import jakarta.persistence.*
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.context.ApplicationContext
import kotlin.reflect.jvm.kotlinProperty

@SpringBootTest
class EntityMetadataExtractorSpringContextTest {

    @Autowired
    lateinit var applicationContext: ApplicationContext

    @Autowired
    lateinit var entityManager: EntityManager

    @Test
    fun extractEntitiesFromMetamodel() {
        val metamodel = entityManager.metamodel

        for (entityType in metamodel.entities) {
            val javaType = entityType.javaType

            println("Entity Class: ${javaType.simpleName}")

            // Table name
            val tableAnnotation = javaType.getAnnotation(Table::class.java)
            val tableName = tableAnnotation?.name?.takeIf { it.isNotBlank() }
                ?: toKebabCase(javaType.simpleName)
            println("Table Name: $tableName")

            for (field in javaType.declaredFields) {
                field.isAccessible = true

                val columnAnnotation = field.getAnnotation(Column::class.java)
                val joinColumnAnnotation = field.getAnnotation(JoinColumn::class.java)

                val columnName = when {
                    columnAnnotation?.name?.isNotBlank() == true -> columnAnnotation.name
                    joinColumnAnnotation?.name?.isNotBlank() == true -> joinColumnAnnotation.name
                    else -> toKebabCase(field.name)
                }

                val isId = field.isAnnotationPresent(Id::class.java)

                println("  Field: ${field.name}")
                println("    Column: $columnName")
                if (isId) println("    >> This is ID field")
            }

            println("--------------------------------------------------")
        }
    }

    private fun toKebabCase(input: String): String {
        return input.replace(Regex("([a-z])([A-Z])"), "$1-$2")
            .lowercase()
    }
}
