#!/usr/bin/env python3
import re
from collections import OrderedDict
import sys

def extract_sql_from_line(line):
    """Log satırından SQL sorgusunu çıkarır"""
    # Pattern: timestamp DEBUG ... org.hibernate.SQL : SQL_QUERY [parametreler]
    match = re.search(r'org\.hibernate\.SQL\s*:\s*(.*?)(?:\s*\[.*\])?$', line)
    if match:
        return match.group(1).strip()
    return None

def main():
    if len(sys.argv) != 3:
        print("Kullanım: python3 parse_sql_logs.py <input_log_file> <output_sql_file>")
        sys.exit(1)
    
    input_file = sys.argv[1]
    output_file = sys.argv[2]
    
    unique_sqls = OrderedDict()
    
    with open(input_file, 'r') as f:
        for line in f:
            sql = extract_sql_from_line(line)
            if sql:
                # Parametreleri standartlaştır
                sql = re.sub(r'\?\d*', '?', sql)
                sql = re.sub(r'\s+', ' ', sql).strip()
                unique_sqls[sql] = None
    
    with open(output_file, 'w') as f:
        for sql in unique_sqls.keys():
            f.write(f"{sql}\n")
    
    print(f"Toplam {len(unique_sqls)} benzersiz SQL sorgusu bulundu.")
    print(f"Temizlenmiş SQL sorguları {output_file} dosyasına yazıldı.")

if __name__ == "__main__":
    main()
