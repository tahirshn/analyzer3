 val repoInterface = repo.javaClass.interfaces.firstOrNull {
        CrudRepository::class.java.isAssignableFrom(it)
    } ?: return@forEach

    val entityClass = (repoInterface.genericInterfaces.firstOrNull {
        it is ParameterizedType
    } as? ParameterizedType)?.actualTypeArguments?.getOrNull(0) as? Class<*>
        ?: return@forEach
