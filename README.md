import org.junit.jupiter.api.Test
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.data.repository.Repository
import org.springframework.data.repository.CrudRepository
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.repository.PagingAndSortingRepository
import org.springframework.data.repository.query.QueryByExampleExecutor
import org.springframework.stereotype.Repository
import javax.persistence.EntityManager
import kotlin.reflect.KClass
import kotlin.reflect.KFunction
import kotlin.reflect.KParameter
import kotlin.reflect.full.*
import kotlin.reflect.jvm.jvmErasure

@SpringBootTest
class RepositoryInterfaceMethodTest {

    @Autowired
    private lateinit var entityManager: EntityManager

    @Autowired
    private lateinit var applicationContext: ApplicationContext

    // Spring Data JPA'nın temel interface'leri (filtrelemek için)
    private val springDataInterfaces = setOf(
        Repository::class,
        CrudRepository::class,
        JpaRepository::class,
        PagingAndSortingRepository::class,
        QueryByExampleExecutor::class
    )

    @Test
    fun `test all custom repository methods`() {
        applicationContext.getBeansWithAnnotation(org.springframework.stereotype.Repository::class)
            .forEach { (beanName, bean) ->
                val repositoryInterface = findRepositoryInterface(bean::class)
                if (repositoryInterface != null) {
                    println("\nTesting repository: ${repositoryInterface.simpleName}")
                    
                    val customMethods = findCustomMethods(repositoryInterface)
                    customMethods.forEach { method ->
                        try {
                            val params = resolveParameters(method.parameters)
                            println("Calling: ${method.name} with params: ${params.joinToString()}")
                            val result = method.call(bean, *params.toTypedArray())
                            println("Result: ${result?.toString()?.take(100)}...")
                            entityManager.clear()
                        } catch (e: Exception) {
                            println("Failed to execute ${repositoryInterface.simpleName}.${method.name}: ${e.message}")
                        }
                    }
                }
            }
    }

    private fun findRepositoryInterface(proxyClass: KClass<*>): KClass<*>? {
        return proxyClass.interfaces.firstOrNull { interfaceClass ->
            interfaceClass.hasAnnotation<org.springframework.stereotype.Repository>() ||
            AnnotationUtils.findAnnotation(interfaceClass.java, org.springframework.stereotype.Repository::class.java) != null
        }
    }

    private fun findCustomMethods(repositoryInterface: KClass<*>): List<KFunction<*>> {
        return repositoryInterface.declaredMemberFunctions.filterNot { method ->
            // Spring Data JPA'nın temel interface'lerinde bulunan metodları filtrele
            springDataInterfaces.any { parentInterface ->
                parentInterface.java.methods.any { parentMethod ->
                    parentMethod.name == method.name &&
                    parentMethod.parameterTypes.map { it.kotlin } == method.parameters.map { it.type.jvmErasure }
                }
            }
        }
    }

    private fun resolveParameters(parameters: List<KParameter>): List<Any?> {
        return parameters.map { param ->
            when (val type = param.type.jvmErasure) {
                String::class -> "test"
                Int::class, Integer::class -> 1
                Long::class, java.lang.Long::class -> 1L
                Boolean::class -> true
                List::class -> emptyList<Any>()
                Pageable::class -> PageRequest.of(0, 10)
                else -> {
                    try {
                        // JPA Entity'leri için veritabanından örnek bul
                        entityManager.entityManagerFactory.metamodel.entities
                            .firstOrNull { it.javaType == type.java }
                            ?.let { entityManager.find(it.javaType, 1) }
                            ?: type.createInstanceOrNull()
                    } catch (e: Exception) {
                        null
                    }
                }
            }
        }
    }

    private fun KClass<*>.createInstanceOrNull(): Any? {
        return try {
            if (this.isData) {
                primaryConstructor?.let { constructor ->
                    constructor.parameters.associateWith { param ->
                        resolveParameters(listOf(param)).first()
                    }.let(constructor::callBy)
                }
            } else {
                createInstance()
            }
        } catch (e: Exception) {
            null
        }
    }
}
