import org.junit.jupiter.api.Test
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest
import org.springframework.data.domain.PageRequest
import org.springframework.data.domain.Sort
import javax.persistence.EntityManager
import kotlin.reflect.KClass
import kotlin.reflect.KType
import kotlin.reflect.full.*
import kotlin.reflect.jvm.jvmErasure

@DataJpaTest
class UniversalRepositoryTest {

    @Autowired
    lateinit var entityManager: EntityManager

    @Autowired
    lateinit var repositories: List<Any> // Tüm JPA repository'leri

    // Temel test verileri
    private val testData = mutableMapOf<KClass<*>, Any>()

    @Test
    fun `test all repository methods dynamically`() {
        repositories.forEach { repository ->
            val repoClass = repository::class
            println("\n⏳ Testing repository: ${repoClass.simpleName}")

            repoClass.declaredMemberFunctions
                .filterNot { it.name == "equals" || it.name == "hashCode" }
                .forEach { function ->
                    try {
                        val params = resolveParameters(function, repository)
                        val result = function.call(repository, *params)
                        
                        println("✅ ${repoClass.simpleName}.${function.name}() - " +
                                "Params: ${params.joinToString()}, " +
                                "Result: ${result?.toString()?.take(100)}...")
                        
                        // Result'ı entity ise testData'ya kaydet
                        result?.let { cacheResultIfEntity(it) }
                        if (result is Iterable<*>) {
                            result.forEach { it?.let { cacheResultIfEntity(it) } }
                        }
                    } catch (e: Exception) {
                        println("❌ ${repoClass.simpleName}.${function.name}() - " +
                               "Failed: ${e.message?.take(150)}")
                    }
                }
        }
    }

    private fun resolveParameters(function: KFunction<*>, repository: Any): Array<Any?> {
        return function.parameters
            .drop(1) // Receiver parametresini atla (repository instance'ı)
            .map { param ->
                when {
                    // Spring Data'nın özel tipleri
                    param.type.jvmErasure == PageRequest::class -> PageRequest.of(0, 10)
                    param.type.jvmErasure == Sort::class -> Sort.unsorted()
                    
                    // Daha önce oluşturulmuş entity'ler
                    testData.containsKey(param.type.jvmErasure) -> testData[param.type.jvmErasure]
                    
                    // Collection tipleri
                    param.type.jvmErasure.isSubclassOf(Collection::class) -> 
                        createCollectionForType(param.type)
                    
                    // Diğer kompleks tipler
                    param.type.classifier is KClass<*> && 
                    (param.type.classifier as KClass<*>).isData -> 
                        createDataClassInstance(param.type)
                    
                    // Basit tipler
                    else -> createSimpleValue(param.type)
                }
            }.toTypedArray()
    }

    private fun createCollectionForType(type: KType): Collection<*> {
        val elementType = type.arguments.first().type!!
        return when {
            type.jvmErasure == List::class -> listOf(createValueForType(elementType))
            type.jvmErasure == Set::class -> setOf(createValueForType(elementType))
            else -> emptyList()
        }
    }

    private fun createValueForType(type: KType): Any? {
        return when (type.jvmErasure) {
            String::class -> "test"
            Long::class -> 1L
            Int::class -> 1
            Boolean::class -> true
            else -> testData[type.jvmErasure] ?: createDataClassInstance(type)
        }
    }

    private fun createDataClassInstance(type: KType): Any {
        val kclass = type.jvmErasure
        return kclass.primaryConstructor!!.let { constructor ->
            constructor.parameters.associateWith { param ->
                createValueForType(param.type)
            }.let { argsMap ->
                constructor.callBy(argsMap)
            }
        }.also { instance ->
            // Entity ise persist et
            if (kclass.annotations.any { it.annotationClass == Entity::class }) {
                entityManager.persist(instance)
                entityManager.flush()
                testData[kclass] = instance
            }
        }
    }

    private fun createSimpleValue(type: KType): Any {
        return when (type.jvmErasure) {
            String::class -> "test_value"
            Long::class -> 1L
            Int::class -> 42
            Boolean::class -> true
            Double::class -> 3.14
            else -> throw IllegalArgumentException("Unsupported simple type: ${type.jvmErasure}")
        }
    }

    private fun cacheResultIfEntity(obj: Any) {
        val kclass = obj::class
        if (kclass.annotations.any { it.annotationClass == Entity::class }) {
            testData[kclass] = obj
        }
    }
}
