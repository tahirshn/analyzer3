@Service
class QueryAnalysisService(
    private val queryExtractor: JpaQueryExtractor,
    private val repositoryScanner: RepositoryScanner
) {
    fun analyzeAndPrintQueries() {
        val repositories = repositoryScanner.findAllRepositoryClasses()
        val queries = queryExtractor.extractAllQueries(repositories)

        queries.forEach { query ->
            println("""
                === Repository: ${query.repositoryName}.${query.methodName}() ===
                Source: ${query.querySource}
                Type: ${if (query.isNative) "Native SQL" else "JPQL"}
                Parameters: ${query.paramNames.joinToString()}
                Query:
                ${query.rawQuery}
                
            """.trimIndent())
        }
    }
}

@Component
class RepositoryScanner(
    private val applicationContext: ApplicationContext
) {
    fun findAllRepositoryClasses(): List<KClass<*>> {
        return applicationContext.getBeansWithAnnotation(Repository::class.java)
            .values
            .map { it::class }
            .distinct()
    }
}
