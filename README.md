package com.sqlinspector.analyzer

import com.sqlinspector.metadata.EntityMetadataExtractor
import com.sqlinspector.metadata.QueryField
import java.io.File

class QueryFieldAnalyzer(
    private val entityMetadataExtractor: EntityMetadataExtractor
) {

    fun analyzeQuery(sql: String): Set<QueryField> {
        val cleanedSql = sql.replace("\n", " ").replace(Regex("\s+"), " ").trim()
        val aliasToEntity = extractAliasToEntityMap(cleanedSql)
        val whereClause = extractWhereClause(cleanedSql)
        val aliasFieldPairs = extractAliasFields(whereClause)

        return aliasFieldPairs.mapNotNull { (alias, field) ->
            val entityName = aliasToEntity[alias.lowercase()]
            val entityMeta = entityMetadataExtractor.getEntityMetadata(entityName ?: return@mapNotNull null)
            val columnMeta = entityMeta?.fields?.find { it.fieldName.equals(field, ignoreCase = true) }
            if (columnMeta != null) QueryField(entityName, columnMeta.columnName) else null
        }.toSet()
    }

    private fun extractAliasToEntityMap(sql: String): Map<String, String> {
        val aliasMap = mutableMapOf<String, String>()
        val regex = Regex("""(?:(from|join|left join|inner join)\s+)([\w.]+)\s+(\w+)""", RegexOption.IGNORE_CASE)
        regex.findAll(sql).forEach {
            val path = it.groupValues[2] // gcr.cashbackCriteria -> cashbackCriteria
            val alias = it.groupValues[3] // alias
            val entity = path.substringAfterLast(".")
            aliasMap[alias.lowercase()] = entity
        }
        return aliasMap
    }

    private fun extractWhereClause(sql: String): String {
        val whereMatch = Regex("where (.+)", RegexOption.IGNORE_CASE).find(sql)
        return whereMatch?.groupValues?.get(1) ?: ""
    }

    private fun extractAliasFields(whereClause: String): List<Pair<String, String>> {
        val regex = Regex("""(\w+)\.(\w+)""")
        return regex.findAll(whereClause)
            .map { it.groupValues[1] to it.groupValues[2] }
            .toList()
    }
}

fun main() {
    val entityExtractor = EntityMetadataExtractor(File("src/main/resources/entities")) // or wherever metadata is
    val analyzer = QueryFieldAnalyzer(entityExtractor)

    val sql = """
        SELECT COUNT(gcr) > 0
        FROM GlobalCashbackRule gcr
        JOIN gcr.cashbackCriteria Cc
        JOIN cc.cashbackCriterionvalues CCV
        LEFT JOIN ccv.merchantCountryCriteria mcc
        WHERE gcr.activeSince = (SELECT MAX(activeSince) FROM GlobalCashbackRule WHERE activeSince <= :transactionTime AND status IN ('OVERRIDDEN', 'APPROVED'))
        AND cc.rule = 'EXCLUDE'
        AND (
            (cc.property = 'MERCHANT_ID' AND ccv.value = :merchantId)
            OR (cc.property = 'MERCHANT_NAME' AND ccv.value = :merchantName)
            OR (cc.property = 'MERCHANT_CATEGORY' AND ccv.value = :merchantCategory)
        )
        AND (
            mcc IS NULL
            OR :merchantCountryCode IS NULL
            OR :merchantCountryCode = mcc.countryCode
        )
    """.trimIndent()

    val fields = analyzer.analyzeQuery(sql)
    fields.forEach { println("Detected field: \${it.table}.\${it.column}") }
}
