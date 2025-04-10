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



#!/usr/bin/env python3
import re
import sys

def extract_unique_queries(log_file_path, output_file_path):
    # SQL sorgularını tutacak küme (otomatik olarak benzersizlik sağlar)
    unique_queries = set()
    
    # SQL sorgusu deseni (Hibernate formatına uygun)
    sql_pattern = re.compile(r'^(?:Hibernate:|org\.hibernate\.SQL.*-)\s*(.*)$', re.MULTILINE)
    
    try:
        with open(log_file_path, 'r', encoding='utf-8') as file:
            content = file.read()
            matches = sql_pattern.finditer(content)
            
            for match in matches:
                query = match.group(1).strip()
                # Sorguyu temizle ve kümede sakla
                if query and not query.startswith('/*'):
                    # Parametre değerlerini kaldır (?: için non-capturing group)
                    cleaned_query = re.sub(r'\?(\d+)(?::\w+)?', '?', query)
                    unique_queries.add(cleaned_query)
                    
    except FileNotFoundError:
        print(f"Hata: Dosya bulunamadı - {log_file_path}")
        sys.exit(1)
    
    # Benzersiz sorguları sıralı listeye çevir
    sorted_queries = sorted(unique_queries)
    
    # Çıktı dosyasına yaz
    with open(output_file_path, 'w', encoding='utf-8') as out_file:
        for i, query in enumerate(sorted_queries, start=1):
            out_file.write(f"{i}. {query}\n\n")
    
    print(f"Toplam {len(sorted_queries)} benzersiz SQL sorgusu '{output_file_path}' dosyasına kaydedildi.")

if __name__ == "__main__":
    if len(sys.argv) != 3:
        print("Kullanım: python extract_sql.py <girdi_log_dosyası> <çıktı_dosyası>")
        sys.exit(1)
    
    input_file = sys.argv[1]
    output_file = sys.argv[2]
    extract_unique_queries(input_file, output_file)
