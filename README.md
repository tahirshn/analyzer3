import java.io.File

data class WhereCondition(val table: String, val column: String)

data class QueryAnalysis(val sql: String, val whereConditions: List<WhereCondition>)

data class SQLAnalysisResult(val queries: Map<String, QueryAnalysis>)

fun extractWhereConditions(query: String): List<WhereCondition> {
    val cleanedQuery = removeComments(query)
    val whereClause = findWhereClause(cleanedQuery) ?: return emptyList()
    return extractTableColumnPairs(whereClause)
}

fun removeComments(query: String): String {
    val singleLineRemoved = query.replace(Regex("--.*?$", RegexOption.MULTILINE), "")
    val blockRemoved = singleLineRemoved.replace(Regex("/\\*.*?\\*/", RegexOption.DOT_MATCHES_ALL), "")
    return blockRemoved
}

fun findWhereClause(query: String): String? {
    val match = Regex("(?i)WHERE\\s+(.*)").find(query)
    return match?.groupValues?.get(1)
}

fun extractTableColumnPairs(whereClause: String): List<WhereCondition> {
    val regex = Regex("(\\w+)\\.(\\w+)")
    return regex.findAll(whereClause)
        .map { matchResult ->
            val (table, column) = matchResult.destructured
            WhereCondition(table, column)
        }
        .toList()
}

fun analyzeSqlFile(filePath: String): SQLAnalysisResult {
    val file = File(filePath)
    if (!file.exists()) {
        return SQLAnalysisResult(mapOf("error" to QueryAnalysis("File not found", emptyList())))
    }

    val lines = file.readLines().map { it.trim() }.filter { it.isNotEmpty() }

    val results = lines.mapIndexed { index, sql ->
        val conditions = extractWhereConditions(sql)
        "Query ${index + 1}" to QueryAnalysis(sql, conditions)
    }.toMap()

    return SQLAnalysisResult(results)
}

fun main() {
    val result = analyzeSqlFile("feature_sql.log")

    if (result.queries.containsKey("error")) {
        println("Error: ${result.queries["error"]?.sql}")
    } else {
        result.queries.forEach { (queryName, analysis) ->
            println("--- $queryName ---")
            println("SQL: ${analysis.sql}")
            println("WHERE Conditions:")
            analysis.whereConditions.forEach {
                println("  Table: ${it.table}, Column: ${it.column}")
            }
        }
    }
}
