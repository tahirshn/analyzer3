package com.example.index

import jakarta.persistence.EntityManager
import jakarta.persistence.PersistenceContext
import org.junit.jupiter.api.Test
import org.springframework.boot.test.context.SpringBootTest

@SpringBootTest
class IndexInspectorTest {

    @PersistenceContext
    lateinit var entityManager: EntityManager

    @Test
    fun `should fetch and print table indexes without transaction`() {
        val sql = """
            SELECT 
                t.relname AS table_name,
                i.relname AS index_name,
                a.attname AS column_name,
                ix.indisunique AS is_unique,
                ix.indisprimary AS is_primary
            FROM 
                pg_class t,
                pg_class i,
                pg_index ix,
                pg_attribute a
            WHERE 
                t.oid = ix.indrelid
                AND i.oid = ix.indexrelid
                AND a.attrelid = t.oid
                AND a.attnum = ANY(ix.indkey)
                AND t.relkind = 'r'
            ORDER BY
                t.relname, i.relname
        """.trimIndent()

        val result = entityManager.createNativeQuery(sql).resultList

        if (result.isEmpty()) {
            println("❌ No indexes found.")
        } else {
            println("✅ Index list:")
            result.forEach { row ->
                val columns = row as Array<*>
                println("Table: ${columns[0]}, Index: ${columns[1]}, Column: ${columns[2]}, Unique: ${columns[3]}, Primary: ${columns[4]}")
            }
        }
    }
}
