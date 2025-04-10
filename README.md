import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.context.ApplicationContext
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.repository.CrudRepository
import org.springframework.stereotype.Repository
import kotlin.reflect.KClass
import kotlin.reflect.KFunction
import kotlin.reflect.full.declaredMemberFunctions
import kotlin.reflect.full.isSubclassOf
import kotlin.reflect.jvm.jvmErasure

@SpringBootTest
class CustomMethodLister {

    @Autowired
    private lateinit var applicationContext: ApplicationContext

    fun listCustomMethods() {
        // Spring Boot context'inden tüm @Repository bean'lerini al
        applicationContext.getBeansWithAnnotation(Repository::class.java).forEach { (_, bean) ->
            val repositoryClass = bean::class

            // Repository sınıfının superclasses'inde, interface'leri kontrol et
            val interfaces = repositoryClass.superclasses.filter { it.isInterface }

            interfaces.forEach { repoInterface ->
                println("➡️ Custom methods in interface: ${repoInterface.simpleName}")
                
                // Interface'deki tüm fonksiyonları al, Spring Data'nın metodları dışla
                repoInterface.declaredMemberFunctions
                    .filterNot { isSpringDataMethod(it) }
                    .forEach { method ->
                        println("  - ${method.name}")
                    }
            }
        }
    }

    // Spring Data metodlarını ayıklamak için kontrol fonksiyonu
    private fun isSpringDataMethod(method: KFunction<*>): Boolean {
        val commonSpringMethods = setOf("save", "delete", "findAll", "count", "existsById")
        val springPrefixes = listOf("findBy", "countBy", "deleteBy", "queryBy", "readBy", "getBy")

        // JpaRepository ve CrudRepository sınıflarındaki methodları kontrol et
        return CrudRepository::class.functions.any { it.name == method.name } ||
               JpaRepository::class.functions.any { it.name == method.name } ||
               method.name in commonSpringMethods ||
               springPrefixes.any { method.name.startsWith(it) }
    }
}
