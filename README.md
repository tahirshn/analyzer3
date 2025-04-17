import re
from dataclasses import dataclass, field
from typing import List, Dict, Optional, Set
from collections import defaultdict

@dataclass
class TableInfo:
    name: str
    alias: Optional[str] = None
    columns: Set[str] = field(default_factory=set)

@dataclass
class QueryAnalysis:
    raw_query: str
    tables: Dict[str, TableInfo]
    potential_issues: List[str]
    join_conditions: List[Dict[str, str]] = field(default_factory=list)
    where_conditions: List[Dict[str, str]] = field(default_factory=list)

class AdvancedSqlAnalyzer:
    def __init__(self):
        self.performance_patterns = {
            'no_index': r'WHERE\s+([\w\.]+)\s*[=<>!]',
            'full_scan': r'WHERE\s+([\w\.]+)\s+IS NOT NULL|\b([\w\.]+)\s+LIKE\s+\'%',
            'wildcard_select': r'SELECT\s+\*',
            'multiple_joins': r'(JOIN\s+\w+\s+ON\s+.*?){3,}',
            'complex_subquery': r'\(SELECT\s+.*?\s+FROM\s+.*?\s+WHERE\s+.*?\)',
            'large_offset': r'OFFSET\s+\d{4,}',
            'cross_join': r'CROSS\s+JOIN',
            'or_conditions': r'WHERE\s+.*?\sOR\s+.*?',
            'order_by_no_index': r'ORDER\s+BY\s+([\w\.]+)(?:\s+DESC)?\s*(?:LIMIT|$)'
        }

    def analyze_file(self, file_path: str) -> Dict[str, QueryAnalysis]:
        with open(file_path, 'r', encoding='utf-8') as f:
            content = f.read()
        
        queries = self._split_queries(content)
        results = {}
        
        for query in queries:
            if not query.strip():
                continue
            
            normalized = self._normalize_query(query)
            analysis = self._analyze_query(normalized, query)
            results[hash(query)] = analysis
            
        return results

    def _split_queries(self, content: str) -> List[str]:
        return [q.strip() for q in re.split(r';(?=(?:[^\'"]*[\'"][^\'"]*[\'"])*[^\'"]*$)', content) if q.strip()]

    def _normalize_query(self, query: str) -> str:
        query = re.sub(r'--.*?$', '', query, flags=re.MULTILINE)
        query = re.sub(r'/\*.*?\*/', '', query, flags=re.DOTALL)
        return ' '.join(query.split()).upper()

    def _analyze_query(self, normalized_query: str, original_query: str) -> QueryAnalysis:
        tables = self._extract_tables_with_aliases(normalized_query)
        self._extract_columns_for_tables(normalized_query, tables)
        
        join_conditions = self._extract_join_conditions(normalized_query)
        where_conditions = self._extract_where_conditions(normalized_query)
        
        analysis = QueryAnalysis(
            raw_query=original_query,
            tables=tables,
            potential_issues=[],
            join_conditions=join_conditions,
            where_conditions=where_conditions
        )
        
        self._detect_issues(normalized_query, analysis)
        return analysis

    def _extract_tables_with_aliases(self, query: str) -> Dict[str, TableInfo]:
        tables = {}
        
        # FROM clause
        from_matches = re.finditer(r'FROM\s+(\w+)(?:\s+(?:AS\s+)?(\w+))?', query)
        for match in from_matches:
            table_name, alias = match.groups()
            tables[alias or table_name] = TableInfo(name=table_name, alias=alias)
        
        # JOIN clauses
        join_matches = re.finditer(r'JOIN\s+(\w+)(?:\s+(?:AS\s+)?(\w+))?', query)
        for match in join_matches:
            table_name, alias = match.groups()
            tables[alias or table_name] = TableInfo(name=table_name, alias=alias)
        
        return tables

    def _extract_columns_for_tables(self, query: str, tables: Dict[str, TableInfo]):
        # SELECT columns
        select_matches = re.finditer(r'SELECT\s+(.*?)(?=\s+FROM)', query)
        for match in select_matches:
            columns = re.findall(r'([\w]+)\.([\w]+)|([\w]+)', match.group(1))
            for table_col, col, simple_col in columns:
                if table_col:
                    if table_col in tables:
                        tables[table_col].columns.add(col)
                elif simple_col and simple_col != '*':
                    for table in tables.values():
                        table.columns.add(simple_col)
        
        # WHERE conditions
        where_matches = re.finditer(r'WHERE\s+(.*?)(?=\s+(?:GROUP BY|HAVING|ORDER BY|LIMIT|$))', query)
        for match in where_matches:
            conditions = re.findall(r'([\w]+)\.([\w]+)|([\w]+)', match.group(1))
            for table_col, col, simple_col in conditions:
                if table_col:
                    if table_col in tables:
                        tables[table_col].columns.add(col)
                elif simple_col:
                    for table in tables.values():
                        table.columns.add(simple_col)

    def _extract_join_conditions(self, query: str) -> List[Dict[str, str]]:
        joins = []
        join_matches = re.finditer(r'JOIN\s+\w+(?:\s+\w+)?\s+ON\s+(.*?)(?=\s+(?:WHERE|JOIN|GROUP BY|$))', query)
        for match in join_matches:
            conditions = re.findall(r'([\w]+)\.([\w]+)\s*=\s*([\w]+)\.([\w]+)', match.group(1))
            for left_table, left_col, right_table, right_col in conditions:
                joins.append({
                    'left_table': left_table,
                    'left_column': left_col,
                    'right_table': right_table,
                    'right_column': right_col
                })
        return joins

    def _extract_where_conditions(self, query: str) -> List[Dict[str, str]]:
        conditions = []
        where_matches = re.finditer(r'WHERE\s+(.*?)(?=\s+(?:GROUP BY|HAVING|ORDER BY|LIMIT|$))', query)
        for match in where_matches:
            clauses = re.split(r'\s+AND\s+|\s+OR\s+', match.group(1))
            for clause in clauses:
                col_match = re.match(r'([\w]+)\.([\w]+)|([\w]+)', clause)
                if col_match:
                    table, col, simple_col = col_match.groups()
                    conditions.append({
                        'table': table or '',
                        'column': col or simple_col,
                        'condition': clause
                    })
        return conditions

    def _detect_issues(self, query: str, analysis: QueryAnalysis):
        for issue_type, pattern in self.performance_patterns.items():
            matches = re.finditer(pattern, query)
            for match in matches:
                if issue_type == 'no_index':
                    cols = [g for g in match.groups() if g]
                    for col in cols:
                        if '.' in col:
                            table, col = col.split('.')
                            if table in analysis.tables:
                                analysis.potential_issues.append(
                                    f"Missing index on {table}.{col} (WHERE condition)"
                                )
                
                elif issue_type == 'full_scan':
                    cols = [g for g in match.groups() if g]
                    for col in cols:
                        if '.' in col:
                            table, col = col.split('.')
                            if table in analysis.tables:
                                analysis.potential_issues.append(
                                    f"Full table scan possible on {table}.{col}"
                                )
                
                elif issue_type == 'wildcard_select':
                    analysis.potential_issues.append("SELECT * used - fetching all columns")
                
                elif issue_type == 'multiple_joins':
                    join_count = len(analysis.join_conditions)
                    if join_count >= 3:
                        analysis.potential_issues.append(f"Multiple joins ({join_count} tables)")
                
                elif issue_type == 'order_by_no_index':
                    cols = [g for g in match.groups() if g]
                    for col in cols:
                        if '.' in col:
                            table, col = col.split('.')
                            if table in analysis.tables:
                                analysis.potential_issues.append(
                                    f"ORDER BY on non-indexed column {table}.{col}"
                                )

    def generate_detailed_report(self, analysis_results: Dict[str, QueryAnalysis]):
        print("SQL Performance Analysis - Detailed Report")
        print("="*90)
        
        for q_hash, analysis in analysis_results.items():
            print(f"\nQuery:\n{analysis.raw_query[:200]}...\n")
            
            print("Tables and Columns:")
            for table_alias, table_info in analysis.tables.items():
                cols = ', '.join(table_info.columns) if table_info.columns else 'N/A'
                print(f"- {table_info.name} ({f'as {table_alias}' if table_alias != table_info.name else 'no alias'})")
                print(f"  Columns: {cols}")
            
            if analysis.join_conditions:
                print("\nJoin Conditions:")
                for join in analysis.join_conditions:
                    print(f"- {join['left_table']}.{join['left_column']} = {join['right_table']}.{join['right_column']}")
            
            if analysis.where_conditions:
                print("\nWHERE Conditions:")
                for cond in analysis.where_conditions:
                    table = f"{cond['table']}." if cond['table'] else ''
                    print(f"- {table}{cond['column']} in: {cond['condition'][:50]}...")
            
            if analysis.potential_issues:
                print("\nPotential Performance Issues:")
                for issue in analysis.potential_issues:
                    print(f"- {issue}")
            
            print("\n" + "-"*90)

if __name__ == "__main__":
    analyzer = AdvancedSqlAnalyzer()
    results = analyzer.analyze_file("sql_queries.sql")
    analyzer.generate_detailed_report(results)
