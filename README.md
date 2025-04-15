package com.sqlinspector.core.analyzer

import jakarta.persistence.EntityManager
import org.springframework.data.jpa.repository.Query
import org.springframework.data.repository.CrudRepository
import java.util.regex.Pattern
import kotlin.reflect.KClass

class QueryFieldAnalyzer(private val entityManager: EntityManager) {

    data class QueryAnalysisResult(
        val methodName: String,
        val rawQuery: String?,
        val extractedFields: Set<String>,
        val tableName: String,
        val issues: List<String>
    )

    fun analyzeRepository(repo: CrudRepository<*, *>): List<QueryAnalysisResult> {
        val repoClass = repo.javaClass
        val results = mutableListOf<QueryAnalysisResult>()

        val entityClass = resolveEntityClass(repoClass) ?: return emptyList()
        val tableName = resolveTableName(entityClass)
        val entityFields = extractEntityFieldNames(entityClass)

        for (method in repoClass.methods) {
            val queryAnnotation = method.getAnnotation(Query::class.java)
            val rawQuery = queryAnnotation?.value ?: continue

            val isNative = queryAnnotation.nativeQuery
            val resolvedQuery = if (isNative) rawQuery else replaceAliasesWithTableNames(rawQuery, entityClass.simpleName)

            val fieldsInQuery = extractFieldsFromQuery(resolvedQuery)

            val unknownFields = fieldsInQuery.filterNot { it in entityFields }.toList()
            val issues = unknownFields.map { "Unknown field in query: $it" }

            results.add(
                QueryAnalysisResult(
                    methodName = method.name,
                    rawQuery = rawQuery,
                    extractedFields = fieldsInQuery,
                    tableName = tableName,
                    issues = issues
                )
            )
        }

        return results
    }

    private fun extractEntityFieldNames(entityClass: Class<*>): Set<String> {
        return entityClass.declaredFields.map { it.name }.toSet()
    }

    private fun resolveEntityClass(repoClass: Class<*>): Class<*>? {
        return repoClass.genericInterfaces
            .flatMap { generic ->
                (generic as? java.lang.reflect.ParameterizedType)?.actualTypeArguments?.toList().orEmpty()
            }
            .firstOrNull() as? Class<*>
    }

    private fun resolveTableName(entityClass: Class<*>): String {
        val table = entityClass.getAnnotation(jakarta.persistence.Table::class.java)
        return table?.name ?: entityClass.simpleName.toSnakeCase()
    }

    private fun extractFieldsFromQuery(query: String): Set<String> {
        val fieldPattern = Pattern.compile("""\b(\w+)\.(\w+)\b""")
        val matcher = fieldPattern.matcher(query)
        val fields = mutableSetOf<String>()

        while (matcher.find()) {
            fields.add(matcher.group(2))
        }

        return fields
    }

    private fun replaceAliasesWithTableNames(query: String, entityName: String): String {
        val aliasMap = mutableMapOf<String, String>()
        val aliasPattern = Regex("""\b(from|join)\s+(\w+)\s+(\w+)""", RegexOption.IGNORE_CASE)
        val replacedQuery = aliasPattern.replace(query) { match ->
            val tableOrEntity = match.groupValues[2]
            val alias = match.groupValues[3]
            aliasMap[alias] = tableOrEntity
            match.value
        }

        var resultQuery = replacedQuery
        aliasMap.forEach { (alias, actual) ->
            val pattern = Regex("""\b$alias\.\b""")
            resultQuery = resultQuery.replace(pattern, "$actual.")
        }

        // Eğer hiç alias yoksa, entity adını doğrudan kullan
        if (aliasMap.isEmpty()) {
            val defaultPattern = Regex("""\b\w+\.\b""")
            resultQuery = defaultPattern.replace(resultQuery, "$entityName.")
        }

        return resultQuery
    }

    private fun String.toSnakeCase(): String =
        this.replace(Regex("([a-z])([A-Z]+)"), "$1_$2").lowercase()
}
