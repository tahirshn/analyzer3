package com.example.jpqltonative

import jakarta.persistence.EntityManager
import org.hibernate.engine.spi.SessionFactoryImplementor
import org.hibernate.query.sqm.SqmInterpreter
import org.hibernate.query.sqm.SqmTranslatorFactory
import org.hibernate.query.sqm.tree.SqmStatement
import org.hibernate.query.sqm.internal.SemanticQueryInterpreter
import org.hibernate.query.spi.QueryEngine
import org.hibernate.query.sql.internal.StandardSqmTranslatorFactory
import org.springframework.stereotype.Component

@Component
class JpqlToNativeConverter(
    private val entityManager: EntityManager
) {
    fun convert(jpql: String): String {
        val session = entityManager.unwrap(org.hibernate.Session::class.java)
        val factory = session.sessionFactory as SessionFactoryImplementor
        val queryEngine: QueryEngine = factory.queryEngine

        // Parse JPQL -> SQM (Semantic Query Model)
        val sqmStatement: SqmStatement<*> = SemanticQueryInterpreter.parse(jpql, queryEngine, null)

        // Translate SQM -> SQL AST -> SQL String
        val sqmTranslatorFactory: SqmTranslatorFactory = StandardSqmTranslatorFactory()
        val translator = sqmTranslatorFactory.createSelectTranslator(
            sqmStatement,
            factory,
            queryEngine.sqmFunctionRegistry
        )

        val jdbcSelect = translator.translate()
        return jdbcSelect.sqlString
    }
}
