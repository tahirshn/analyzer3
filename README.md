#!/usr/bin/env python3
import re
from collections import OrderedDict
import sys

def extract_sql_from_line(line):
    """Log satırından SQL sorgusunu çıkarır"""
    # Pattern: timestamp DEBUG thread org.hibernate.SQL : SQL_QUERY [parameters]
    pattern = r'org\.hibernate\.SQL\s*:\s*(.*?)(?:\s*\[\s*\])?$'
    match = re.search(pattern, line)
    if match:
        return match.group(1).strip()
    return None

def clean_sql(sql):
    """SQL sorgusunu temizler"""
    # Fazla boşlukları kaldır
    sql = re.sub(r'\s+', ' ', sql).strip()
    return sql

def process_log_file(input_file, output_file):
    """Log dosyasını işler ve temiz SQL'leri kaydeder"""
    unique_sqls = OrderedDict()
    
    with open(input_file, 'r', encoding='utf-8') as f:
        for line in f:
            sql = extract_sql_from_line(line)
            if sql:
                cleaned_sql = clean_sql(sql)
                unique_sqls[cleaned_sql] = None
    
    with open(output_file, 'w', encoding='utf-8') as f:
        for i, sql in enumerate(unique_sqls.keys(), 1):
            f.write(f"{i}. {sql}\n\n")
    
    return len(unique_sqls)

def main():
    if len(sys.argv) != 3:
        print("Kullanım: python3 parse_sql_logs.py <input_log_file> <output_sql_file>")
        sys.exit(1)
    
    input_file = sys.argv[1]
    output_file = sys.argv[2]
    
    count = process_log_file(input_file, output_file)
    
    print(f"Toplam {count} benzersiz SQL sorgusu bulundu.")
    print(f"Temizlenmiş SQL sorguları '{output_file}' dosyasına yazıldı.")

if __name__ == "__main__":
    main()
