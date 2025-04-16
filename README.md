import re
from dataclasses import dataclass
from typing import List, Dict, Optional

@dataclass
class SqlParameter:
    table: str
    column: str
    parameter_index: int
    context: str  # WHERE/JOIN/INSERT etc.

class SqlParameterAnalyzer:
    def __init__(self):
        self.parameters: List[SqlParameter] = []
        self.current_param_index = 0

    def analyze_sql(self, sql: str) -> List[SqlParameter]:
        """Analyze SQL and extract parameterized values with context"""
        self.parameters = []
        self.current_param_index = 0
        
        # Normalize SQL for easier parsing
        normalized_sql = self._normalize_sql(sql)
        
        # Extract parameters from different clauses
        self._extract_where_parameters(normalized_sql)
        self._extract_join_parameters(normalized_sql)
        self._extract_insert_parameters(normalized_sql)
        self._extract_update_parameters(normalized_sql)
        
        return self.parameters

    def _normalize_sql(self, sql: str) -> str:
        """Normalize SQL for consistent parsing"""
        # Remove comments
        sql = re.sub(r'--.*?$', '', sql, flags=re.MULTILINE)
        sql = re.sub(r'/\*.*?\*/', '', sql, flags=re.DOTALL)
        
        # Standardize whitespace
        sql = ' '.join(sql.split())
        
        # Handle IN clauses with multiple parameters
        sql = re.sub(r'\(\s*\?\s*(,\s*\?)+\s*\)', ' (?)', sql)
        
        return sql

    def _extract_where_parameters(self, sql: str):
        """Extract parameters from WHERE clause"""
        where_match = re.search(r'WHERE\s+(.*?)(?=\s+(GROUP BY|HAVING|ORDER BY|LIMIT|$)', sql, re.IGNORECASE)
        if where_match:
            where_clause = where_match.group(1)
            self._process_conditions(where_clause, 'WHERE')

    def _extract_join_parameters(self, sql: str):
        """Extract parameters from JOIN conditions"""
        join_matches = re.finditer(r'JOIN\s+.*?\s+ON\s+(.*?)(?=\s+(WHERE|GROUP BY|HAVING|ORDER BY|LIMIT|JOIN|$))', 
                                sql, re.IGNORECASE)
        for match in join_matches:
            join_conditions = match.group(1)
            self._process_conditions(join_conditions, 'JOIN')

    def _extract_insert_parameters(self, sql: str):
        """Extract parameters from INSERT VALUES clause"""
        insert_match = re.search(r'INSERT\s+INTO\s+(\w+)\s*\(.*?\)\s*VALUES\s*\(.*?\)', sql, re.IGNORECASE)
        if insert_match:
            table = insert_match.group(1)
            values_match = re.search(r'VALUES\s*\((.*?)\)', sql, re.IGNORECASE)
            if values_match:
                values = values_match.group(1)
                param_count = values.count('?')
                for i in range(param_count):
                    self.parameters.append(SqlParameter(
                        table=table,
                        column=f"column_{self.current_param_index + 1}",
                        parameter_index=self.current_param_index,
                        context='INSERT'
                    ))
                    self.current_param_index += 1

    def _extract_update_parameters(self, sql: str):
        """Extract parameters from UPDATE SET clause"""
        update_match = re.search(r'UPDATE\s+(\w+)\s+SET\s+(.*?)(?=\s+WHERE|$)', sql, re.IGNORECASE)
        if update_match:
            table = update_match.group(1)
            set_clause = update_match.group(2)
            set_items = [item.strip() for item in set_clause.split(',')]
            
            for item in set_items:
                if '?' in item:
                    column_match = re.match(r'([\w\.]+)\s*=', item)
                    if column_match:
                        column = column_match.group(1).split('.')[-1]  # Remove table prefix if exists
                        self.parameters.append(SqlParameter(
                            table=table,
                            column=column,
                            parameter_index=self.current_param_index,
                            context='UPDATE'
                        ))
                        self.current_param_index += 1

    def _process_conditions(self, conditions: str, context: str):
        """Process conditions and extract parameters"""
        # Split conditions by AND/OR but handle nested parentheses
        condition_parts = self._split_conditions(conditions)
        
        for part in condition_parts:
            if '?' in part:
                # Extract column name from condition
                column_match = re.search(r'([\w\.]+)\s*[=<>!]+\s*\?', part)
                if column_match:
                    full_column = column_match.group(1)
                    table, column = (full_column.split('.') + [None])[:2]
                    if column is None:
                        column = table
                        table = None  # Table name not specified
                    
                    self.parameters.append(SqlParameter(
                        table=table,
                        column=column,
                        parameter_index=self.current_param_index,
                        context=context
                    ))
                    self.current_param_index += 1

    def _split_conditions(self, conditions: str) -> List[str]:
        """Split conditions while handling nested parentheses"""
        parts = []
        current = []
        paren_level = 0
        
        for char in conditions:
            if char == '(':
                paren_level += 1
            elif char == ')':
                paren_level -= 1
            
            if char in (' ', '\t') and paren_level == 0:
                if current:
                    parts.append(''.join(current))
                    current = []
            else:
                current.append(char)
        
        if current:
            parts.append(''.join(current))
        
        # Further split by AND/OR at top level
        final_parts = []
        for part in parts:
            if part.upper() in ('AND', 'OR'):
                continue
            final_parts.append(part)
        
        return final_parts

# Example usage
if __name__ == "__main__":
    analyzer = SqlParameterAnalyzer()
    
    test_sql = """
    SELECT * FROM users WHERE id = ? AND status = ?;
    UPDATE products SET price = ? WHERE id = ?;
    INSERT INTO orders (user_id, product_id) VALUES (?, ?);
    SELECT u.name FROM users u JOIN orders o ON u.id = o.user_id WHERE o.date > ?;
    """
    
    parameters = analyzer.analyze_sql(test_sql)
    
    print("Parameter Analysis:")
    for param in parameters:
        print(f"Param #{param.parameter_index}: {param.table or 'unknown'}.{param.column} ({param.context})")
