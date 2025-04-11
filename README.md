import re

def extract_tables_and_columns(query):
    """SQL sorgusunda tabloları ve ilgili kolonları çıkar."""
    tables = {}
    
    # FROM ve JOIN'deki tabloları ayıklayalım
    from_tables = re.findall(r"\bfrom\b\s+([^\s,]+)", query, re.IGNORECASE)
    join_tables = re.findall(r"\bjoin\b\s+([^\s,]+)", query, re.IGNORECASE)
    tables.update({table: [] for table in from_tables + join_tables})
    
    # WHERE ve ON koşullarındaki kolonları tespit et
    where_match = re.findall(r"(?:where|on)\s+(.+?)(?:\s+(?:and|or|group|order|limit|;|$))", query, re.IGNORECASE)
    
    for condition in where_match:
        # WHERE ve JOIN koşullarında yer alan kolonları çıkartalım
        columns = re.findall(r"\b(\w+)\b", condition)
        
        for column in columns:
            for table in tables:
                # Tablonun kolonlarıyla ilişkilendir
                if column not in tables[table]:
                    tables[table].append(column)
    
    return tables

def analyze_sql_queries(file_path):
    with open(file_path, 'r', encoding='utf-8') as file:
        sql_queries = file.read().split(';')  # Sorguları ayır

    for query in sql_queries:
        query = query.strip()
        if not query:
            continue

        # Tablo ve kolonları çıkar
        tables_and_columns = extract_tables_and_columns(query)

        print(f"Analyzing Query: {query}")
        for table, columns in tables_and_columns.items():
            print(f"Table: {table}")
            print(f"Columns: {columns}")
        print("-" * 50)

# Kullanım
analyze_sql_queries('output_raw_sql.txt')
