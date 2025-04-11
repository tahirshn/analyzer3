import sqlparse
from sqlparse.sql import Identifier, Comparison, Parenthesis, Where
from sqlparse.tokens import Keyword, Punctuation
from typing import List, Dict, Any
import json

class SQLQueryAnalyzer:
    def __init__(self):
        self.queries = []
        self.current_query = None
        self.query_id = 0

    def analyze_sql_file(self, file_path: str) -> List[Dict[str, Any]]:
        """Analyze SQL file using sqlparse"""
        with open(file_path, 'r') as file:
            sql_content = file.read()
        
        # Parse SQL content into statements
        statements = sqlparse.parse(sql_content)
        
        for statement in statements:
            # Skip empty statements and non-SELECT statements
            if not statement.is_group or not self._is_select_statement(statement):
                continue
                
            self._analyze_statement(statement)
            
        return self.queries

    def _is_select_statement(self, statement) -> bool:
        """Check if the statement is a SELECT statement"""
        first_token = statement.token_first(skip_ws=True, skip_cm=True)
        return first_token and first_token.value.upper() == 'SELECT'

    def _analyze_statement(self, statement):
        """Analyze a single SQL statement"""
        self.query_id += 1
        self.current_query = {
            "query_id": self.query_id,
            "query_text": str(statement).strip(),
            "tables": [],
            "joins": [],
            "where_conditions": []
        }
        
        # Process the FROM clause
        self._extract_from_clause(statement)
        
        # Extract JOIN clauses
        self._extract_joins(statement)
        
        # Extract WHERE conditions
        self._extract_where_conditions(statement)
        
        self.queries.append(self.current_query)

    def _extract_from_clause(self, statement):
        """Extract tables from FROM clause"""
        from_seen = False
        for token in statement.tokens:
            if from_seen:
                if token.ttype is Keyword and token.value.upper() in ['WHERE', 'GROUP', 'HAVING', 'ORDER', 'LIMIT']:
                    break
                self._process_from_token(token)
            
            if token.ttype is Keyword and token.value.upper() == 'FROM':
                from_seen = True

    def _process_from_token(self, token):
        """Process tokens in FROM clause"""
        if isinstance(token, Identifier):
            table_name = token.get_real_name()
            alias = token.get_alias() or table_name
            self.current_query["tables"].append({
                "table": table_name,
                "alias": alias
            })

    def _extract_joins(self, statement):
        """Extract JOIN information from statement"""
        for token in statement.tokens:
            if isinstance(token, sqlparse.sql.TokenList):
                if token.ttype is Keyword and token.value.upper().endswith('JOIN'):
                    join_info = self._process_join_token(token)
                    if join_info:
                        self.current_query["joins"].append(join_info)
                # Recursively check nested tokens
                self._extract_joins(token)

    def _process_join_token(self, token) -> Dict[str, Any]:
        """Process a single JOIN token"""
        join_type = token.value.upper().replace('JOIN', '').strip() or 'INNER'
        
        # Get the join target (table being joined)
        join_idx = token.parent.tokens.index(token)
        if join_idx + 1 >= len(token.parent.tokens):
            return None
            
        join_target = token.parent.tokens[join_idx + 1]
        
        if not isinstance(join_target, Identifier):
            return None
            
        table_name = join_target.get_real_name()
        alias = join_target.get_alias() or table_name
        
        # Find ON condition
        on_condition = None
        on_clause = self._find_on_clause(token.parent, join_idx)
        if on_clause:
            on_condition = str(on_clause)
            columns = self._extract_columns_from_expression(on_clause)
        else:
            columns = []
        
        return {
            "type": join_type,
            "table": table_name,
            "alias": alias,
            "condition": on_condition,
            "columns": columns
        }

    def _find_on_clause(self, parent_token, start_idx):
        """Find ON clause following a JOIN"""
        for i in range(start_idx + 2, len(parent_token.tokens)):
            token = parent_token.tokens[i]
            if token.ttype is Keyword and token.value.upper() == 'ON':
                if i + 1 < len(parent_token.tokens):
                    return parent_token.tokens[i + 1]
            if token.ttype is Keyword and token.value.upper() in ['WHERE', 'GROUP', 'HAVING', 'ORDER', 'LIMIT']:
                break
        return None

    def _extract_where_conditions(self, statement):
        """Extract WHERE conditions from statement"""
        where_seen = False
        for token in statement.tokens:
            if where_seen:
                if isinstance(token, Where):
                    self._process_where_clause(token)
                break
                
            if token.ttype is Keyword and token.value.upper() == 'WHERE':
                where_seen = True

    def _process_where_clause(self, where_token):
        """Process WHERE clause and extract conditions"""
        conditions = self._split_where_conditions(where_token)
        
        for condition in conditions:
            columns = self._extract_columns_from_expression(condition)
            self.current_query["where_conditions"].append({
                "condition": str(condition).strip(),
                "columns": columns
            })

    def _split_where_conditions(self, where_token) -> List:
        """Split WHERE clause into individual conditions"""
        conditions = []
        current_condition = []
        
        for token in where_token.tokens:
            if token.is_whitespace:
                continue
                
            if (isinstance(token, sqlparse.sql.TokenList) and \
               (token.value.upper() in ['AND', 'OR']):
                if current_condition:
                    conditions.append(sqlparse.sql.TokenList(current_condition))
                    current_condition = []
                continue
                
            current_condition.append(token)
        
        if current_condition:
            conditions.append(sqlparse.sql.TokenList(current_condition))
            
        return conditions

    def _extract_columns_from_expression(self, expr) -> List[Dict[str, str]]:
        """Extract table.column references from an expression"""
        columns = []
        
        for token in expr.tokens:
            if isinstance(token, Identifier):
                # Handle table.column format
                if '.' in token.value:
                    parts = token.value.split('.')
                    if len(parts) >= 2:
                        columns.append({
                            "table": parts[0],
                            "column": parts[1]
                        })
            elif isinstance(token, Comparison):
                # Handle comparison operators
                left = token.left
                right = token.right
                columns.extend(self._extract_columns_from_expression(left))
                columns.extend(self._extract_columns_from_expression(right))
            elif isinstance(token, Parenthesis):
                # Handle expressions in parentheses
                columns.extend(self._extract_columns_from_expression(token))
            elif isinstance(token, sqlparse.sql.TokenList):
                # Recursively check nested tokens
                columns.extend(self._extract_columns_from_expression(token))
                
        return columns

    def print_analysis(self):
        """Print analysis results in readable format"""
        for query in self.queries:
            print(f"\n=== Query {query['query_id']} ===")
            print(f"Full query: {query['query_text']}\n")
            
            print("Tables:")
            for table in query['tables']:
                print(f"- {table['table']} (as {table['alias']})")
            
            print("\nJOINs:")
            for join in query['joins']:
                print(f"- {join['type']} JOIN {join['table']} (as {join['alias']})")
                print(f"  ON {join['condition']}")
                print("  Columns used:")
                for col in join['columns']:
                    print(f"    - {col['table']}.{col['column']}")
            
            print("\nWHERE Conditions:")
            for condition in query['where_conditions']:
                print(f"- {condition['condition']}")
                print("  Columns used:")
                for col in condition['columns']:
                    print(f"    - {col['table']}.{col['column']}")
            
            print("\n" + "="*50)

# Example usage
if __name__ == "__main__":
    analyzer = SQLQueryAnalyzer()
    
    # Analyze SQL file
    queries = analyzer.analyze_sql_file("sql_queries.sql")
    
    # Print results
    analyzer.print_analysis()
    
    # Save to JSON
    with open("sql_analysis.json", "w") as f:
        json.dump(queries, f, indent=2)
