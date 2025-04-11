import re
from collections import defaultdict

class SQLQueryAnalyzer:
    def __init__(self, sql):
        self.sql = sql.strip().lower()
        self.tables = defaultdict(list)  # tablonun ismi -> kolonlar
        self.aliases = {}  # alias -> tablo adı
    
    def extract_tables_and_columns(self):
        """SQL sorgusunda kullanılan tabloları ve kolonları ayıkla."""
        # 1. FROM ve JOIN ifadelerinde yer alan tabloları çıkaralım
        self._extract_from_and_joins()

        # 2. WHERE ve ON kısımlarındaki kolonları çıkartalım
        self._extract_columns_from_conditions()

    def _extract_from_and_joins(self):
        """FROM ve JOIN ifadelerinden tabloları çıkar."""
        # Tabloları bulmak için FROM ve JOIN anahtar kelimelerini kullanıyoruz
        from_and_joins = re.findall(r"\b(from|join)\s+([^\s,]+)", self.sql)
        
        for keyword, table in from_and_joins:
            table_name = table.strip().split()[0]  # Alias kullanılıyorsa, önce tablonun adını al
            if table_name not in self.aliases:
                self.aliases[table_name] = table_name
            else:
                self.aliases[table_name] = table
            
            # Tabloyu ve kolonları ilişkilendir
            self.tables[table_name] = []

    def _extract_columns_from_conditions(self):
        """WHERE ve ON koşullarındaki kolonları çıkar."""
        # WHERE ve ON koşullarını bul
        where_on_conditions = re.findall(r"(where|on)\s+([^\s;]+(?:[^\s;]+)*)", self.sql)
        
        for condition in where_on_conditions:
            condition_str = condition[1].strip()
            columns = re.findall(r"\b(\w+)\b", condition_str)  # Kolonları çıkart
            for table, columns_in_condition in self.tables.items():
                # Her tablodaki kolonları kaydedelim
                self.tables[table].extend(columns)

    def get_results(self):
        return self.tables

def analyze_sql_queries(file_path):
    with open(file_path, 'r', encoding='utf-8') as file:
        sql_queries = file.read().split(';')  # Sorguları ayır

    for query in sql_queries:
        query = query.strip()
        if not query:
            continue
        
        analyzer = SQLQueryAnalyzer(query)
        analyzer.extract_tables_and_columns()
        tables_and_columns = analyzer.get_results()

        print(f"Analyzing Query: {query}")
        for table, columns in tables_and_columns.items():
            print(f"Table: {table}")
            print(f"Columns: {set(columns)}")  # Set yapısı ile tekrarlanan kolonları engeller
        print("-" * 50)

# Kullanım
analyze_sql_queries('output_raw_sql.txt')
