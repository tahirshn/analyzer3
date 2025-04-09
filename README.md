import org.junit.jupiter.api.Test
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest
import kotlin.reflect.full.declaredMemberFunctions
import kotlin.reflect.jvm.isAccessible

@DataJpaTest
class RepositoryMethodInvokerTest {

    @Autowired
    lateinit var repositories: List<Any> // Tüm repository'ler otomatik enjekte edilir

    @Test
    fun `invoke all repository methods`() {
        repositories.forEach { repository ->
            val repositoryClass = repository::class
            println("\nTesting repository: ${repositoryClass.simpleName}")

            repositoryClass.declaredMemberFunctions.forEach { function ->
                try {
                    function.isAccessible = true
                    when (function.parameters.size) {
                        1 -> function.call(repository) // No-arg methods
                        2 -> { // Methods with parameters
                            val mockParam = createMockParam(function.parameters[1].type)
                            function.call(repository, mockParam)
                        }
                        else -> println("Skipping ${function.name} - complex parameters")
                    }
                    println("Called: ${function.name}")
                } catch (e: Exception) {
                    println("Failed to call ${function.name}: ${e.message}")
                }
            }
        }
    }

    private fun createMockParam(type: kotlin.reflect.KType): Any {
        // Basit mock nesneleri oluştur
        return when (type.classifier) {
            String::class -> "test"
            Long::class -> 1L
            else -> Any() // Daha kompleks tipler için genişletebilirsiniz
        }
    }
}
