import org.junit.jupiter.api.Test
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.repository.CrudRepository
import org.springframework.stereotype.Repository
import javax.persistence.EntityManager
import kotlin.reflect.KClass
import kotlin.reflect.KFunction
import kotlin.reflect.KParameter
import kotlin.reflect.full.*
import kotlin.reflect.jvm.jvmErasure

@SpringBootTest
class CustomRepositoryMethodTest {

    @Autowired
    private lateinit var entityManager: EntityManager

    @Autowired
    private lateinit var applicationContext: ApplicationContext

    @Test
    fun `test all custom repository methods`() {
        applicationContext.getBeansWithAnnotation(Repository::class.java).values.forEach { repository ->
            val repositoryClass = repository::class
            println("\nTesting custom methods in: ${repositoryClass.simpleName}")

            repositoryClass.declaredMemberFunctions
                .filterNot { isSpringDataMethod(it) } // Spring Data'nın otomatik metodlarını filtrele
                .filterNot { it.isAbstract } // Soyut metodları atla
                .forEach { method ->
                    try {
                        val params = resolveParameters(method.parameters.drop(1)) // 'this' parametresini atla
                        println("Calling custom method: ${method.name}(${params.joinToString()})")
                        
                        val result = method.call(repository, *params.toTypedArray())
                        println("Result: ${result?.toString()?.take(200)}...")
                        
                        entityManager.clear() // Persistence context'i temizle
                    } catch (e: Exception) {
                        println("❌ Failed to execute ${method.name}: ${e.message?.take(150)}")
                    }
                }
        }
    }

    private fun isSpringDataMethod(method: KFunction<*>): Boolean {
        // Spring Data JPA'nın otomatik eklediği metodları tespit et
        val declaringClass = method.annotations.find { it.annotationClass == Repository::class } != null
        return CrudRepository::class.java.methods.any { it.name == method.name } ||
               JpaRepository::class.java.methods.any { it.name == method.name } ||
               method.name.startsWith("findBy") ||
               method.name.startsWith("countBy") ||
               method.name.startsWith("deleteBy") ||
               method.name.startsWith("queryBy") ||
               method.name.startsWith("readBy") ||
               method.name.startsWith("getBy") ||
               method.name == "save" ||
               method.name == "delete" ||
               method.name == "findAll" ||
               method.name == "count" ||
               method.name == "existsById"
    }

    private fun resolveParameters(parameters: List<KParameter>): List<Any?> {
        return parameters.map { param ->
            when (val type = param.type.jvmErasure) {
                String::class -> "test_value_${parameters.indexOf(param)}"
                Int::class, Integer::class -> parameters.indexOf(param) + 1
                Long::class, java.lang.Long::class -> (parameters.indexOf(param) + 1).toLong()
                Boolean::class -> true
                List::class -> emptyList<Any>()
                Pageable::class -> PageRequest.of(0, 5)
                else -> {
                    // JPA Entity'leri için örnek kayıt oluştur
                    entityManager.entityManagerFactory.metamodel.entities
                        .firstOrNull { it.javaType == type.java }
                        ?.let { findOrCreateEntity(type) }
                        ?: try { type.createInstance() } catch (e: Exception) { null }
                }
            }.also { 
                if (it == null) println("⚠️ Could not resolve parameter: ${param.name} of type ${param.type}") 
            }
        }
    }

    private fun findOrCreateEntity(entityClass: KClass<*>): Any? {
        return try {
            entityManager.createQuery("FROM ${entityClass.simpleName}", entityClass.java)
                .setMaxResults(1)
                .resultList
                .firstOrNull()
                ?: entityClass.createInstance().also { entityManager.persist(it); entityManager.flush() }
        } catch (e: Exception) {
            println("⚠️ Could not create entity: ${entityClass.simpleName}")
            null
        }
    }
}
