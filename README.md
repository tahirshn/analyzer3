package com.sqlinspector.analyzer

import net.sf.jsqlparser.parser.CCJSqlParserUtil
import net.sf.jsqlparser.statement.select.*
import net.sf.jsqlparser.statement.Statement
import net.sf.jsqlparser.schema.Column
import net.sf.jsqlparser.expression.*
import net.sf.jsqlparser.util.TablesNamesFinder
import net.sf.jsqlparser.util.deparser.ExpressionDeParser

data class SqlAnalysisResult(
    val whereColumns: Set<String>,
    val joinColumns: Set<String>,
    val allReferencedTables: Set<String>
)

object SqlAnalyzer {

    fun analyzeRawQuery(query: String): SqlAnalysisResult {
        val whereColumns = mutableSetOf<String>()
        val joinColumns = mutableSetOf<String>()
        val referencedTables = mutableSetOf<String>()

        val statement: Statement = CCJSqlParserUtil.parse(query)
        if (statement is Select) {
            val selectBody = statement.selectBody
            extractTables(statement, referencedTables)
            visitSelect(selectBody, whereColumns, joinColumns)
        }

        return SqlAnalysisResult(
            whereColumns = whereColumns,
            joinColumns = joinColumns,
            allReferencedTables = referencedTables
        )
    }

    private fun extractTables(statement: Statement, tables: MutableSet<String>) {
        val finder = TablesNamesFinder()
        val found = finder.getTableList(statement)
        tables.addAll(found)
    }

    private fun visitSelect(
        selectBody: SelectBody,
        whereColumns: MutableSet<String>,
        joinColumns: MutableSet<String>
    ) {
        when (selectBody) {
            is PlainSelect -> {
                extractWhereColumns(selectBody.where, whereColumns)
                extractJoins(selectBody.joins, joinColumns)
                extractSubselects(selectBody.fromItem, whereColumns, joinColumns)
            }

            is SetOperationList -> {
                selectBody.selects.forEach {
                    visitSelect(it, whereColumns, joinColumns)
                }
                selectBody.withItemsList?.forEach {
                    visitWithItem(it, whereColumns, joinColumns)
                }
            }

            is WithItem -> {
                visitWithItem(selectBody, whereColumns, joinColumns)
            }
        }
    }

    private fun visitWithItem(
        withItem: WithItem,
        whereColumns: MutableSet<String>,
        joinColumns: MutableSet<String>
    ) {
        withItem.selectBody?.let {
            visitSelect(it, whereColumns, joinColumns)
        }
    }

    private fun extractWhereColumns(
        expression: Expression?,
        columns: MutableSet<String>
    ) {
        expression?.accept(object : ExpressionDeParser() {
            override fun visit(column: Column) {
                columns.add(column.fullyQualifiedName)
            }
        })
    }

    private fun extractJoins(
        joins: List<Join>?,
        joinColumns: MutableSet<String>
    ) {
        joins?.forEach { join ->
            join.onExpression?.accept(object : ExpressionDeParser() {
                override fun visit(column: Column) {
                    joinColumns.add(column.fullyQualifiedName)
                }
            })

            extractSubselects(join.rightItem, joinColumns, joinColumns)
        }
    }

    private fun extractSubselects(
        fromItem: FromItem?,
        whereColumns: MutableSet<String>,
        joinColumns: MutableSet<String>
    ) {
        when (fromItem) {
            is SubSelect -> {
                visitSelect(fromItem.selectBody, whereColumns, joinColumns)
            }
            is LateralSubSelect -> {
                visitSelect(fromItem.subSelect.selectBody, whereColumns, joinColumns)
            }
            is ValuesList -> {
                // Skip
            }
            is TableFunction -> {
                // Optional: handle function call arguments
            }
        }
    }
}
