package com.github.sqlinspector

import jakarta.persistence.*
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.context.ApplicationContext
import org.springframework.data.jpa.repository.Query
import org.springframework.data.repository.CrudRepository
import org.springframework.data.repository.query.parser.PartTree
import java.lang.reflect.ParameterizedType
import java.nio.file.Files
import java.nio.file.Paths

class EntityIndexAnalyzer : SingleServiceTestsBase() {

    @Autowired
    private lateinit var applicationContext: ApplicationContext

    @Autowired
    private lateinit var entityManager: EntityManager

    data class IndexMetadata(
        val indexName: String?,
        val columnName: String?,
        val isUnique: Boolean,
        val isPrimary: Boolean,
        val tableName: String
    )

    data class EntityField(
        val entityClassName: String,
        val tableName: String,
        val fieldName: String,
        val columnName: String,
        val isId: Boolean,
        val isIgnored: Boolean
    )

    @Test
    fun `should analyze repositories and detect missing indexes`() {
        val report = StringBuilder()
        val entityFields = extractEntityFields()
        val indexes = fetchTableIndexes(entityFields.map { it.tableName }.distinct())

        // Analyze each repository bean
        applicationContext.getBeansOfType(CrudRepository::class.java).forEach { (beanName, repository) ->
            val repoInterface = repository.javaClass.interfaces.firstOrNull { CrudRepository::class.java.isAssignableFrom(it) } ?: return@forEach
            val entityClass = (repoInterface.genericInterfaces.firstOrNull { it is ParameterizedType } as? ParameterizedType)
                ?.actualTypeArguments?.getOrNull(0) as? Class<*> ?: return@forEach

            val tableName = resolveTableName(entityClass)
            val fieldMap = entityFields.filter { it.tableName == tableName }.associateBy { it.fieldName }

            report.appendLine("\nREPOSITORY: $beanName -> ENTITY: ${entityClass.simpleName}")

            repoInterface.declaredMethods.forEach { method ->
                report.appendLine("  METHOD: ${method.name}")

                val rawQuery = method.getAnnotation(Query::class.java)?.value.orEmpty()
                if (rawQuery.isNotBlank()) {
                    report.appendLine("    @Query: $rawQuery")
                    analyzeRawQuery(rawQuery, entityFields).forEach { field ->
                        report.appendLine(buildIndexReport(field, indexes))
                    }
                } else {
                    val partTree = PartTree(method.name, entityClass)
                    partTree.parts.flatMap { it.property.stream() }
                        .map { it.segment.replaceFirstChar(Char::lowercase) }
                        .forEach { name ->
                            val field = fieldMap[name]
                            report.appendLine(field?.let { buildIndexReport(it, indexes) } ?: "    [UNKNOWN FIELD] $name")
                        }
                }
            }
        }

        writeReport(report.toString())
    }

    private fun extractEntityFields(): List<EntityField> =
        entityManager.metamodel.entities.flatMap { entityType ->
            val clazz = entityType.javaType
            val tableName = clazz.getAnnotation(Table::class.java)?.name ?: clazz.simpleName.toSnakeCase()

            clazz.declaredFields.mapNotNull { field ->
                val isId = field.isAnnotationPresent(Id::class.java) || field.isAnnotationPresent(EmbeddedId::class.java)
                val columnName = field.getAnnotation(Column::class.java)?.name
                    ?: field.getAnnotation(JoinColumn::class.java)?.name
                    ?: field.name.toSnakeCase()
                val isIgnored = !isId && field.getAnnotation(Column::class.java) == null && field.getAnnotation(JoinColumn::class.java) == null

                EntityField(clazz.simpleName, tableName, field.name, columnName, isId, isIgnored)
            }
        }

    private fun fetchTableIndexes(tableNames: List<String>): Map<String, List<IndexMetadata>> {
        val indexMap = mutableMapOf<String, MutableList<IndexMetadata>>()

        val query = """
            SELECT t.relname AS table_name, i.relname AS index_name, a.attname AS column_name,
                   ix.indisunique AS is_unique, ix.indisprimary AS is_primary
            FROM pg_class t, pg_class i, pg_index ix, pg_attribute a
            WHERE t.oid = ix.indrelid AND i.oid = ix.indexrelid AND a.attrelid = t.oid
              AND a.attnum = ANY(ix.indkey) AND t.relkind = 'r'
              AND t.relname IN (${tableNames.joinToString(",") { "'$it'" }})
            ORDER BY t.relname, i.relname
        """.trimIndent()

        val result = entityManager.createNativeQuery(query).resultList
        result.forEach { row ->
            val cols = row as Array<*>
            indexMap.getOrPut(cols[0] as String) {
                mutableListOf()
            }.add(
                IndexMetadata(
                    tableName = cols[0] as String,
                    indexName = cols[1] as String,
                    columnName = cols[2] as String,
                    isUnique = cols[3] as Boolean,
                    isPrimary = cols[4] as Boolean
                )
            )
        }

        return indexMap
    }

    private fun analyzeRawQuery(query: String, entities: List<EntityField>): List<EntityField> {
        val clean = query.replace("\n", " ").replace(Regex("\\s+"), " ").trim()
        val aliasToEntity = extractAliasToEntityMap(clean)
        val where = extractWhereClause(clean)
        val aliasFields = extractAliasFields(where)

        return aliasFields.mapNotNull { (alias, field) ->
            val entity = aliasToEntity[alias.lowercase()]
            entities.find { it.entityClassName == entity && it.fieldName == field }
        }
    }

    private fun extractAliasToEntityMap(sql: String): Map<String, String> {
        val regex = Regex("""(?:(from|join|left join|inner join|left join fetch)\s+)([\w.]+)\s+(\w+)""", RegexOption.IGNORE_CASE)
        return regex.findAll(sql).associate { match ->
            val entityPath = match.groupValues[2]
            val alias = match.groupValues[3]
            alias.lowercase() to entityPath.substringAfterLast(".")
        }
    }

    private fun extractWhereClause(sql: String): String =
        Regex("where (.+)", RegexOption.IGNORE_CASE).find(sql)?.groupValues?.get(1) ?: ""

    private fun extractAliasFields(where: String): List<Pair<String, String>> =
        Regex("""(\w+)\.(\w+)""").findAll(where).map { it.groupValues[1] to it.groupValues[2] }.toList()

    private fun resolveTableName(clazz: Class<*>): String =
        clazz.getAnnotation(Table::class.java)?.name ?: clazz.simpleName.toSnakeCase()

    private fun buildIndexReport(field: EntityField, indexMap: Map<String, List<IndexMetadata>>): String {
        return if (field.isIgnored) {
            "    IGNORED - [MISSING INDEX] on field '${field.fieldName}' -> column '${field.columnName}'"
        } else if (indexMap[field.tableName]?.any { it.columnName.equals(field.columnName, true) } == true) {
            "    [INDEXED] on field '${field.fieldName}' -> column '${field.columnName}'"
        } else {
            "    [MISSING INDEX] on field '${field.fieldName}' -> column '${field.columnName}'"
        }
    }

    private fun writeReport(content: String) {
        val outputDir = Paths.get("build/sql_analysis_logs")
        Files.createDirectories(outputDir)
        val outputFile = outputDir.resolve("result_set.txt").toFile()
        outputFile.writeText(content)
        println("Indexes written to: ${outputFile.absolutePath}")
    }

    private fun String.toSnakeCase(): String =
        replace(Regex("([a-z])([A-Z]+)"), "$1_$2").lowercase()
}
