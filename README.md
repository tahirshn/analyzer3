#!/usr/bin/env python3
import re
from collections import OrderedDict
import sys

def clean_sql(sql):
    """SQL sorgusunu temizler ve standardize eder"""
    # Parametreleri standartlaştır
    sql = re.sub(r'\?\d*', '?', sql)
    sql = re.sub(r':[a-zA-Z0-9_]+', '?', sql)
    # Fazla boşlukları kaldır
    sql = re.sub(r'\s+', ' ', sql).strip()
    return sql

def extract_unique_sqls(input_file):
    """Log dosyasından benzersiz SQL sorgularını çıkarır"""
    sql_pattern = re.compile(r'(?:Hibernate:|org\.hibernate\.SQL.*-)\s*(.*)')
    unique_sqls = OrderedDict()
    
    with open(input_file, 'r') as f:
        for line in f:
            match = sql_pattern.search(line)
            if match:
                sql = match.group(1).strip()
                if sql and sql.lower().startswith(('select', 'insert', 'update', 'delete', 'call', 'alter', 'create', 'drop')):
                    cleaned_sql = clean_sql(sql)
                    unique_sqls[cleaned_sql] = None
    return unique_sqls

def write_sqls_to_file(unique_sqls, output_file):
    """SQL sorgularını numaralandırılmış şekilde dosyaya yazar"""
    with open(output_file, 'w') as f:
        for i, sql in enumerate(unique_sqls.keys(), 1):
            f.write(f"{i}. {sql}\n\n")

def main():
    if len(sys.argv) != 3:
        print("Kullanım: python3 parse_sql_logs.py <input_log_file> <output_sql_file>")
        sys.exit(1)
    
    input_file = sys.argv[1]
    output_file = sys.argv[2]
    
    unique_sqls = extract_unique_sqls(input_file)
    write_sqls_to_file(unique_sqls, output_file)
    
    print(f"Toplam {len(unique_sqls)} benzersiz SQL sorgusu bulundu.")
    print(f"Temizlenmiş SQL sorguları {output_file} dosyasına yazıldı.")

if __name__ == "__main__":
    main()
