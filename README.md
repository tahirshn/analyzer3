val resolvableType = org.springframework.core.ResolvableType.forClass(repoClass)
    .`as`(CrudRepository::class.java)

val entityClass = resolvableType.generic(0).resolve() ?: return@forEac
