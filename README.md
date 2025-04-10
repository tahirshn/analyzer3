#!/usr/bin/env python3
import re
import sys
from pathlib import Path

def extract_sql_queries(log_file_path, output_file_path=None):
    """
    Hibernate SQL log dosyasından raw SQL sorgularını çıkarır
    
    Args:
        log_file_path (str): SQL log dosyasının yolu
        output_file_path (str, optional): Çıktı dosyası. None ise konsola yazar.
    """
    try:
        log_content = Path(log_file_path).read_text(encoding='utf-8')
    except FileNotFoundError:
        print(f"Hata: {log_file_path} dosyası bulunamadı!")
        sys.exit(1)

    # SQL sorgularını yakalamak için regex pattern
    sql_pattern = re.compile(
        r'(?:Hibernate:|org\.hibernate\.SQL.*?-)\s*(.*?)(?=\n\S|\Z)', 
        re.DOTALL
    )
    
    # Parametre değerlerini yakalamak için regex
    param_pattern = re.compile(r'binding parameter \[(\d+)\] as \[([^\]]+)\] - \[([^\]]+)\]')
    
    queries = sql_pattern.findall(log_content)
    param_bindings = param_pattern.findall(log_content)
    
    # Parametreleri sorgulara yerleştir
    formatted_queries = []
    param_dict = {}
    
    # Parametreleri topla
    for param in param_bindings:
        pos, param_type, value = param
        param_dict[int(pos)] = (param_type, value)
    
    for query in queries:
        # Temizleme
        query = query.strip()
        if not query:
            continue
            
        # Parametreleri yerleştir
        for pos, (param_type, value) in param_dict.items():
            if param_type.lower() in ('string', 'varchar', 'char'):
                value = f"'{value}'"
            query = re.sub(r'\?', value, query, count=1)
        
        # Formatlama
        query = re.sub(r'\s+', ' ', query).strip()
        query = query.replace(' ,', ',').replace('( ', '(').replace(' )', ')')
        formatted_queries.append(query)
    
    # Benzersiz sorguları al
    unique_queries = list(dict.fromkeys(formatted_queries))
    
    # Çıktıyı hazırla
    output = f"Toplam {len(unique_queries)} SQL sorgusu bulundu:\n\n"
    output += "\n\n".join(f"{i+1}. {q}" for i, q in enumerate(unique_queries))
    
    if output_file_path:
        Path(output_file_path).write_text(output, encoding='utf-8')
        print(f"SQL sorguları {output_file_path} dosyasına kaydedildi")
    else:
        print(output)

if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("Kullanım: python3 extract_sql.py <log_dosyası> [<çıktı_dosyası>]")
        sys.exit(1)
        
    log_file = sys.argv[1]
    output_file = sys.argv[2] if len(sys.argv) > 2 else None
    
    extract_sql_queries(log_file, output_file)
