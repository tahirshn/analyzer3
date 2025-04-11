import re
from collections import defaultdict

def analyze_sql_file(file_path):
    """SQL dosyasını okuyup sorguları analiz eder."""
    with open(file_path, 'r', encoding='utf-8') as file:
        sql_content = file.read()
    
    # Sorguları ayır (basit regex ile)
    queries = re.split(r';\s*\n', sql_content)
    analysis_results = []
    
    for query in queries:
        query = query.strip()
        if not query or query.startswith('--'):
            continue
            
        try:
            analysis = analyze_query(query)
            if analysis:
                analysis_results.append(analysis)
        except Exception as e:
            print(f"Sorgu analiz hatası: {str(e)}")
            continue
    
    return analysis_results

def analyze_query(query):
    """Tek bir SQL sorgusunu analiz eder."""
    # Case insensitive analiz için küçük harfe çevir
    lower_query = query.lower()
    
    # Temel bilgileri topla
    tables = extract_tables(query)
    join_columns = extract_join_columns(query)
    where_columns = extract_where_columns(query)
    order_by_columns = extract_order_by_columns(query)
    group_by_columns = extract_group_by_columns(query)
    
    # Index önerilerini oluştur
    index_recommendations = generate_index_recommendations(
        tables, join_columns, where_columns, order_by_columns, group_by_columns
    )
    
    return {
        'query': query,
        'tables': tables,
        'join_columns': join_columns,
        'where_columns': where_columns,
        'order_by_columns': order_by_columns,
        'group_by_columns': group_by_columns,
        'index_recommendations': index_recommendations
    }

def extract_tables(query):
    """FROM clause'dan tablo isimlerini çıkarır."""
    # Basit regex yaklaşımı
    tables = set()
    from_match = re.search(r'from\s+([^\s,(]+)(?:\s*,\s*([^\s,(]+))?', query, re.I)
    if from_match:
        tables.update(t for t in from_match.groups() if t)
    
    # JOIN ifadelerinden tablolar
    join_matches = re.finditer(r'(?:inner|left|right|full)\s+join\s+([^\s,]+)', query, re.I)
    tables.update(m.group(1) for m in join_matches)
    
    return list(tables)

def extract_join_columns(query):
    """JOIN koşullarındaki sütunları çıkarır."""
    joins = defaultdict(set)
    join_matches = re.finditer(r'on\s+([^=]+)\s*=\s*([^)\s,]+)', query, re.I)
    
    for match in join_matches:
        left, right = match.groups()
        left_table = left.split('.')[0] if '.' in left else None
        right_table = right.split('.')[0] if '.' in right else None
        
        if left_table:
            joins[left_table].add(left.split('.')[-1])
        if right_table:
            joins[right_table].add(right.split('.')[-1])
    
    return dict(joins)

def extract_where_columns(query):
    """WHERE clause'daki sütunları çıkarır."""
    where_columns = set()
    where_pos = query.lower().find('where')
    if where_pos == -1:
        return []
    
    where_part = query[where_pos+5:]
    # Basit koşulları bul
    conditions = re.split(r'\band\b|\bor\b', where_part, flags=re.I)
    
    for cond in conditions:
        # Sütun adını bulmaya çalış
        match = re.search(r'([a-z_][a-z0-9_]*)(?:\.[a-z_][a-z0-9_]*)?\s*[=<>]', cond, re.I)
        if match:
            col = match.group(1)
            if '.' in col:
                col = col.split('.')[-1]
            where_columns.add(col)
    
    return list(where_columns)

def extract_order_by_columns(query):
    """ORDER BY sütunlarını çıkarır."""
    order_match = re.search(r'order\s+by\s+([^);]+)', query, re.I)
    if not order_match:
        return []
    
    order_part = order_match.group(1)
    columns = re.findall(r'([a-z_][a-z0-9_]*)(?:\.[a-z_][a-z0-9_]*)?(?:\s+asc|\s+desc)?', order_part, re.I)
    return [col[0].split('.')[-1] for col in columns]

def extract_group_by_columns(query):
    """GROUP BY sütunlarını çıkarır."""
    group_match = re.search(r'group\s+by\s+([^);]+)', query, re.I)
    if not group_match:
        return []
    
    group_part = group_match.group(1)
    columns = re.findall(r'([a-z_][a-z0-9_]*)(?:\.[a-z_][a-z0-9_]*)?', group_part, re.I)
    return [col.split('.')[-1] for col in columns]

def generate_index_recommendations(tables, join_columns, where_columns, order_by, group_by):
    """Analiz sonuçlarına göre index önerileri üretir."""
    recommendations = []
    
    # JOIN sütunları için index
    for table, columns in join_columns.items():
        if table in tables and columns:
            rec = f"CREATE INDEX idx_{table}_join ON {table}({', '.join(columns)});  -- JOIN optimizasyonu"
            recommendations.append(rec)
    
    # WHERE sütunları için index
    for column in where_columns:
        if column:
            # Hangi tabloya ait olduğunu bul
            table = find_table_for_column(tables, column)
            if table:
                rec = f"CREATE INDEX idx_{table}_{column} ON {table}({column});  -- WHERE koşulu optimizasyonu"
                recommendations.append(rec)
    
    # ORDER BY için composite index
    if order_by and len(order_by) > 0:
        table = find_table_for_column(tables, order_by[0])
        if table:
            rec = f"CREATE INDEX idx_{table}_order ON {table}({', '.join(order_by)});  -- ORDER BY optimizasyonu"
            recommendations.append(rec)
    
    # GROUP BY için composite index
    if group_by and len(group_by) > 0:
        table = find_table_for_column(tables, group_by[0])
        if table:
            rec = f"CREATE INDEX idx_{table}_group ON {table}({', '.join(group_by)});  -- GROUP BY optimizasyonu"
            recommendations.append(rec)
    
    return list(set(recommendations))  # Tekilleri al

def find_table_for_column(tables, column):
    """Bir sütunun hangi tabloya ait olabileceğini tahmin eder."""
    if not tables:
        return None
    # Basit yaklaşım: ilk tabloyu döndür
    return tables[0]

def print_analysis_report(results):
    """Analiz sonuçlarını okunabilir şekilde yazdırır."""
    for i, result in enumerate(results, 1):
        print(f"\n=== Sorgu {i} Analizi ===")
        print(f"\nSorgu:\n{result['query'][:200]}...")  # Sadece ilk 200 karakter
        
        print(f"\nTablolar: {', '.join(result['tables'])}")
        print(f"JOIN Sütunları: {result['join_columns']}")
        print(f"WHERE Sütunları: {', '.join(result['where_columns'])}")
        print(f"ORDER BY Sütunları: {', '.join(result['order_by_columns'])}")
        print(f"GROUP BY Sütunları: {', '.join(result['group_by_columns'])}")
        
        print("\nÖnerilen Indexler:")
        for rec in result['index_recommendations']:
            print(f"- {rec}")
        
        print("\n" + "="*80)

def main():
    print("SQL Sorgu Analiz ve Index Öneri Aracı")
    file_path = input("SQL dosya yolu (varsayılan: output_raw_sql.txt): ") or "output_raw_sql.txt"
    
    try:
        results = analyze_sql_file(file_path)
        print_analysis_report(results)
        print(f"\nToplam {len(results)} sorgu analiz edildi.")
    except Exception as e:
        print(f"Hata oluştu: {str(e)}")

if __name__ == "__main__":
    main()
