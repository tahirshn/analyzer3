import org.springframework.data.repository.query.parser.PartTree
import java.lang.reflect.Method

fun analyzeRepositoryMethod(method: Method, entityFieldMap: Map<String, FieldMeta>, entityTableName: String, tableIndexes: Map<String, List<IndexMeta>>, entityClass: Class<*>) {
    val queryAnnotation = method.getAnnotation(Query::class.java)
    if (queryAnnotation != null) {
        // --- @Query kısmı aynı kalabilir (SQL içinden analiz ediliyor) ---
        val queryText = queryAnnotation.value

        val conditionRegex = Regex("""(?:(?:where|and|or|on)\s+)(.+?)(?=(\s+(and|or|order\s+by|group\s+by|$)))""", RegexOption.IGNORE_CASE)
        val conditions = conditionRegex.findAll(queryText)
            .flatMap { it.groupValues[1].split("and", "or", ",") }
            .mapNotNull { condition ->
                Regex("""([a-zA-Z_][a-zA-Z0-9_]*)\.([a-zA-Z_][a-zA-Z0-9_]*)""")
                    .findAll(condition)
                    .map { it.groupValues[2] }
                    .toList()
            }
            .flatten()
            .toSet()

        conditions.forEach { field ->
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
        // --- PartTree ile parse edilmiş method ismi analizi ---
        val partTree = PartTree(method.name, entityClass)

        val allFields = partTree.parts.flatMap { orPart ->
            orPart.map { part ->
                part.property.segment // örnek: "name", "status", "createdAt"
            }
        }.toSet()

        allFields.forEach { field ->
            val fieldName = field.replaceFirstChar(Char::lowercase) // camelCase yap
            val fieldMeta = entityFieldMap[fieldName]
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
