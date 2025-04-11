import re
import json
from typing import List, Dict, Set

class SimpleSQLAnalyzer:
    def __init__(self):
        self.tables_columns: Dict[str, Set[str]] = {}

    def analyze_sql_file(self, file_path: str) -> Dict[str, List[str]]:
        """SQL dosyasını okuyup tablo ve sütunları çıkarır"""
        with open(file_path, 'r') as file:
            for line in file:
                line = line.strip()
                if line and not line.startswith(('--', '/*', '*/')):
                    self._analyze_line(line)
        
        # Set'leri listeye çevir
        result = {table: sorted(columns) for table, columns in self.tables_columns.items()}
        return {"tables": [{"table_name": k, "columns": v} for k, v in result.items()]}

    def _analyze_line(self, line: str):
        """Bir SQL satırını analiz eder"""
        # FROM ve JOIN'den tabloları bul
        self._extract_tables_from_from(line)
        self._extract_tables_from_join(line)
        
        # Sütunları bul (tablo.sütun formatında)
        column_matches = re.finditer(r'(\w+)\.(\w+)', line)
        for match in column_matches:
            table = match.group(1)
            column = match.group(2)
            self._add_column(table, column)

    def _extract_tables_from_from(self, line: str):
        """FROM ifadesinden tabloları çıkarır"""
        from_match = re.search(r'FROM\s+(\w+)(?:\s+(\w+))?', line, re.IGNORECASE)
        if from_match:
            table = from_match.group(1)
            alias = from_match.group(2) if from_match.group(2) else table
            self._add_table(table, alias)

    def _extract_tables_from_join(self, line: str):
        """JOIN ifadelerinden tabloları çıkarır"""
        join_matches = re.finditer(
            r'(?:INNER\s+|LEFT\s+|RIGHT\s+|FULL\s+)?JOIN\s+(\w+)(?:\s+(\w+))?',
            line, re.IGNORECASE
        )
        for match in join_matches:
            table = match.group(1)
            alias = match.group(2) if match.group(2) else table
            self._add_table(table, alias)

    def _add_table(self, table: str, alias: str):
        """Tabloyu listeye ekler (sütun olmadan)"""
        if table not in self.tables_columns:
            self.tables_columns[table] = set()

    def _add_column(self, table: str, column: str):
        """Tabloya sütun ekler"""
        if table in self.tables_columns:
            self.tables_columns[table].add(column)
        else:
            # Tablo adı olmayan sütun referansı (belki alias)
            # Bu durumda tabloyu da ekliyoruz
            self.tables_columns[table] = {column}

# Kullanım örneği
if __name__ == "__main__":
    analyzer = SimpleSQLAnalyzer()
    
    # SQL dosyasını analiz et
    result = analyzer.analyze_sql_file("sql_queries.sql")  # Dosya adını değiştirin
    
    # Sonuçları JSON olarak yazdır
    print(json.dumps(result, indent=2))
    
    # Veya basit bir şekilde yazdır
    print("\nKullanılan Tablolar ve Sütunlar:")
    for table in result["tables"]:
        print(f"\nTablo: {table['table_name']}")
        print("Sütunlar:", ", ".join(table["columns"]))
