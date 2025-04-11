import re
from collections import defaultdict
import sqlparse

def analyze_sql_file(file_path):
    # SQL dosyasını oku
    with open(file_path, 'r') as file:
        sql_content = file.read()
    
    # Sorguları ayır
    queries = sqlparse.split(sql_content)
    
    # Analiz sonuçlarını sakla
    analysis_results = []
    
    for query in queries:
        if not query.strip():
            continue
            
        try:
            parsed = sqlparse.parse(query)[0]
            analysis = analyze_query(parsed)
            if analysis:
                analysis_results.append(analysis)
        except Exception as e:
            print(f"Query analysis error: {e}")
            continue
    
    return analysis_results

def analyze_query(parsed_query):
    # Temel bilgileri topla
    tables = set()
    join_columns = defaultdict(set)
    where_columns = set()
    order_by_columns = set()
    group_by_columns = set()
    
    # FROM clause analizi
    for token in parsed_query.tokens:
        if isinstance(token, sqlparse.sql.IdentifierList):
            for identifier in token.get_identifiers():
                tables.add(identifier.get_real_name())
        elif isinstance(token, sqlparse.sql.Identifier):
            tables.add(token.get_real_name())
    
    # WHERE clause analizi
    where_clause = None
    for token in parsed_query.tokens:
        if isinstance(token, sqlparse.sql.Where):
            where_clause = token
            break
    
    if where_clause:
        for token in where_clause.tokens:
            if isinstance(token, sqlparse.sql.Comparison):
                left = token.left.value
                where_columns.add(left)
    
    # JOIN analizi (basitleştirilmiş)
    join_keywords = ['join', 'inner join', 'left join', 'right join']
    for token in parsed_query.tokens:
        if token.value.lower() in join_keywords:
            # Basit JOIN analizi - gerçekte daha karmaşık olmalı
            next_token = parsed_query.token_next(token)[1]
            if next_token:
                join_columns[next_token.get_real_name()].add(token.value)
    
    # ORDER BY ve GROUP BY analizi
    for token in parsed_query.tokens:
        if isinstance(token, sqlparse.sql.IdentifierList):
            if any(t.value.lower() == 'order by' for t in token.tokens):
                order_by_columns.update(get_columns_from_list(token))
            elif any(t.value.lower() == 'group by' for t in token.tokens):
                group_by_columns.update(get_columns_from_list(token))
    
    # Index önerilerini oluştur
    index_recommendations = generate_index_recommendations(
        tables, join_columns, where_columns, order_by_columns, group_by_columns
    )
    
    return {
        'query': str(parsed_query),
        'tables': list(tables),
        'join_columns': dict(join_columns),
        'where_columns': list(where_columns),
        'order_by_columns': list(order_by_columns),
        'group_by_columns': list(group_by_columns),
        'index_recommendations': index_recommendations
    }

def get_columns_from_list(token_list):
    columns = set()
    for identifier in token_list.get_identifiers():
        columns.add(identifier.get_real_name())
    return columns

def generate_index_recommendations(tables, join_columns, where_columns, order_by, group_by):
    recommendations = []
    
    # JOIN için index önerileri
    for table, joins in join_columns.items():
        for join_type in joins:
            recommendations.append(
                f"CREATE INDEX idx_{table}_join ON {table}({', '.join(join_columns[table])}); -- JOIN optimizasyonu"
            )
    
    # WHERE için index önerileri
    for column in where_columns:
        recommendations.append(
            f"CREATE INDEX idx_{column}_filter ON {column.split('.')[0]}({column.split('.')[-1]}); -- WHERE koşulu optimizasyonu"
        )
    
    # ORDER BY/GROUP BY için index önerileri
    if order_by:
        recommendations.append(
            f"CREATE INDEX idx_ordering ON {order_by[0].split('.')[0]}({', '.join([col.split('.')[-1] for col in order_by])}); -- ORDER BY optimizasyonu"
        )
    
    if group_by:
        recommendations.append(
            f"CREATE INDEX idx_grouping ON {group_by[0].split('.')[0]}({', '.join([col.split('.')[-1] for col in group_by])}); -- GROUP BY optimizasyonu"
        )
    
    return recommendations

def main():
    file_path = 'output_raw_sql.txt'  # Varsayılan dosya yolu
    results = analyze_sql_file(file_path)
    
    # Sonuçları yazdır
    for i, result in enumerate(results, 1):
        print(f"\n=== Sorgu {i} Analizi ===")
        print(f"\nTablo(lar): {', '.join(result['tables'])}")
        print(f"\nJOIN Sütunları: {result['join_columns']}")
        print(f"\nWHERE Sütunları: {', '.join(result['where_columns'])}")
        print(f"\nORDER BY Sütunları: {', '.join(result['order_by_columns'])}")
        print(f"\nGROUP BY Sütunları: {', '.join(result['group_by_columns'])}")
        
        print("\nÖnerilen Indexler:")
        for rec in result['index_recommendations']:
            print(f"- {rec}")
        
        print("\n" + "="*50 + "\n")

if __name__ == "__main__":
    main()
