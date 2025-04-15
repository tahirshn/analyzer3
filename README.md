import net.sf.jsqlparser.parser.CCJSqlParserUtil
import net.sf.jsqlparser.statement.select.*
import net.sf.jsqlparser.statement.Statement
import net.sf.jsqlparser.util.TablesNamesFinder
import net.sf.jsqlparser.schema.Column
import net.sf.jsqlparser.expression.operators.relational.EqualsTo
import net.sf.jsqlparser.expression.ExpressionVisitorAdapter
import net.sf.jsqlparser.statement.select.PlainSelect
import net.sf.jsqlparser.statement.select.SubSelect

data class SqlAnalysisResult(
    val tables: Set<String>,
    val whereColumns: Set<String>,
    val joinColumns: Set<String>
)

object SqlAnalyzer {

    fun analyze(sql: String): SqlAnalysisResult {
        val statement: Statement = CCJSqlParserUtil.parse(sql)

        val tables = mutableSetOf<String>()
        val whereColumns = mutableSetOf<String>()
        val joinColumns = mutableSetOf<String>()

        val tableFinder = TablesNamesFinder()
        tables.addAll(tableFinder.getTableList(statement))

        if (statement is Select) {
            val selectBody = statement.selectBody
            visitSelect(selectBody, whereColumns, joinColumns)
        }

        return SqlAnalysisResult(
            tables = tables,
            whereColumns = whereColumns,
            joinColumns = joinColumns
        )
    }

    private fun visitSelect(
        selectBody: SelectBody,
        whereColumns: MutableSet<String>,
        joinColumns: MutableSet<String>
    ) {
        when (selectBody) {
            is PlainSelect -> {
                selectBody.where?.accept(object : ExpressionVisitorAdapter() {
                    override fun visit(column: Column) {
                        whereColumns.add(column.fullyQualifiedName)
                    }
                })

                selectBody.joins?.forEach { join ->
                    join.onExpression?.accept(object : ExpressionVisitorAdapter() {
                        override fun visit(column: Column) {
                            joinColumns.add(column.fullyQualifiedName)
                        }
                    })
                }

                // Subqueries in FROM
                val fromItem = selectBody.fromItem
                if (fromItem is SubSelect) {
                    visitSelect(fromItem.selectBody, whereColumns, joinColumns)
                }
            }
            is WithItem -> {
                visitSelect(selectBody.selectBody, whereColumns, joinColumns)
            }
            is SetOperationList -> {
                selectBody.selects.forEach {
                    visitSelect(it, whereColumns, joinColumns)
                }
            }
        }
    }
}
