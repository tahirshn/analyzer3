import re
from dataclasses import dataclass
from typing import List, Dict, Optional, Set, Tuple
import sqlparse
from sqlparse.sql import Identifier, Comparison, Parenthesis, Where, IdentifierList, Function
from sqlparse.tokens import Keyword, Punctuation, Name

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
    errors: List[str]

class SqlQueryAnalyzer:
    def __init__(self):
        self.current_query = None

    def analyze_sql_file(self, file_path: str) -> Dict[str, QueryAnalysis]:
        """Analyze all SQL queries in a file with better error handling"""
        try:
            with open(file_path, 'r', encoding='utf-8') as f:
                content = f.read()
        except Exception as e:
            return {"error": f"File reading error: {str(e)}"}
        
        # Split content into individual queries more reliably
        query_strings = self._split_sql_queries(content)
        results = {}
        
        for i, query in enumerate(query_strings, 1):
            if not query.strip():
                continue
            
            query_id = f"query_{i}"
            try:
                parsed = sqlparse.parse(query)
                if not parsed:
                    results[query_id] = QueryAnalysis(
                        tables=set(),
                        joins=[],
                        where_conditions=[],
                        selected_columns=[],
                        errors=["Empty query after parsing"]
                    )
                    continue
                
                results[query_id] = self._analyze_query(parsed[0])
            except Exception as e:
                results[query_id] = QueryAnalysis(
                    tables=set(),
                    joins=[],
                    where_conditions=[],
                    selected_columns=[],
                    errors=[f"Parsing error: {str(e)}"]
                )
            
        return results

    def _split_sql_queries(self, content: str) -> List[str]:
        """More reliable SQL query splitting"""
        # Split on semicolons that aren't inside strings or comments
        queries = []
        current_query = []
        in_string = False
        string_char = None
        in_comment = False
        
        for line in content.split('\n'):
            line = line.strip()
            if not line:
                continue
                
            if in_comment:
                if '*/' in line:
                    line = line.split('*/', 1)[1]
                    in_comment = False
                else:
                    continue
            
            if '/*' in line and '*/' not in line:
                in_comment = True
                line = line.split('/*', 1)[0]
            
            if '--' in line:
                line = line.split('--', 1)[0]
                
            for char in line:
                if char in ('"', "'") and not in_comment:
                    if in_string and char == string_char:
                        in_string = False
                        string_char = None
                    elif not in_string:
                        in_string = True
                        string_char = char
                
                current_query.append(char)
                
                if char == ';' and not in_string and not in_comment:
                    queries.append(''.join(current_query).strip())
                    current_query = []
        
        if current_query:
            queries.append(''.join(current_query).strip())
            
        return [q for q in queries if q]

    def _analyze_query(self, parsed_query) -> QueryAnalysis:
        """Analyze a single SQL query with better error handling"""
        self.current_query = parsed_query
        analysis = QueryAnalysis(
            tables=set(),
            joins=[],
            where_conditions=[],
            selected_columns=[],
            errors=[]
        )

        try:
            # Extract tables (FROM and JOIN clauses)
            self._extract_tables(parsed_query, analysis)
            
            # Extract JOIN conditions
            self._extract_joins(parsed_query, analysis)
            
            # Extract WHERE conditions
            self._extract_where_conditions(parsed_query, analysis)
            
            # Extract selected columns (SELECT clause)
            self._extract_selected_columns(parsed_query, analysis)
            
        except Exception as e:
            analysis.errors.append(f"Analysis error: {str(e)}")
        
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

            if token.is_keyword and token.value.upper() in ('JOIN', 'INNER JOIN', 'LEFT JOIN', 
                                                          'RIGHT JOIN', 'FULL JOIN', 'CROSS JOIN'):
                if len(parsed_query.tokens) > parsed_query.token_index(token) + 1:
                    next_token = parsed_query.tokens[parsed_query.token_index(token) + 1]
                    table_ref = self._parse_table_reference(next_token)
                    if table_ref:
                        analysis.tables.add(table_ref)

    def _parse_table_reference(self, token) -> Optional[TableReference]:
        """Parse a table reference with optional alias"""
        if isinstance(token, (Identifier, Function)):
            # Handle cases like "schema.table" or "table AS alias"
            parts = []
            for t in token.flatten():
                if t.is_whitespace or t.value == '.':
                    continue
                if t.ttype == Name or t.is_keyword:
                    parts.append(t.value)
            
            if len(parts) == 1:
                return TableReference(name=parts[0])
            elif len(parts) >= 2 and parts[-2].upper() == 'AS':
                return TableReference(name='.'.join(parts[:-2]), alias=parts[-1])
            elif len(parts) == 2 and parts[-2].upper() != 'AS':
                # Could be schema.table or table alias
                if '.' in parts[0]:
                    return TableReference(name=parts[0], alias=parts[1])
                else:
                    return TableReference(name=parts[0], alias=parts[1])
            elif len(parts) > 2:
                # Handle schema.table AS alias
                return TableReference(name='.'.join(parts[:-2]), alias=parts[-1])
        return None

    def _extract_joins(self, parsed_query, analysis: QueryAnalysis):
        """Extract JOIN conditions with better handling"""
        for i, token in enumerate(parsed_query.tokens):
            if isinstance(token, Parenthesis):
                self._extract_joins(token, analysis)
                
            if token.is_keyword and token.value.upper() == 'ON':
                # Get the next token which should be the join condition
                if i + 1 < len(parsed_query.tokens):
                    condition = parsed_query.tokens[i + 1]
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
                    elif condition.is_group:
                        # Handle complex join conditions
                        for sub_token in condition.tokens:
                            if isinstance(sub_token, Comparison):
                                left, op, right = sub_token.left, sub_token.token_next(sub_token.left)[1], sub_token.right
                                left_col = self._parse_column_reference(left)
                                right_col = self._parse_column_reference(right)
                                if left_col and right_col:
                                    analysis.joins.append(JoinCondition(
                                        left_column=left_col,
                                        right_column=right_col,
                                        operator=op.value
                                    ))

    def _parse_column_reference(self, token) -> Optional[ColumnReference]:
        """Parse a column reference with optional table qualifier"""
        if isinstance(token, Identifier):
            parts = []
            for t in token.flatten():
                if t.is_whitespace or t.value == '.':
                    continue
                if t.ttype == Name or t.is_keyword:
                    parts.append(t.value)
            
            if len(parts) == 1:
                return ColumnReference(name=parts[0])
            elif len(parts) == 2:
                return ColumnReference(table=parts[0], name=parts[1])
            elif len(parts) == 3:  # schema.table.column
                return ColumnReference(table=f"{parts[0]}.{parts[1]}", name=parts[2])
        return None

    def _extract_where_conditions(self, parsed_query, analysis: QueryAnalysis):
        """Extract WHERE conditions with better handling"""
        where_clause = None
        for token in parsed_query.tokens:
            if isinstance(token, Where):
                where_clause = token
                break
                
        if where_clause:
            for condition in where_clause.get_sublists():
                if isinstance(condition, Comparison):
                    col = self._parse_column_reference(condition.left)
                    op_token = condition.token_next(condition.left)
                    op = op_token[1].value if op_token else "="
                    value = condition.right.value if hasattr(condition, 'right') else "?"
                    if col:
                        analysis.where_conditions.append(WhereCondition(
                            column=col,
                            operator=op,
                            value=value
                        ))

    def _extract_selected_columns(self, parsed_query, analysis: QueryAnalysis):
        """Extract selected columns from SELECT clause with better handling"""
        select_seen = False
        for token in parsed_query.tokens:
            if token.is_keyword and token.value.upper() == 'SELECT':
                select_seen = True
                continue
                
            if select_seen:
                if token.is_keyword and token.value.upper() in ('FROM', 'WHERE', 'GROUP BY', 
                                                             'HAVING', 'ORDER BY', 'LIMIT'):
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
                elif isinstance(token, Function):
                    # Handle function calls like COUNT(*)
                    analysis.selected_columns.append(ColumnReference(name=token.get_name()))

def print_analysis_results(analysis_results: Dict[str, QueryAnalysis]):
    """Print the analysis results in a readable format"""
    for query_id, analysis in analysis_results.items():
        print(f"\n=== Analysis for {query_id} ===")
        
        if analysis.errors:
            print("\nErrors:")
            for error in analysis.errors:
                print(f"- {error}")
            continue
        
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
