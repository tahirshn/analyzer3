import re
from dataclasses import dataclass, field
from typing import List, Dict, Optional, Set, Tuple
from collections import defaultdict
import logging

# Log ayarları
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')
logger = logging.getLogger(__name__)

@dataclass
class TableInfo:
    name: str
    alias: Optional[str] = None
    columns: Set[str] = field(default_factory=set)
    is_derived: bool = False

@dataclass
class QueryAnalysis:
    raw_query: str
    normalized_query: str
    tables: Dict[str, TableInfo] = field(default_factory=dict)
    potential_issues: List[str] = field(default_factory=list)
    join_conditions: List[Dict[str, str]] = field(default_factory=list)
    where_conditions: List[Dict[str, str]] = field(default_factory=list)
    subqueries: List['QueryAnalysis'] = field(default_factory=list)

class AdvancedSqlAnalyzer:
    def __init__(self):
        self.performance_patterns = {
            'no_index': r'WHERE\s+(?:[\w\.]+\.)?([\w]+)\s*[=<>!]',
            'full_scan': r'(?:[\w\.]+\.)?([\w]+)\s+IS NOT NULL|(?:[\w\.]+\.)?([\w]+)\s+LIKE\s+\'%',
            'wildcard_select': r'SELECT\s+(?:\w+\.)?\*',
            'multiple_joins': r'(?:INNER\s+|LEFT\s+|RIGHT\s+|FULL\s+)?JOIN\s+',
            'complex_subquery': r'\(SELECT\s+.*?\s+FROM\s+.*?\s+(?:WHERE\s+.*?)?\)',
            'large_offset': r'OFFSET\s+\d{4,}',
            'cross_join': r'CROSS\s+JOIN',
            'or_conditions': r'WHERE\s+.*?\sOR\s+.*?',
            'order_by_no_index': r'ORDER\s+BY\s+(?:[\w\.]+\.)?([\w]+)(?:\s+DESC)?\s*(?:LIMIT|$)',
            'implicit_conversion': r'WHERE\s+(?:[\w\.]+\.)?([\w]+)\s*=\s*\'.*?\'',
            'nested_loop_risk': r'WHERE\s+EXISTS\s*\(|WHERE\s+IN\s*\(',
            'temp_table_usage': r'(?:CREATE\s+TEMPORARY\s+TABLE|INTO\s+TEMP\s+TABLE)'
        }

    def analyze_file(self, file_path: str) -> Dict[str, QueryAnalysis]:
        """SQL dosyasını analiz eder"""
        try:
            with open(file_path, 'r', encoding='utf-8') as f:
                content = f.read()
            
            queries = self._split_queries(content)
            results = {}
            
            for query in queries:
                if not query.strip():
                    continue
                
                try:
                    normalized = self._normalize_query(query)
                    analysis = self._analyze_query(normalized, query)
                    results[hash(normalized)] = analysis
                except Exception as e:
                    logger.error(f"Query analysis failed: {str(e)}")
                    continue
                
            return results
        except Exception as e:
            logger.error(f"File processing failed: {str(e)}")
            return {}

    def _split_queries(self, content: str) -> List[str]:
        """SQL içeriğini tek tek sorgulara ayırır"""
        # String literal'leri ve blok yorumlarını geçici olarak değiştir
        placeholder_map = {}
        temp_id = 0
        
        # String literal'leri sakla
        def string_replacer(match):
            nonlocal temp_id
            key = f"__STR_{temp_id}__"
            temp_id += 1
            placeholder_map[key] = match.group(0)
            return key
        
        # Yorumları kaldır
        content = re.sub(r'--.*?$', '', content, flags=re.MULTILINE)
        content = re.sub(r'/\*.*?\*/', '', content, flags=re.DOTALL)
        
        # String literal'leri değiştir
        content = re.sub(r"'(?:\\.|[^'\\])*'", string_replacer, content)
        content = re.sub(r'"(?:\\.|[^"\\])*"', string_replacer, content)
        
        # Noktalı virgülle ayır
        queries = [q.strip() for q in re.split(r';(?=(?:[^\'"]*[\'"][^\'"]*[\'"])*[^\'"]*$)', content) if q.strip()]
        
        # String literal'leri geri yükle
        for key, value in placeholder_map.items():
            queries = [q.replace(key, value) for q in queries]
        
        return queries

    def _normalize_query(self, query: str) -> str:
        """Sorguyu analiz için normalize eder"""
        # Temel normalizasyon
        query = ' '.join(query.split()).upper()
        
        # Alias'ları standartlaştır
        query = re.sub(r'\bAS\b', '', query)
        
        # Veri tiplerini normalize et
        query = re.sub(r'\bINTEGER\b', 'INT', query)
        query = re.sub(r'\bVARCHAR\b', 'STRING', query)
        
        return query

    def _analyze_query(self, normalized_query: str, original_query: str) -> QueryAnalysis:
        """Tek bir sorguyu analiz eder"""
        analysis = QueryAnalysis(
            raw_query=original_query,
            normalized_query=normalized_query
        )
        
        # Türetilmiş tabloları (subquery'ler) işle
        self._process_subqueries(normalized_query, analysis)
        
        # Temel tabloları ve alias'ları çıkar
        self._extract_tables(normalized_query, analysis)
        
        # JOIN koşullarını analiz et
        self._extract_joins(normalized_query, analysis)
        
        # WHERE koşullarını analiz et
        self._extract_where_conditions(normalized_query, analysis)
        
        # SELECT ifadesindeki kolonları analiz et
        self._extract_select_columns(normalized_query, analysis)
        
        # Performans problemlerini tespit et
        self._detect_performance_issues(analysis)
        
        return analysis

    def _process_subqueries(self, query: str, analysis: QueryAnalysis):
        """Alt sorguları işler"""
        subquery_pattern = r'FROM\s*\(((SELECT\s+.*?\s+FROM\s+.*?(?:\s+WHERE\s+.*?)?))\)'
        
        for match in re.finditer(subquery_pattern, query, re.IGNORECASE):
            subquery = match.group(1)
            try:
                sub_analysis = self._analyze_query(subquery, subquery)
                analysis.subqueries.append(sub_analysis)
                
                # Türetilmiş tabloya alias bul
                alias_match = re.search(r'FROM\s*\(.*?\)\s*(\w+)', query[match.end():], re.IGNORECASE)
                if alias_match:
                    alias = alias_match.group(1)
                    analysis.tables[alias] = TableInfo(
                        name=f"SUBQUERY_{len(analysis.subqueries)}",
                        alias=alias,
                        is_derived=True
                    )
            except Exception as e:
                logger.warning(f"Subquery analysis failed: {str(e)}")

    def _extract_tables(self, query: str, analysis: QueryAnalysis):
        """Tabloları ve alias'ları çıkarır"""
        # FROM clause
        from_matches = re.finditer(
            r'(?:FROM|JOIN)\s+(\w+)(?:\s+(\w+))?(?=\s+(?:ON|WHERE|JOIN|GROUP|HAVING|ORDER|LIMIT|$))', 
            query, 
            re.IGNORECASE
        )
        
        for match in from_matches:
            table_name, alias = match.groups()
            alias = alias or table_name
            if alias not in analysis.tables:
                analysis.tables[alias] = TableInfo(
                    name=table_name,
                    alias=alias if alias != table_name else None
                )

    def _extract_joins(self, query: str, analysis: QueryAnalysis):
        """JOIN koşullarını analiz eder"""
        join_pattern = r'(?:INNER\s+|LEFT\s+|RIGHT\s+|FULL\s+)?JOIN\s+\w+(?:\s+\w+)?\s+ON\s+(.*?)(?=\s+(?:WHERE|JOIN|GROUP|HAVING|ORDER|LIMIT|$))'
        
        for match in re.finditer(join_pattern, query, re.IGNORECASE):
            condition = match.group(1)
            self._parse_join_condition(condition, analysis)

    def _parse_join_condition(self, condition: str, analysis: QueryAnalysis):
        """Tek bir JOIN koşulunu analiz eder"""
        # Basit eşitlik koşulları
        equality_matches = re.finditer(
            r'([\w]+)\.([\w]+)\s*=\s*([\w]+)\.([\w]+)', 
            condition, 
            re.IGNORECASE
        )
        
        for match in equality_matches:
            left_table, left_col, right_table, right_col = match.groups()
            
            # Tablolara kolonları ekle
            if left_table in analysis.tables:
                analysis.tables[left_table].columns.add(left_col)
            if right_table in analysis.tables:
                analysis.tables[right_table].columns.add(right_col)
            
            analysis.join_conditions.append({
                'left_table': left_table,
                'left_column': left_col,
                'right_table': right_table,
                'right_column': right_col,
                'condition': match.group(0)
            })

    def _extract_where_conditions(self, query: str, analysis: QueryAnalysis):
        """WHERE koşullarını analiz eder"""
        where_match = re.search(
            r'WHERE\s+(.*?)(?=\s+(?:GROUP BY|HAVING|ORDER BY|LIMIT|$))', 
            query, 
            re.IGNORECASE
        )
        
        if where_match:
            where_clause = where_match.group(1)
            self._parse_where_clause(where_clause, analysis)

    def _parse_where_clause(self, clause: str, analysis: QueryAnalysis):
        """WHERE koşulunu parçalarına ayırır"""
        # Koşulları AND/OR ile ayır (basitçe)
        conditions = re.split(r'\s+(?:AND|OR)\s+', clause)
        
        for cond in conditions:
            # Kolon kullanımını tespit et
            col_match = re.match(
                r'(?:([\w]+)\.)?([\w]+)\s*([=<>!]+)\s*(.*)', 
                cond, 
                re.IGNORECASE
            )
            
            if col_match:
                table, column, operator, value = col_match.groups()
                
                if table and table in analysis.tables:
                    analysis.tables[table].columns.add(column)
                elif not table:
                    # Alias yoksa, tüm tablolarda ara
                    for table_info in analysis.tables.values():
                        if column in table_info.columns:
                            table = table_info.alias or table_info.name
                            break
                
                analysis.where_conditions.append({
                    'table': table or '',
                    'column': column,
                    'operator': operator,
                    'value': value,
                    'full_condition': cond
                })

    def _extract_select_columns(self, query: str, analysis: QueryAnalysis):
        """SELECT ifadesindeki kolonları analiz eder"""
        select_match = re.search(
            r'SELECT\s+(.*?)(?=\s+FROM)', 
            query, 
            re.IGNORECASE
        )
        
        if select_match:
            select_clause = select_match.group(1)
            self._parse_select_clause(select_clause, analysis)

    def _parse_select_clause(self, clause: str, analysis: QueryAnalysis):
        """SELECT ifadesini analiz eder"""
        # Kolonları ayır (virgülle)
        columns = [col.strip() for col in clause.split(',')]
        
        for col in columns:
            # Kolon ve tablo/alias eşleştirme
            col_match = re.match(
                r'(?:([\w]+)\.)?([\w*]+)(?:\s+AS\s+\w+)?', 
                col, 
                re.IGNORECASE
            )
            
            if col_match:
                table, column = col_match.groups()
                
                if column == '*':
                    analysis.potential_issues.append("Wildcard SELECT (*) used")
                    continue
                
                if table and table in analysis.tables:
                    analysis.tables[table].columns.add(column)
                elif not table:
                    # Alias yoksa, tüm tablolarda ara
                    for table_info in analysis.tables.values():
                        if column in table_info.columns:
                            table = table_info.alias or table_info.name
                            break

    def _detect_performance_issues(self, analysis: QueryAnalysis):
        """Performans problemlerini tespit eder"""
        # JOIN sayısı kontrolü
        if len(analysis.join_conditions) >= 3:
            analysis.potential_issues.append(
                f"Multiple joins detected ({len(analysis.join_conditions)} tables)"
            )
        
        # WHERE koşulları analizi
        for cond in analysis.where_conditions:
            # LIKE with leading %
            if 'LIKE' in cond['full_condition'].upper() and "'%" in cond['full_condition'].upper():
                analysis.potential_issues.append(
                    f"Leading wildcard in LIKE condition: {cond['table']}.{cond['column']}"
                )
            
            # Implicit conversion
            if cond['operator'] == '=' and "'" in cond['value'] and cond['column'].upper() not in ('DATE', 'TIME', 'TEXT'):
                analysis.potential_issues.append(
                    f"Possible implicit conversion: {cond['table']}.{cond['column']} = {cond['value']}"
                )
        
        # ORDER BY kontrolü
        order_match = re.search(
            r'ORDER\s+BY\s+(?:[\w]+\.)?([\w]+)', 
            analysis.normalized_query, 
            re.IGNORECASE
        )
        if order_match:
            column = order_match.group(1)
            analysis.potential_issues.append(
                f"ORDER BY on column: {column} (check for index)"
            )

    def generate_report(self, analysis_results: Dict[str, QueryAnalysis]):
        """Analiz raporu oluşturur"""
        print("="*100)
        print("SQL PERFORMANCE ANALYSIS REPORT".center(100))
        print("="*100)
        
        for q_hash, analysis in analysis_results.items():
            print(f"\n{' QUERY '.center(100, '-')}")
            print(f"\nOriginal Query:\n{analysis.raw_query[:500]}...\n")
            
            print("Tables and Columns:")
            for alias, table in analysis.tables.items():
                cols = ', '.join(sorted(table.columns)) if table.columns else 'N/A'
                print(f"- {table.name}{f' (as {alias})' if alias != table.name else ''}")
                print(f"  Columns: {cols}")
            
            if analysis.join_conditions:
                print("\nJoin Conditions:")
                for join in analysis.join_conditions:
                    print(f"- {join['left_table']}.{join['left_column']} = {join['right_table']}.{join['right_column']}")
            
            if analysis.where_conditions:
                print("\nWHERE Conditions:")
                for cond in analysis.where_conditions:
                    table_ref = f"{cond['table']}." if cond['table'] else ''
                    print(f"- {table_ref}{cond['column']} {cond['operator']} {cond['value']}")
            
            if analysis.potential_issues:
                print("\nPotential Performance Issues:")
                for issue in analysis.potential_issues:
                    print(f"- {issue}")
            
            if analysis.subqueries:
                print("\nSubqueries Found:")
                for i, subquery in enumerate(analysis.subqueries, 1):
                    print(f"  Subquery #{i}: {len(subquery.tables)} tables, {len(subquery.potential_issues)} issues")
            
            print("\n" + "-"*100)

if __name__ == "__main__":
    analyzer = AdvancedSqlAnalyzer()
    
    try:
        # SQL dosyasını analiz et
        results = analyzer.analyze_file("sql_queries.sql")
        
        # Rapor oluştur
        analyzer.generate_report(results)
        
        # Özet istatistikler
        total_queries = len(results)
        problematic_queries = sum(1 for a in results.values() if a.potential_issues)
        
        print("\nSUMMARY STATISTICS:")
        print(f"Total Queries Analyzed: {total_queries}")
        print(f"Queries with Potential Issues: {problematic_queries} ({problematic_queries/total_queries:.1%})")
        
    except Exception as e:
        logger.error(f"Analysis failed: {str(e)}")
