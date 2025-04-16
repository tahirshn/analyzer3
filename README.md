import re
from dataclasses import dataclass
from typing import List, Dict, Optional, Set
import sqlparse
from sqlparse.sql import Identifier, Comparison, Parenthesis, Where, IdentifierList
from sqlparse.tokens import Keyword, Punctuation

@dataclass
class TableReference:
    name: str
    alias: Optional[str] = None

@dataclass
class ColumnReference:
    name: str
    table: Optional[str] = None

@dataclass
class JoinCondition:
    left_column: ColumnReference
    right_column: ColumnReference
    operator: str

@dataclass
class WhereCondition:
    column: ColumnReference
    operator: str
    value: str

@dataclass
class QueryAnalysis:
    tables: Set[TableReference]
    joins: List[JoinCondition]
    where_conditions: List[WhereCondition]
    selected_columns: List[ColumnReference]

class SqlQueryAnalyzer:
    def __init__(self):
        self.current_query = None

    def analyze_sql_file(self, file_path: str) -> Dict[str, QueryAnalysis]:
        """Analyze all SQL queries in a file"""
        with open(file_path, 'r') as f:
            content = f.read()
        
        queries = sqlparse.split(content)
        results = {}
        
        for query in queries:
            if not query.strip():
                continue
            
            parsed = sqlparse.parse(query)[0]
            query_id = self._generate_query_id(query)
            results[query_id] = self._analyze_query(parsed)
            
        return results

    def _generate_query_id(self, query: str) -> str:
        """Generate a simple identifier for the query"""
        return f"query_{hash(query.strip())}"

    def _analyze_query(self, parsed_query) -> QueryAnalysis:
        """Analyze a single SQL query"""
        self.current_query = parsed_query
        analysis = QueryAnalysis(
            tables=set(),
            joins=[],
            where_conditions=[],
            selected_columns=[]
        )

        # Extract tables (FROM and JOIN clauses)
        self._extract_tables(parsed_query, analysis)
        
        # Extract JOIN conditions
        self._extract_joins(parsed_query, analysis)
        
        # Extract WHERE conditions
        self._extract_where_conditions(parsed_query, analysis)
        
        # Extract selected columns (SELECT clause)
        self._extract_selected_columns(parsed_query, analysis)
        
        return analysis

    def _extract_tables(self, parsed_query, analysis: QueryAnalysis):
        """Extract table references from FROM and JOIN clauses"""
        from_seen = False
        for token in parsed_query.tokens:
            if token.is_keyword and token.value.upper() == 'FROM':
                from_seen = True
                continue
                
            if from_seen:
                if token.is_group:
                    for identifier in token.get_identifiers():
                        table_ref = self._parse_table_reference(identifier)
                        if table_ref:
                            analysis.tables.add(table_ref)
                from_seen = False

            if token.is_keyword and token.value.upper() == 'JOIN':
                if len(parsed_query.tokens) > parsed_query.token_index(token) + 1:
                    next_token = parsed_query.tokens[parsed_query.token_index(token) + 1]
                    table_ref = self._parse_table_reference(next_token)
                    if table_ref:
                        analysis.tables.add(table_ref)

    def _parse_table_reference(self, token) -> Optional[TableReference]:
        """Parse a table reference with optional alias"""
        if isinstance(token, Identifier):
            parts = [p.value for p in token.flatten() if not p.is_whitespace]
            if len(parts) == 1:
                return TableReference(name=parts[0])
            elif len(parts) == 3 and parts[1].upper() == 'AS':
                return TableReference(name=parts[0], alias=parts[2])
            elif len(parts) == 2:
                return TableReference(name=parts[0], alias=parts[1])
        return None

    def _extract_joins(self, parsed_query, analysis: QueryAnalysis):
        """Extract JOIN conditions"""
        for token in parsed_query.tokens:
            if isinstance(token, Parenthesis):
                self._extract_joins(token, analysis)
                
            if token.is_keyword and token.value.upper() == 'ON':
                # Get the next token which should be the join condition
                idx = parsed_query.token_index(token)
                if idx + 1 < len(parsed_query.tokens):
                    condition = parsed_query.tokens[idx + 1]
                    if isinstance(condition, Comparison):
                        left, op, right = condition.left, condition.token_next(condition.left)[1], condition.right
                        left_col = self._parse_column_reference(left)
                        right_col = self._parse_column_reference(right)
                        if left_col and right_col:
                            analysis.joins.append(JoinCondition(
                                left_column=left_col,
                                right_column=right_col,
                                operator=op.value
                            ))

    def _extract_where_conditions(self, parsed_query, analysis: QueryAnalysis):
        """Extract WHERE conditions"""
        where_clause = None
        for token in parsed_query.tokens:
            if isinstance(token, Where):
                where_clause = token
                break
                
        if where_clause:
            for condition in where_clause.get_sublists():
                if isinstance(condition, Comparison):
                    col = self._parse_column_reference(condition.left)
                    op = condition.token_next(condition.left)[1].value
                    value = condition.right.value
                    if col:
                        analysis.where_conditions.append(WhereCondition(
                            column=col,
                            operator=op,
                            value=value
                        ))

    def _parse_column_reference(self, token) -> Optional[ColumnReference]:
        """Parse a column reference with optional table qualifier"""
        if not hasattr(token, 'tokens'):
            return None
            
        parts = [t.value for t in token.tokens if not t.is_whitespace and t.value != '.']
        
        if len(parts) == 1:
            return ColumnReference(name=parts[0])
        elif len(parts) == 2:
            return ColumnReference(table=parts[0], name=parts[1])
        return None

    def _extract_selected_columns(self, parsed_query, analysis: QueryAnalysis):
        """Extract selected columns from SELECT clause"""
        select_seen = False
        for token in parsed_query.tokens:
            if token.is_keyword and token.value.upper() == 'SELECT':
                select_seen = True
                continue
                
            if select_seen:
                if token.is_keyword and token.value.upper() in ('FROM', 'WHERE', 'GROUP BY', 'HAVING', 'ORDER BY', 'LIMIT'):
                    break
                    
                if isinstance(token, IdentifierList):
                    for identifier in token.get_identifiers():
                        col = self._parse_column_reference(identifier)
                        if col:
                            analysis.selected_columns.append(col)
                elif isinstance(token, Identifier):
                    col = self._parse_column_reference(token)
                    if col:
                        analysis.selected_columns.append(col)

def print_analysis_results(analysis_results: Dict[str, QueryAnalysis]):
    """Print the analysis results in a readable format"""
    for query_id, analysis in analysis_results.items():
        print(f"\n=== Analysis for {query_id} ===")
        
        print("\nTables:")
        for table in analysis.tables:
            print(f"- {table.name}" + (f" (as {table.alias})" if table.alias else ""))
        
        print("\nJoins:")
        for join in analysis.joins:
            left = f"{join.left_column.table}.{join.left_column.name}" if join.left_column.table else join.left_column.name
            right = f"{join.right_column.table}.{join.right_column.name}" if join.right_column.table else join.right_column.name
            print(f"- {left} {join.operator} {right}")
        
        print("\nWHERE Conditions:")
        for cond in analysis.where_conditions:
            col = f"{cond.column.table}.{cond.column.name}" if cond.column.table else cond.column.name
            print(f"- {col} {cond.operator} {cond.value}")
        
        print("\nSelected Columns:")
        for col in analysis.selected_columns:
            col_ref = f"{col.table}.{col.name}" if col.table else col.name
            print(f"- {col_ref}")

if __name__ == "__main__":
    analyzer = SqlQueryAnalyzer()
    
    # Example usage - replace with your SQL file path
    analysis_results = analyzer.analyze_sql_file("queries.sql")
    
    print_analysis_results(analysis_results)
