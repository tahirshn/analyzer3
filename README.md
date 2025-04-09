import org.springframework.data.jpa.repository.Query
import org.springframework.stereotype.Component
import javax.persistence.EntityManager
import kotlin.reflect.KClass
import kotlin.reflect.full.declaredFunctions
import kotlin.reflect.full.findAnnotation
import kotlin.reflect.jvm.javaMethod

@Component
class JpaQueryExtractor {

    data class RepositoryQueryInfo(
        val repositoryName: String,
        val methodName: String,
        val querySource: QuerySource,
        val query: String,
        val isNative: Boolean
    )

    enum class QuerySource {
        ANNOTATION, 
        METHOD_NAME, 
        DERIVED, 
        UNKNOWN
    }

    /**
     * Repository'lerdeki tüm query'leri çıkarır
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
                    RepositoryQueryInfo(
                        repositoryName = repositoryClass.simpleName!!,
                        methodName = function.name,
                        querySource = QuerySource.ANNOTATION,
                        query = queryAnnotation.value,
                        isNative = queryAnnotation.nativeQuery
                    )
                }
                isQueryMethod(method.name) -> {
                    // Method name'den türetilmiş query
                    val query = generateQueryFromMethodName(repositoryClass, function)
                    RepositoryQueryInfo(
                        repositoryName = repositoryClass.simpleName!!,
                        methodName = function.name,
                        querySource = QuerySource.METHOD_NAME,
                        query = query,
                        isNative = false
                    )
                }
                else -> null
            }
        }
    }

    private fun isQueryMethod(methodName: String): Boolean {
        val queryKeywords = listOf(
            "find", "read", "get", "query", "search", 
            "stream", "count", "exists", "delete", "remove"
        )
        return queryKeywords.any { methodName.startsWith(it, ignoreCase = true) }
    }

    private fun generateQueryFromMethodName(repositoryClass: KClass<*>, function: KFunction<*>): String {
        // Repository'nin entity tipini bul
        val entityClass = repositoryClass.supertypes
            .firstOrNull { it.toString().contains("JpaRepository") }
            ?.arguments?.get(0)?.type?.classifier as? KClass<*>
            ?: return "Could not determine entity class"

        val entityName = entityClass.simpleName!!
        val methodName = function.name

        return when {
            methodName.startsWith("findBy") -> {
                val properties = extractProperties(methodName.removePrefix("findBy"))
                "SELECT e FROM $entityName e WHERE ${buildWhereClause(properties)}"
            }
            methodName.startsWith("countBy") -> {
                val properties = extractProperties(methodName.removePrefix("countBy"))
                "SELECT COUNT(e) FROM $entityName e WHERE ${buildWhereClause(properties)}"
            }
            methodName.startsWith("deleteBy") -> {
                val properties = extractProperties(methodName.removePrefix("deleteBy"))
                "DELETE FROM $entityName e WHERE ${buildWhereClause(properties)}"
            }
            methodName.startsWith("existsBy") -> {
                val properties = extractProperties(methodName.removePrefix("existsBy"))
                "SELECT CASE WHEN COUNT(e) > 0 THEN true ELSE false END FROM $entityName e WHERE ${buildWhereClause(properties)}"
            }
            else -> "Derived query for $methodName on $entityName"
        }
    }

    private fun extractProperties(methodPart: String): List<String> {
        // "FirstNameAndLastName" -> ["firstName", "lastName"]
        return methodPart.split("(?=[A-Z])".toRegex())
            .map { it.decapitalize() }
            .filter { it.isNotBlank() }
    }

    private fun buildWhereClause(properties: List<String>): String {
        return properties.joinToString(" AND ") { "e.${it} = ?${properties.indexOf(it) + 1}" }
    }
}
