import re
from dataclasses import dataclass
from typing import List, Optional

@dataclass
class WhereCondition:
    """Represents a condition in a WHERE clause"""
    table: Optional[str]
    column: str
    operator: str
    value: str

class SqlWhereAnalyzer:
    def __init__(self):
        # Regex pattern to identify WHERE clauses and their conditions
        self.where_pattern = re.compile(
            r'WHERE\s+(.*?)(?=(?:\s+GROUP BY|\s+HAVING|\s+ORDER BY|\s+LIMIT|;|$))',
            re.IGNORECASE | re.DOTALL
        )
        self.condition_pattern = re.compile(
            r'(["\w.]+)\s*(=|!=|<>|<|>|<=|>=|LIKE|ILIKE|IN|NOT IN|BETWEEN|IS|IS NOT)\s*((?:\?.+?|["\w.]+(?:\([^)]*\))?)(?:\s+(?:AND|OR))?)',
            re.IGNORECASE
        )

    def analyze_sql_file(self, file_path: str) -> Dict[str, List[WhereCondition]]:
        """Analyze a SQL file and extract WHERE conditions"""
        with open(file_path, 'r', encoding='utf-8') as f:
            content = f.read()
        
        # Split into individual queries
        queries = self._split_sql_queries(content)
        results = {}
        
        for query in queries:
            if not query.strip():
                continue
            
            query_id = self._generate_query_id(query)
            conditions = self._extract_where_conditions(query)
            results[query_id] = conditions
            
        return results

    def _split_sql_queries(self, content: str) -> List[str]:
        """Split SQL content into individual queries"""
        # Split on semicolons that aren't inside strings or parentheses
        return re.split(r';(?=(?:[^"\']|"[^"]*"|\'[^\']*\')*$)', content)

    def _generate_query_id(self, query: str) -> str:
        """Generate a simple identifier for the query"""
        return f"query_{abs(hash(query.strip()))}"

    def _extract_where_conditions(self, query: str) -> List[WhereCondition]:
        """Extract conditions from WHERE clause"""
        where_match = self.where_pattern.search(query)
        if not where_match:
            return []
        
        where_clause = where_match.group(1)
        conditions = []
        
        # Split conditions by AND/OR but preserve nested parentheses
        condition_strings = self._split_conditions(where_clause)
        
        for cond_str in condition_strings:
            cond = self._parse_condition(cond_str)
            if cond:
                conditions.append(cond)
        
        return conditions

    def _split_conditions(self, where_clause: str) -> List[str]:
        """Split WHERE clause into individual conditions"""
        conditions = []
        current = []
        paren_level = 0
        
        tokens = re.findall(r'(\bAND\b|\bOR\b|\(|\)|[^()]+)', where_clause, re.IGNORECASE)
        
        for token in tokens:
            token = token.strip()
            if not token:
                continue
                
            if token.upper() in ('AND', 'OR'):
                if paren_level == 0 and current:
                    conditions.append(' '.join(current))
                    current = []
                continue
                
            if token == '(':
                paren_level += 1
            elif token == ')':
                paren_level -= 1
                
            current.append(token)
        
        if current:
            conditions.append(' '.join(current))
        
        return conditions

    def _parse_condition(self, condition_str: str) -> Optional[WhereCondition]:
        """Parse a single condition string"""
        # Clean up the condition string
        condition_str = condition_str.strip()
        if not condition_str:
            return None
            
        # Handle nested parentheses
        if condition_str.startswith('(') and condition_str.endswith(')'):
            condition_str = condition_str[1:-1].strip()
            
        # Match the condition pattern
        match = self.condition_pattern.match(condition_str)
        if not match:
            return None
            
        left, operator, right = match.groups()
        
        # Parse table and column
        table, column = self._parse_column_reference(left)
        
        # Clean up the right side value
        right = right.split()[0]  # Take only the first part before AND/OR if present
        right = right.strip('"\'')  # Remove string quotes
        
        return WhereCondition(
            table=table,
            column=column,
            operator=operator.upper(),
            value=right
        )

    def _parse_column_reference(self, reference: str) -> Tuple[Optional[str], str]:
        """Parse a column reference into table and column parts"""
        parts = reference.split('.')
        if len(parts) == 2:
            return parts[0], parts[1]
        return None, reference

def print_where_analysis(results: Dict[str, List[WhereCondition]]):
    """Print the analysis results"""
    for query_id, conditions in results.items():
        print(f"\n=== Query: {query_id} ===")
        
        if not conditions:
            print("No WHERE conditions found")
            continue
            
        print("WHERE Conditions:")
        for cond in conditions:
            col_ref = f"{cond.table}.{cond.column}" if cond.table else cond.column
            print(f"  {col_ref} {cond.operator} {cond.value}")

if __name__ == "__main__":
    analyzer = SqlWhereAnalyzer()
    
    # Example SQL file with WHERE clauses
    sample_sql = """
    SELECT * FROM users WHERE id = 1 AND status = 'active';
    UPDATE products SET price = 9.99 WHERE category = 'electronics' AND stock > 0;
    DELETE FROM orders WHERE created_at < '2023-01-01' OR customer_id IS NULL;
    SELECT * FROM accounts WHERE balance BETWEEN 100 AND 1000;
    """
    
    # Write sample SQL to temporary file
    with open("temp_queries.sql", "w", encoding='utf-8') as f:
        f.write(sample_sql)
    
    # Analyze the file
    analysis_results = analyzer.analyze_sql_file("temp_queries.sql")
    
    # Print results
    print_where_analysis(analysis_results)
