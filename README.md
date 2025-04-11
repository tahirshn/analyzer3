import re

class SQLQueryAnalyzer:
    def __init__(self, sql):
        self.sql = sql.strip().lower()
        self.tables = set()
        self.columns = set()
        self.joins = []
        self.where_conditions = []
        self.order_by_columns = set()
        self.group_by_columns = set()
        self.subqueries = []

    def analyze(self):
        """SQL sorgusunu analiz et."""
        self.extract_tables_and_columns()
        self.extract_joins()
        self.extract_conditions()
        self.extract_order_by_and_group_by()
        self.extract_subqueries()

    def extract_tables_and_columns(self):
        """SQL sorgusundaki tablolardan ve kolonlardan bilgileri çıkar."""
        # Tabloları ve kolonları FROM, JOIN ve WHERE ifadelerinden alalım
        tables_and_columns = re.findall(r'(\bfrom\b|\bjoin\b)\s+(\w+)(?:\s+as\s+(\w+))?', self.sql)
        
        for keyword, table, alias in tables_and_columns:
            table_name = alias if alias else table
            self.tables.add(table_name)

            # Kolonları WHERE, JOIN ve SELECT içinde arayalım
            columns_in_table = re.findall(r'\b' + re.escape(table_name) + r'\.(\w+)', self.sql)
            self.columns.update(columns_in_table)

    def extract_joins(self):
        """JOIN ifadelerinde kullanılan kolonları çıkar."""
        join_conditions = re.findall(r'\b(on|using)\s+([^\s;]+)', self.sql)
        
        for condition in join_conditions:
            join_condition = condition[1]
            columns_in_join = re.findall(r'(\w+)', join_condition)
            self.columns.update(columns_in_join)

    def extract_conditions(self):
        """WHERE koşullarındaki kolonları çıkar."""
        where_conditions = re.findall(r'\bwhere\b\s+([^\s;]+)', self.sql)
        
        for condition in where_conditions:
            columns_in_condition = re.findall(r'(\w+)', condition)
            self.columns.update(columns_in_condition)

    def extract_order_by_and_group_by(self):
        """ORDER BY ve GROUP BY koşullarındaki kolonları çıkar."""
        order_by_columns = re.findall(r'\border by\b\s+([^\s;]+)', self.sql)
        group_by_columns = re.findall(r'\bgroup by\b\s+([^\s;]+)', self.sql)
        
        self.order_by_columns.update(order_by_columns)
        self.group_by_columns.update(group_by_columns)

    def extract_subqueries(self):
        """Alt sorgulardaki kolonları çıkar."""
        subqueries = re.findall(r'\(.*?\)', self.sql)
        self.subqueries.extend(subqueries)

    def get_analysis(self):
        """Sorgunun analizini döndüren fonksiyon."""
        return {
            "tables": self.tables,
            "columns": self.columns,
            "joins": self.joins,
            "where_conditions": self.where_conditions,
            "order_by_columns": self.order_by_columns,
            "group_by_columns": self.group_by_columns,
            "subqueries": self.subqueries
        }

def analyze_sql_queries(file_path):
    with open(file_path, 'r', encoding='utf-8') as file:
        sql_queries = file.read().split(';')  # Sorguları ayır

    for query in sql_queries:
        query = query.strip()
        if not query:
            continue
        
        analyzer = SQLQueryAnalyzer(query)
        analyzer.analyze()
        analysis = analyzer.get_analysis()

        # Sorgu analizi ve index gereksinimlerini değerlendirelim
        print(f"Analyzing Query: {query}")
        
        # 1. ÇOKLU JOIN içeren sorgular
        if len(analysis['joins']) > 1:
            print("Multiple JOIN detected. Indexes may be needed on the join conditions.")
        
        # 2. BÜYÜK VERİ KÜMELERİNE ÇALIŞAN SORGULAR
        if len(analysis['tables']) > 3:
            print("Query involves multiple large tables. Indexing may improve performance.")
        
        # 3. KARMASIK WHERE KOŞULLARI
        if len(analysis['where_conditions']) > 2:
            print("Complex WHERE conditions detected. Consider indexing involved columns.")
        
        # 4. ORDER BY veya GROUP BY kullanılan sorgular
        if analysis['order_by_columns'] or analysis['group_by_columns']:
            print("ORDER BY or GROUP BY used. Indexing relevant columns may help performance.")
        
        # 5. ALT SORGULAR
        if analysis['subqueries']:
            print("Subqueries detected. Ensure proper indexing for subquery performance.")
        
        print("-" * 50)

# Kullanım
analyze_sql_queries('output_raw_sql.txt')
