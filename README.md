import re

class SqlParser:
    def __init__(self):
        self.join_columns = set()
        self.where_columns = set()
        self.tables_involved = set()

    # SQL sorgusunu analiz et
    def parse_sql(self, query: str):
        self.join_columns.clear()
        self.where_columns.clear()
        self.tables_involved.clear()

        # JOIN'lerde kullanılan kolonları çıkar
        join_pattern = r"join\s+(\w+)\s+on\s+(\w+)\.(\w+)\s*=\s*(\w+)\.(\w+)"
        join_matches = re.findall(join_pattern, query, re.IGNORECASE)

        for match in join_matches:
            table1 = match[1]  # 1. kolon
            column1 = match[2]  # 2. kolon
            table2 = match[3]  # 3. kolon
            column2 = match[4]  # 4. kolon
            self.join_columns.add(f"{table1}.{column1}")
            self.join_columns.add(f"{table2}.{column2}")
        
        # WHERE koşulunda kullanılan kolonları çıkar
        where_columns = self.extract_where_columns(query)
        self.where_columns.update(where_columns)

        # Tabloları tespit et
        self.tables_involved.update(self.get_tables(query))

        # Sonuçları yazdır
        print("Tables involved:", self.tables_involved)
        print("JOIN columns:", self.join_columns)
        print("WHERE columns:", self.where_columns)

    # WHERE koşulundaki kolonları çıkart
    def extract_where_columns(self, query: str):
        # WHERE bloğunu al
        where_pattern = r"where\s+(.*?)(group|order|having|limit|;|$)"
        match = re.search(where_pattern, query, re.IGNORECASE)

        where_columns = set()

        if match:
            where_clause = match.group(1)

            # Kolonları çıkar, hem AND hem de OR koşulları için geçerli
            where_columns.update(re.findall(r"\b\w+\.\w+\b", where_clause))

        return where_columns

    # SQL sorgusunda kullanılan tablolara bak
    def get_tables(self, query: str):
        table_pattern = r"(from|join)\s+(\w+)"
        table_matches = re.findall(table_pattern, query, re.IGNORECASE)
        return {match[1] for match in table_matches}

# Dosyadan sorguları oku
def parse_queries_from_file(file_path: str):
    with open(file_path, 'r') as file:
        queries = file.read().split(';')  # SQL sorgularını ';' ile ayır

    # Sorguları analiz et
    parser = SqlParser()
    for query in queries:
        query = query.strip()
        if query:
            print(f"Parsing query: {query[:50]}...")  # Sadece sorgunun başını göster
            parser.parse_sql(query)
            print("\n" + "="*50 + "\n")

# Örnek kullanım
if __name__ == "__main__":
    file_path = 'output_raw_sql.txt'  # SQL sorgularını içeren dosya yolu
    parse_queries_from_file(file_path)
