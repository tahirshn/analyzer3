val aliasMap = mutableMapOf<String, String>()
val aliasRegex = Regex("""\b(from|join)\s+(\w+)\s+(\w+)""", RegexOption.IGNORE_CASE)
aliasRegex.findAll(queryText).forEach { match ->
    val tableOrEntity = match.groupValues[2]
    val alias = match.groupValues[3]
    aliasMap[alias] = tableOrEntity
}

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
