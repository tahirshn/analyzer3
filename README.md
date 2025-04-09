import org.springframework.data.jpa.repository.Query
import org.springframework.stereotype.Component
import javax.persistence.EntityManager
import javax.persistence.PersistenceContext
import kotlin.reflect.KClass
import kotlin.reflect.full.declaredFunctions
import kotlin.reflect.full.findAnnotation
import kotlin.reflect.jvm.javaMethod

@Component
class JpaQueryExtractor(
    @PersistenceContext private val entityManager: EntityManager
) {
    data class RepositoryQueryInfo(
        val repositoryName: String,
        val methodName: String,
        val querySource: QuerySource,
        val rawQuery: String,
        val isNative: Boolean,
        val paramNames: List<String>
    )

    enum class QuerySource {
        ANNOTATION, METHOD_NAME, UNKNOWN
    }

    /**
     * Tüm repository'lerdeki query'leri çıkarır
     */
    fun extractAllQueries(repositories: List<KClass<*>>): List<RepositoryQueryInfo> {
        return repositories.flatMap { extractQueriesFromRepository(it) }
    }

    /**
     * Tek bir repository'deki query'leri çıkarır
     */
    fun extractQueriesFromRepository(repositoryClass: KClass<*>): List<RepositoryQueryInfo> {
        return repositoryClass.declaredFunctions.mapNotNull { function ->
            val queryAnnotation = function.findAnnotation<Query>()
            val method = function.javaMethod ?: return@mapNotNull null

            when {
                queryAnnotation != null -> {
                    // @Query ile tanımlanmış query
                    extractAnnotationQuery(repositoryClass, function, queryAnnotation)
                }
                isQueryMethod(method.name) -> {
                    // Method name inference ile oluşturulmuş query
                    extractDerivedQuery(repositoryClass, function)
                }
                else -> null
            }
        }
    }

    private fun extractAnnotationQuery(
        repositoryClass: KClass<*>,
        function: KFunction<*>,
        queryAnnotation: Query
    ): RepositoryQueryInfo {
        val query = queryAnnotation.value
        val isNative = queryAnnotation.nativeQuery
        
        // Parametre isimlerini çıkar
        val paramNames = function.annotations
            .filterIsInstance<Param>()
            .associate { it.value to it.value }
            .values.toList()

        return RepositoryQueryInfo(
            repositoryName = repositoryClass.simpleName!!,
            methodName = function.name,
            querySource = QuerySource.ANNOTATION,
            rawQuery = query,
            isNative = isNative,
            paramNames = paramNames
        )
    }

    private fun extractDerivedQuery(
        repositoryClass: KClass<*>,
        function: KFunction<*>
    ): RepositoryQueryInfo {
        val entityClass = resolveEntityClass(repositoryClass)
        val entityName = entityClass?.simpleName ?: "UnknownEntity"
        val methodName = function.name

        val (query, paramNames) = when {
            methodName.startsWith("findBy") -> {
                val parts = parseMethodName(methodName.removePrefix("findBy"))
                val whereClause = parts.joinToString(" AND ") { "e.${it.property} = ?${it.index + 1}" }
                "SELECT e FROM $entityName e WHERE $whereClause" to parts.map { it.property }
            }
            methodName.startsWith("countBy") -> {
                val parts = parseMethodName(methodName.removePrefix("countBy")))
                val whereClause = parts.joinToString(" AND ") { "e.${it.property} = ?${it.index + 1}" }
                "SELECT COUNT(e) FROM $entityName e WHERE $whereClause" to parts.map { it.property }
            }
            methodName.startsWith("deleteBy") -> {
                val parts = parseMethodName(methodName.removePrefix("deleteBy")))
                val whereClause = parts.joinToString(" AND ") { "e.${it.property} = ?${it.index + 1}" }
                "DELETE FROM $entityName e WHERE $whereClause" to parts.map { it.property }
            }
            else -> "Unknown query for method $methodName" to emptyList()
        }

        return RepositoryQueryInfo(
            repositoryName = repositoryClass.simpleName!!,
            methodName = function.name,
            querySource = QuerySource.METHOD_NAME,
            rawQuery = query,
            isNative = false,
            paramNames = paramNames
        )
    }

    private data class MethodPart(val property: String, val index: Int)

    private fun parseMethodName(methodPart: String): List<MethodPart> {
        val parts = methodPart.split("And")
        return parts.mapIndexed { index, part ->
            MethodPart(property = part.decapitalize(), index = index)
        }
    }

    private fun resolveEntityClass(repositoryClass: KClass<*>): Class<*>? {
        return repositoryClass.supertypes
            .firstOrNull { it.toString().contains("JpaRepository") }
            ?.arguments?.get(0)?.type?.classifier?.let {
                (it as? KClass<*>)?.java
            }
    }

    private fun isQueryMethod(methodName: String): Boolean {
        val queryKeywords = listOf("find", "read", "get", "query", "search", "stream", 
                                 "count", "exists", "delete", "remove")
        return queryKeywords.any { methodName.startsWith(it, ignoreCase = true) }
    }
}
