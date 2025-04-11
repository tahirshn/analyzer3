import org.junit.jupiter.api.Test
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.beans.factory.annotation.Autowired
import jakarta.persistence.EntityManager
import jakarta.persistence.metamodel.EntityType
import java.io.File
import java.nio.file.Files
import java.nio.file.Paths
import com.fasterxml.jackson.databind.ObjectMapper
import com.fasterxml.jackson.module.kotlin.jacksonObjectMapper
import com.fasterxml.jackson.module.kotlin.writeValueAsString

@SpringBootTest
class IndexLoggerJsonTest {

    @Autowired
    lateinit var entityManager: EntityManager

    @Test
    fun `log indexes as JSON to file`() {
        val connection = entityManager.unwrap(java.sql.Connection::class.java)
        val statement = connection.createStatement()

        // Sadece projedeki entity'lere ait tablo adlarını al
        val entityTableNames = entityManager.metamodel.entities
            .mapNotNull { it.getTableName() }
            .toSet()

        val query = """
            SELECT 
                t.relname AS table_name,
                i.relname AS index_name,
                a.attname AS column_name,
                ix.indisunique AS is_unique,
                ix.indisprimary AS is_primary
            FROM 
                pg_class t,
                pg_class i,
                pg_index ix,
                pg_attribute a
            WHERE 
                t.oid = ix.indrelid
                AND i.oid = ix.indexrelid
                AND a.attrelid = t.oid
                AND a.attnum = ANY(ix.indkey)
                AND t.relkind = 'r'
            ORDER BY
                t.relname, i.relname
        """.trimIndent()

        val resultSet = statement.executeQuery(query)

        val resultList = mutableListOf<Map<String, Any>>()

        while (resultSet.next()) {
            val table = resultSet.getString("table_name")
            if (table !in entityTableNames) continue

            val row = mapOf(
                "table" to table,
                "index" to resultSet.getString("index_name"),
                "column" to resultSet.getString("column_name"),
                "isUnique" to resultSet.getBoolean("is_unique"),
                "isPrimary" to resultSet.getBoolean("is_primary")
            )
            resultList.add(row)
        }

        val mapper: ObjectMapper = jacksonObjectMapper()
        val json = mapper.writerWithDefaultPrettyPrinter().writeValueAsString(resultList)

        val outputDir = Paths.get("build/sql_logs")
        Files.createDirectories(outputDir)

        val outputFile = outputDir.resolve("indexes.json").toFile()
        outputFile.writeText(json)

        println("Indexes written to: ${outputFile.absolutePath}")
    }

    // Extension ile tablo adını al (annotation'dan)
    private fun EntityType<*>.getTableName(): String? {
        return this.javaType.getAnnotation(jakarta.persistence.Table::class.java)?.name
            ?: this.name.lowercase() // Eğer @Table yoksa entity adı
    }
}
