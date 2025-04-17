import re
from dataclasses import dataclass
from typing import List, Dict, Optional

@dataclass
class QueryAnalysis:
    query: str
    tables: List[str]
    columns: List[str]
    potential_issues: List[str]
    execution_time: Optional[float] = None

class SqlPerformanceAnalyzer:
    def __init__(self):
        self.performance_patterns = {
            'no_index': r'WHERE\s+([\w\.]+)\s*[=<>!]',
            'full_table_scan': r'WHERE\s+([\w\.]+)\s+IS NOT NULL',
            'wildcard_select': r'SELECT\s+\*\s+FROM',
            'multiple_joins': r'(JOIN\s+.*?){3,}',
            'complex_subquery': r'\(SELECT\s+.*?\s+FROM\s+.*?\s+WHERE\s+.*?\)',
            'like_with_wildcard': r'LIKE\s+[\'"]%[^\'"]+[\'"]',
            'order_by_non_indexed': r'ORDER BY\s+([\w\.]+)(?:\s+DESC)?\s*$',
            'large_offset': r'OFFSET\s+\d{4,}',
            'cross_join': r'CROSS JOIN',
            'or_conditions': r'WHERE\s+.*?\sOR\s+.*?',
        }

    def analyze_sql_file(self, file_path: str) -> Dict[str, QueryAnalysis]:
        """Analyze SQL file for potential performance issues"""
        with open(file_path, 'r', encoding='utf-8') as f:
            content = f.read()
        
        queries = self._split_sql_queries(content)
        results = {}
        
        for query in queries:
            if not query.strip():
                continue
            
            analysis = self._analyze_query(query)
            if analysis.potential_issues:
                results[hash(query)] = analysis
                
        return results

    def _split_sql_queries(self, content: str) -> List[str]:
        """Split SQL content into individual queries"""
        # Split on semicolons that aren't inside strings or parentheses
        return re.split(r';(?=(?:[^\'"]*[\'"][^\'"]*[\'"])*[^\'"]*$)', content)

    def _analyze_query(self, query: str) -> QueryAnalysis:
        """Analyze a single SQL query for performance issues"""
        normalized_query = self._normalize_query(query)
        tables = self._extract_tables(normalized_query)
        columns = self._extract_columns(normalized_query)
        
        analysis = QueryAnalysis(
            query=query,
            tables=tables,
            columns=columns,
            potential_issues=[]
        )
        
        self._detect_performance_issues(normalized_query, analysis)
        return analysis

    def _normalize_query(self, query: str) -> str:
        """Normalize SQL query for analysis"""
        # Remove comments
        query = re.sub(r'--.*?$', '', query, flags=re.MULTILINE)
        query = re.sub(r'/\*.*?\*/', '', query, flags=re.DOTALL)
        
        # Standardize whitespace
        query = ' '.join(query.split())
        query = query.upper()
        
        return query

    def _extract_tables(self, query: str) -> List[str]:
        """Extract table names from SQL query"""
        # FROM clause tables
        from_tables = re.findall(r'FROM\s+([\w]+)(?:\s+AS\s+[\w]+)?', query)
        
        # JOIN clause tables
        join_tables = re.findall(r'JOIN\s+([\w]+)(?:\s+AS\s+[\w]+)?', query)
        
        # INSERT/UPDATE tables
        dml_tables = re.findall(r'(?:INSERT\s+INTO|UPDATE)\s+([\w]+)', query)
        
        # Combine and deduplicate
        tables = list(set(from_tables + join_tables + dml_tables))
        return [t for t in tables if t]

    def _extract_columns(self, query: str) -> List[str]:
        """Extract column names from SQL query"""
        # WHERE clause columns
        where_columns = re.findall(r'WHERE\s+([\w\.]+)\s*[=<>!]', query)
        
        # SELECT columns (excluding *)
        select_columns = re.findall(r'SELECT\s+([\w\.]+)(?:\s*,\s*([\w\.]+))*', query)
        select_columns = [col for group in select_columns for col in group if col]
        
        # JOIN conditions
        join_columns = re.findall(r'ON\s+([\w\.]+)\s*=\s*[\w\.]+', query)
        
        # Combine and deduplicate
        columns = list(set(where_columns + select_columns + join_columns))
        return [col.split('.')[-1] for col in columns if col]

    def _detect_performance_issues(self, query: str, analysis: QueryAnalysis):
        """Detect potential performance issues in the query"""
        for issue_type, pattern in self.performance_patterns.items():
            matches = re.findall(pattern, query)
            if matches:
                if issue_type == 'no_index':
                    columns = [m.split('.')[-1] if '.' in m else m for m in matches]
                    analysis.potential_issues.append(
                        f"Potential missing index on columns: {', '.join(columns)}"
                    )
                elif issue_type == 'full_table_scan':
                    columns = [m.split('.')[-1] if '.' in m else m for m in matches]
                    analysis.potential_issues.append(
                        f"Full table scan possible on: {', '.join(columns)}"
                    )
                elif issue_type == 'wildcard_select':
                    analysis.potential_issues.append("SELECT * - fetching all columns")
                elif issue_type == 'multiple_joins':
                    analysis.potential_issues.append(f"Multiple joins ({len(matches)} tables)")
                elif issue_type == 'complex_subquery':
                    analysis.potential_issues.append("Complex subquery detected")
                elif issue_type == 'like_with_wildcard':
                    analysis.potential_issues.append("LIKE with leading wildcard (%)")
                elif issue_type == 'order_by_non_indexed':
                    columns = [m.split('.')[-1] if '.' in m else m for m in matches]
                    analysis.potential_issues.append(
                        f"ORDER BY non-indexed column: {', '.join(columns)}"
                    )
                elif issue_type == 'large_offset':
                    analysis.potential_issues.append("Large OFFSET value (pagination issue)")
                elif issue_type == 'cross_join':
                    analysis.potential_issues.append("CROSS JOIN (cartesian product)")
                elif issue_type == 'or_conditions':
                    analysis.potential_issues.append("OR conditions preventing index usage")

    def generate_report(self, analysis_results: Dict[str, QueryAnalysis]):
        """Generate a performance analysis report"""
        print("SQL Performance Analysis Report\n")
        print("="*80)
        
        for query_hash, analysis in analysis_results.items():
            print(f"\nQuery (abbreviated): {analysis.query[:100]}...")
            print(f"\nTables: {', '.join(analysis.tables)}")
            print(f"Columns: {', '.join(analysis.columns)}")
            print("\nPotential Performance Issues:")
            for issue in analysis.potential_issues:
                print(f"- {issue}")
            print("\n" + "-"*80)

# Example usage
if __name__ == "__main__":
    analyzer = SqlPerformanceAnalyzer()
    
    # Replace with your SQL file path
    analysis_results = analyzer.analyze_sql_file("sql_queries.sql")
    
    analyzer.generate_report(analysis_results)
