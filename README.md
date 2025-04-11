import sqlparse
from sql_metadata import Parser

def analyze_sql_queries(file_path):
    with open(file_path, 'r', encoding='utf-8') as file:
        sql_queries = file.read().split(';')  # Sorguları ayır

    for query in sql_queries:
        query = query.strip()
        if not query:
            continue

        parser = Parser(query)
        tables = parser.tables
        columns = parser.columns

        print(f"Analyzing Query: {query}")
        print(f"Tables: {tables}")
        print(f"Columns: {columns}")
        print("-" * 50)

# Kullanım
analyze_sql_queries('output_raw_sql.txt')
