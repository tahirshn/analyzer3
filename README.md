sping hibernate


import org.hibernate.SessionFactory
import org.hibernate.engine.spi.SessionFactoryImplementor
import org.hibernate.hql.internal.ast.QueryTranslatorImpl
import org.springframework.data.jpa.repository.Query
import org.springframework.data.repository.core.support.RepositoryFactorySupport
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
    private val sessionFactory: SessionFactoryImplementor by lazy {
        entityManager.entityManagerFactory.unwrap(SessionFactory::class.java) as SessionFactoryImplementor
    }

    data class RepositoryQueryInfo(
        val repositoryName: String,
        val methodName: String,
        val querySource: QuerySource,
        val rawQuery: String,
        val isNative: Boolean
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
                    val query = queryAnnotation.value
                    val isNative = queryAnnotation.nativeQuery
                    val translatedQuery = if (!isNative) translateHqlToSql(query) else query

                    RepositoryQueryInfo(
                        repositoryName = repositoryClass.simpleName!!,
                        methodName = function.name,
                        querySource = QuerySource.ANNOTATION,
                        rawQuery = translatedQuery,
                        isNative = isNative
                    )
                }
                isQueryMethod(method.name) -> {
                    // Method name inference ile oluşturulmuş query
                    val query = generateQueryFromMethodName(repositoryClass, function)
                    RepositoryQueryInfo(
                        repositoryName = repositoryClass.simpleName!!,
                        methodName = function.name,
                        querySource = QuerySource.METHOD_NAME,
                        rawQuery = query,
                        isNative = false
                    )
                }
                else -> null
            }
        }
    }

    private fun isQueryMethod(methodName: String): Boolean {
        val queryKeywords = listOf("find", "read", "get", "query", "search", "stream", "count", "exists", "delete", "remove")
        return queryKeywords.any { methodName.startsWith(it, ignoreCase = true) }
    }

    private fun generateQueryFromMethodName(repositoryClass: KClass<*>, function: KFunction<*>): String {
        // Basit bir implementasyon - gerçekte Spring'in query creation mekanizması daha karmaşıktır
        val entityClass = repositoryClass.supertypes
            .firstOrNull { it.toString().contains("JpaRepository") }
            ?.arguments?.get(0)?.type?.classifier as? KClass<*>
            ?: return "Could not determine entity class"

        val entityName = entityClass.simpleName!!
        val methodName = function.name

        return when {
            methodName.startsWith("findBy") -> {
                val property = methodName.removePrefix("findBy").decapitalize()
                "SELECT e FROM $entityName e WHERE e.$property = ?1"
            }
            methodName.startsWith("countBy") -> {
                val property = methodName.removePrefix("countBy").decapitalize()
                "SELECT COUNT(e) FROM $entityName e WHERE e.$property = ?1"
            }
            methodName.startsWith("deleteBy") -> {
                val property = methodName.removePrefix("deleteBy").decapitalize()
                "DELETE FROM $entityName e WHERE e.$property = ?1"
            }
            else -> "Generated query for $methodName on $entityName"
        }.let { translateHqlToSql(it) }
    }

    private fun translateHqlToSql(hql: String): String {
        return try {
            val translators = QueryTranslatorImpl(
                queryIdentifier = hql,
                query = hql,
                enabledFilters = emptyMap(),
                factory = sessionFactory,
                replacements = QueryTranslatorImpl.QUERY_TRANSLATOR
            )
            translators.compile(emptyMap(), false)
            translators.sqlString
        } catch (e: Exception) {
            "Error translating HQL to SQL: ${e.message}"
        }
    }
}
