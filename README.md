import re
import json
from typing import List, Dict, Any

class SQLQueryParser:
    def __init__(self):
        self.queries = []
        self.current_query = None
        self.query_buffer = []
        self.query_id = 0

    def parse_sql_file(self, file_path: str) -> List[Dict[str, Any]]:
        """Parse SQL file line by line and extract queries"""
        with open(file_path, 'r') as file:
            for line in file:
                line = line.strip()
                
                # Skip empty lines and comments
                if not line or line.startswith('--'):
                    continue
                    
                # Add line to buffer
                self.query_buffer.append(line)
                
                # Check if line ends a query (even without semicolon)
                if self._is_query_end(line):
                    self._process_query_buffer()
                    
        return self.queries

    def _is_query_end(self, line: str) -> bool:
        """Determine if the line likely ends a query"""
        # Common query-ending keywords (case insensitive)
        end_keywords = {
            'select', 'insert', 'update', 'delete', 'create', 'drop', 
            'alter', 'with', '--', '/*'
        }
        
        # If line ends with semicolon or starts a new query
        return (line.endswith(';') or 
                any(line.lower().startswith(kw) for kw in end_keywords))

    def _process_query_buffer(self):
        """Process the accumulated query lines"""
        if not self.query_buffer:
            return
            
        # Combine lines into a single query string
        query_text = ' '.join(self.query_buffer).strip()
        self.query_buffer = []
        
        # Remove trailing semicolon if exists
        if query_text.endswith(';'):
            query_text = query_text[:-1]
            
        # Only process SELECT queries (can be extended for others)
        if not query_text.lower().startswith('select'):
            return
            
        self.query_id += 1
        self.current_query = {
            "query_id": self.query_id,
            "query_text": query_text,
            "joins": [],
            "where_conditions": []
        }
        
        self._extract_joins(query_text)
        self._extract_where_conditions(query_text)
        
        self.queries.append(self.current_query)

    def _extract_joins(self, query: str):
        """Extract JOIN conditions and identify tables/columns"""
        # Find all JOIN clauses (case insensitive)
        join_matches = re.finditer(
            r'(?:INNER\s+|LEFT\s+|RIGHT\s+|FULL\s+)?JOIN\s+(\w+)(?:\s+AS\s+)?(\w+)?\s+ON\s+([^)]+)',
            query, 
            re.IGNORECASE
        )
        
        for match in join_matches:
            table = match.group(1)
            alias = match.group(2) if match.group(2) else table
            condition = match.group(3).strip()
            
            # Extract columns used in the JOIN condition
            columns = self._extract_columns_from_condition(condition)
            
            self.current_query["joins"].append({
                "table": table,
                "alias": alias,
                "condition": condition,
                "columns": columns
            })

    def _extract_where_conditions(self, query: str):
        """Extract WHERE conditions and identify tables/columns"""
        # Find WHERE clause (case insensitive)
        where_match = re.search(
            r'WHERE\s+(.*?)(?:\s+(?:GROUP\s+BY|HAVING|ORDER\s+BY|LIMIT|$)', 
            query, 
            re.IGNORECASE | re.DOTALL
        )
        
        if not where_match:
            return
            
        where_clause = where_match.group(1)
        
        # Split conditions (simple split by AND/OR)
        conditions = re.split(r'\s+(?:AND|OR)\s+', where_clause, flags=re.IGNORECASE)
        
        for condition in conditions:
            condition = condition.strip()
            if not condition:
                continue
                
            # Extract columns used in the condition
            columns = self._extract_columns_from_condition(condition)
            
            self.current_query["where_conditions"].append({
                "condition": condition,
                "columns": columns
            })

    def _extract_columns_from_condition(self, condition: str) -> List[Dict[str, str]]:
        """Extract table and column references from a condition"""
        # Pattern to match table.column references
        column_pattern = re.compile(r'(\w+)\.(\w+)')
        columns = []
        
        for match in column_pattern.finditer(condition):
            table = match.group(1)
            column = match.group(2)
            columns.append({
                "table": table,
                "column": column
            })
            
        return columns

    def print_analysis_results(self):
        """Print the analysis results in a readable format"""
        for query in self.queries:
            print(f"\n=== Query {query['query_id']} ===")
            print(f"Full query: {query['query_text']}\n")
            
            print("JOIN Conditions:")
            for join in query['joins']:
                print(f"- Table: {join['table']} (alias: {join['alias']})")
                print(f"  Condition: {join['condition']}")
                print("  Columns used:")
                for col in join['columns']:
                    print(f"    - {col['table']}.{col['column']}")
                print()
            
            print("WHERE Conditions:")
            for where in query['where_conditions']:
                print(f"- Condition: {where['condition']}")
                print("  Columns used:")
                for col in where['columns']:
                    print(f"    - {col['table']}.{col['column']}")
                print()

# Example usage
if __name__ == "__main__":
    parser = SQLQueryParser()
    
    # Parse the SQL file
    queries = parser.parse_sql_file("sql_queries.sql")  # Replace with your file path
    
    # Print analysis results
    parser.print_analysis_results()
    
    # Optionally save to JSON
    with open("sql_analysis.json", "w") as f:
        json.dump(queries, f, indent=2)
