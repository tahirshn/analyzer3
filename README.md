val methodName = method.name
val domainClass = (method.genericReturnType as? ParameterizedType)
    ?.actualTypeArguments
    ?.firstOrNull()
    ?.let { Class.forName(it.typeName) }
    ?: return@forEach

val partTree = PartTree(methodName, domainClass)

// findByNameAndStatusOrCreatedAt → ["name", "status", "createdAt"]
val allFields = partTree.parts.flatMap { orPart ->
    orPart.mapNotNull { part ->
        runCatching { part.property?.leafProperty?.segment }.getOrNull()
    }
}.toSet()

allFields.forEach { field ->
    val fieldMeta = entityFieldMap[field]
    if (fieldMeta != null) {
        val hasIndex = tableIndexes[entityTableName]?.any { it.columnName.equals(fieldMeta.columnName, ignoreCase = true) } == true
        if (!hasIndex) {
            println("    [MISSING INDEX] on inferred field '${fieldMeta.fieldName}' -> column '${fieldMeta.columnName}'")
        }
    } else {
        println("    [UNKNOWN FIELD] $field")
    }
}
