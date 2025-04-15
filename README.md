val repoBeans = applicationContext.getBeansOfType(CrudRepository::class.java)
        repoBeans.forEach { (name, repo) ->
            val repoClass = repo.javaClass
            val interfaceType = repoClass.genericInterfaces
                .firstOrNull { it is ParameterizedType } as? ParameterizedType
                ?: repoClass.superclass.genericInterfaces
                    .firstOrNull { it is ParameterizedType } as? ParameterizedType
                ?: return@forEach

            val entityClass = interfaceType.actualTypeArguments.getOrNull(0) as? Class<*> ?: return@forEach
            val entityTableName = resolveTableName(entityClass)
            val entityFieldMap = entityFields.filter { it.tableName == entityTableName }.associateBy { it.fieldName }

            println("\nREPOSITORY: $name -> ENTITY: ${entityClass.simpleName}")

            repoClass.methods.forEach { method ->
                val queryAnnotation = method.getAnnotation(Query::class.java)
                val queryText = queryAnnotation?.value ?: ""

                println("  METHOD: ${method.name}")
                if (queryText.isNotBlank()) {
                    println("    @Query: $queryText")

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
                    val inferredFields = Regex("findBy([A-Z][a-zA-Z0-9]*)").findAll(method.name)
                        .map { it.groupValues[1].replaceFirstChar { it.lowercase() } }
                        .toSet()

                    inferredFields.forEach { field ->
                        val fieldMeta = entityFieldMap[field]
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
