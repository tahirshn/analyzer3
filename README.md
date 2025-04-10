#!/usr/bin/env python3
import re
from collections import OrderedDict
import sys

def extract_sql_from_line(line):
    """Log satırından SQL sorgusunu çıkarır"""
    # Pattern: timestamp DEBUG ... org.hibernate.SQL : SQL_SORGUSU []
    match = re.search(r'org\.hibernate\.SQL\s*:\s*(.*?)(\s*\[\])?$', line)
    if match:
        return match.group(1).strip()
    return None

def process_log_file(input_file, output_file):
    """Log dosyasını işler ve temizlenmiş SQL'leri yazar"""
    unique_sqls = OrderedDict()
    
    with open(input_file, 'r') as f:
        for line in f:
            sql = extract_sql_from_line(line)
            if sql:
                # Parametreleri standartlaştır
                sql = re.sub(r'\?\d*', '?', sql)
                sql = re.sub(r':[a-zA-Z0-9_]+', '?', sql)
                # Fazla boşlukları temizle
                sql = re.sub(r'\s+', ' ', sql).strip()
                unique_sqls[sql] = None
    
    # SQL'leri dosyaya yaz
    with open(output_file, 'w') as f:
        for sql in unique_sqls.keys():
            f.write(f"{sql}\n")
    
    return len(unique_sqls)

def main():
    if len(sys.argv) != 3:
        print("Kullanım: python3 parse_sql_logs.py <input_log_file> <output_sql_file>")
        sys.exit(1)
    
    input_file = sys.argv[1]
    output_file = sys.argv[2]
    
    count = process_log_file(input_file, output_file)
    print(f"Toplam {count} benzersiz SQL sorgusu bulundu ve {output_file} dosyasına yazıldı.")

if __name__ == "__main__":
    main()
