import org.junit.jupiter.api.Test
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.core.annotation.AnnotationUtils
import org.springframework.stereotype.Component
import org.springframework.stereotype.Repository
import javax.persistence.EntityManager
import kotlin.reflect.KClass
import kotlin.reflect.KType
import kotlin.reflect.full.*
import kotlin.reflect.jvm.jvmErasure

@SpringBootTest
class ComprehensiveRepositoryTest {

    @Autowired
    private lateinit var entityManager: EntityManager

    @Autowired
    private lateinit var applicationContext: ApplicationContext

    @Test
    fun `test all repository methods`() {
        // 1. Tüm Spring bean'lerini al
        val allBeans = applicationContext.beanDefinitionNames
        
        allBeans.forEach { beanName ->
            val bean = applicationContext.getBean(beanName)
            val beanClass = bean::class
            
            // 2. Sadece @Repository veya @Component ile işaretlenmiş sınıfları filtrele
            if (isRepository(beanClass)) {
                println("\nTesting repository: ${beanClass.simpleName}")
                
                // 3. Sınıftaki tüm metodları al (üst sınıflardaki Spring Data JPA metodlarını hariç tut)
                val declaredMethods = beanClass.declaredMemberFunctions
                    .filterNot { it.name.startsWith("get") || it.name.startsWith("set") }
                    .filterNot { it.isAbstract }
                
                declaredMethods.forEach { method ->
                    try {
                        // 4. Metod parametrelerini oluştur
                        val params = resolveParameters(method.parameters.drop(1)) // 'this' parametresini atla
                        
                        // 5. Metodu çağır
                        println("Calling: ${method.name} with params: ${params.joinToString()}")
                        val result = method.call(bean, *params.toTypedArray())
                        
                        // 6. Sonucu logla
                        println("Result: ${result?.toString()?.take(100)}...")
                        
                        // 7. EntityManager'ı temizle
                        entityManager.clear()
                    } catch (e: Exception) {
                        println("Failed to execute ${beanClass.simpleName}.${method.name}: ${e.message}")
                    }
                }
            }
        }
    }

    private fun isRepository(clazz: KClass<*>): Boolean {
        return clazz.hasAnnotation<Repository>() || 
               clazz.hasAnnotation<Component>() ||
               AnnotationUtils.findAnnotation(clazz.java, Repository::class.java) != null ||
               AnnotationUtils.findAnnotation(clazz.java, Component::class.java) != null
    }

    private fun resolveParameters(parameters: List<KParameter>): List<Any?> {
        return parameters.map { param ->
            when (param.type.jvmErasure) {
                String::class -> "test"
                Int::class, Integer::class -> 1
                Long::class, java.lang.Long::class -> 1L
                Boolean::class -> true
                List::class -> emptyList<Any>()
                Pageable::class -> PageRequest.of(0, 10)
                else -> {
                    // Karmaşık tipler için örnek nesne oluştur
                    try {
                        param.type.jvmErasure.createInstance()
                    } catch (e: Exception) {
                        // JPA Entity'leri için boş instance
                        entityManager.entityManagerFactory.metamodel.entities
                            .firstOrNull { it.javaType == param.type.jvmErasure.java }
                            ?.let { entityManager.find(it.javaType, 1) }
                            ?: null
                    }
                }
            }
        }
    }
}
