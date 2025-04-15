package com.sqlinspector.analyzer

import jakarta.persistence.Entity
import org.springframework.core.annotation.AnnotatedElementUtils
import org.springframework.data.jpa.repository.Query
import java.lang.reflect.Method

sealed interface QueryType {
    data object JPQL : QueryType
    data object Native : QueryType
    data object Derived : QueryType
    data object Unknown : QueryType
}

data class RepositoryMethodMetadata(
    val repositoryClass: Class<*>,
    val method: Method,
    val queryType: QueryType,
    val query: String?,
    val parameters: List<MethodParameterMetadata>,
    val whereConditions: List<String>,
    val referencedEntities: List<EntityReference>
)

data class MethodParameterMetadata(
    val name: String,
    val type: Class<*>,
    val isEntity: Boolean,
    val entityMetadata: EntityMetadata? = null
)

data class EntityReference(
    val className: String,
    val tableName: String,
    val idFields: List<String>
)

data class EntityMetadata(
    val tableName: String,
    val idFields: List<String>
)

object MethodQueryAnalyzer {

    fun analyze(repositoryClass: Class<*>): List<RepositoryMethodMetadata> {
        return repositoryClass.declaredMethods.map { method ->
            val queryAnnotation = AnnotatedElementUtils.findMergedAnnotation(method, Query::class.java)
            val (queryType, queryString) = if (queryAnnotation != null) {
                if (queryAnnotation.nativeQuery) QueryType.Native to queryAnnotation.value
                else QueryType.JPQL to queryAnnotation.value
            } else {
                if (method.name.matches(Regex("(find|read|get|exists|count|delete|remove).+By.+"))) {
                    QueryType.Derived to null
                } else QueryType.Unknown to null
            }

            val paramMetadata = method.parameters.mapIndexed { idx, param ->
                val isEntity = param.type.isAnnotationPresent(Entity::class.java)
                val metadata = if (isEntity) EntityInspector.inspect(param.type) else null
                MethodParameterMetadata(
                    name = param.name ?: "arg$idx",
                    type = param.type,
                    isEntity = isEntity,
                    entityMetadata = metadata
                )
            }

            val referencedEntities = paramMetadata
                .filter { it.isEntity && it.entityMetadata != null }
                .map {
                    EntityReference(
                        className = it.type.simpleName,
                        tableName = it.entityMetadata!!.tableName,
                        idFields = it.entityMetadata!!.idFields
                    )
                }

            val whereConditions = if (queryString != null) {
                QueryWhereClauseExtractor.extract(queryString, queryType)
            } else if (queryType == QueryType.Derived) {
                DerivedQueryParser.parseConditions(method.name)
            } else emptyList()

            RepositoryMethodMetadata(
                repositoryClass = repositoryClass,
                method = method,
                queryType = queryType,
                query = queryString,
                parameters = paramMetadata,
                whereConditions = whereConditions,
                referencedEntities = referencedEntities
            )
        }
    }
}

object DerivedQueryParser {
    fun parseConditions(methodName: String): List<String> {
        val wherePart = methodName.substringAfterLast("By", "")
        return wherePart
            .split("And", "Or")
            .map { it.replace(Regex("(Is|Equals|Like|Not|In|Between|LessThan|GreaterThan|Before|After)", "")) }
            .filter { it.isNotBlank() }
            .map { it.replaceFirstChar { c -> c.lowercase() } }
    }
}

object QueryWhereClauseExtractor {
    fun extract(query: String, type: QueryType): List<String> {
        val lower = query.lowercase()
        val whereIndex = lower.indexOf("where")
        if (whereIndex == -1) return emptyList()
        val whereClause = query.substring(whereIndex + 5).split("group by", "order by", "limit", ignoreCase = true)[0]
        return whereClause.split("and", "or", ignoreCase = true)
            .map { it.trim().removeSuffix(")").removePrefix("(") }
            .filter { it.isNotBlank() }
    }
}

object EntityInspector {
    fun inspect(clazz: Class<*>): EntityMetadata {
        val tableName = clazz.getAnnotation(jakarta.persistence.Table::class.java)?.name
            ?: clazz.simpleName.replace(Regex("([a-z])([A-Z])"), "$1-$2").lowercase()

        val idFields = clazz.declaredFields.filter {
            it.isAnnotationPresent(jakarta.persistence.Id::class.java)
        }.map { it.name }

        return EntityMetadata(
            tableName = tableName,
            idFields = idFields
        )
    }
}
