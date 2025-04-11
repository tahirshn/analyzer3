import re
import json
from datetime import datetime
from typing import List, Dict, Any

class SQLQueryAnalyzer:
    def __init__(self):
        self.queries = []
        self.current_query = None
        self.query_id = 0

    def analyze_sql_file(self, file_path: str) -> Dict[str, Any]:
        """SQL dosyasını okuyup analiz eder"""
        with open(file_path, 'r') as file:
            sql_content = file.read()
        
        # Sorguları ayır (noktalı virgül ile biten ifadeler)
        raw_queries = [q.strip() for q in sql_content.split(';') if q.strip()]
        
        for raw_query in raw_queries:
            self.query_id += 1
            self.current_query = {
                "query_id": self.query_id,
                "query_text": raw_query + ';',
                "tables": [],
                "where_conditions": [],
                "group_by": [],
                "having_condition": None,
                "order_by": []
            }
            self._analyze_query(raw_query)
            self.queries.append(self.current_query)
        
        return self._generate_output()

    def _analyze_query(self, query: str):
        """Tek bir SQL sorgusunu analiz eder"""
        self._extract_tables_and_joins(query)
        self._extract_where_conditions(query)
        self._extract_group_by(query)
        self._extract_having(query)
        self._extract_order_by(query)

    def _extract_tables_and_joins(self, query: str):
        """FROM ve JOIN ifadelerinden tablo bilgilerini çıkarır"""
        # FROM kısmını bul
        from_match = re.search(r'FROM\s+([^\s(]+)(?:\s+([^\s,]+))?', query, re.IGNORECASE)
        if from_match:
            table_name = from_match.group(1)
            alias = from_match.group(2) if from_match.group(2) else table_name
            self._add_table(table_name, alias)

        # JOIN ifadelerini bul
        join_matches = re.finditer(
            r'(?:INNER\s+|LEFT\s+|RIGHT\s+|FULL\s+)?JOIN\s+([^\s]+)(?:\s+([^\s]+))?\s+ON\s+([^)]+)',
            query, re.IGNORECASE
        )
        
        for match in join_matches:
            table_name = match.group(1)
            alias = match.group(2) if match.group(2) else table_name
            join_condition = match.group(3)
            
            table_info = self._add_table(table_name, alias)
            joined_table = self._extract_joined_table_from_condition(join_condition, alias)
            
            if joined_table:
                table_info["join_conditions"].append({
                    "joined_table": joined_table["table"],
                    "on_condition": join_condition,
                    "joined_table_alias": joined_table["alias"]
                })

    def _add_table(self, table_name: str, alias: str) -> Dict[str, Any]:
        """Tabloyu sorguya ekler veya varsa günceller"""
        # Tablo zaten eklenmiş mi kontrol et
        for table in self.current_query["tables"]:
            if table["alias"] == alias:
                return table
        
        # Yeni tablo ekle
        table_info = {
            "table_name": table_name,
            "alias": alias,
            "columns": self._extract_columns_from_query(alias),
            "join_conditions": []
        }
        self.current_query["tables"].append(table_info)
        return table_info

    def _extract_columns_from_query(self, table_alias: str) -> List[str]:
        """SELECT ifadesinden belirli bir tabloya ait sütunları çıkarır"""
        select_match = re.search(r'SELECT\s+(.*?)\s+FROM', self.current_query["query_text"], re.IGNORECASE | re.DOTALL)
        if not select_match:
            return []
        
        select_clause = select_match.group(1)
        columns = []
        pattern = re.compile(rf'{table_alias}\.(\w+)')
        
        for match in pattern.finditer(select_clause):
            columns.append(match.group(1))
        
        return list(set(columns))  # Tekrarları kaldır

    def _extract_joined_table_from_condition(self, condition: str, current_alias: str) -> Dict[str, str]:
        """JOIN koşulundaki diğer tabloyu ve takma adını çıkarır"""
        # Koşuldaki tabloları bul (örnek: a.id = b.id)
        parts = re.split(r'=|!=|>|<|>=|<=|<>|\s+', condition)
        tables_in_condition = []
        
        for part in parts:
            table_match = re.match(r'(\w+)\.\w+', part.strip())
            if table_match and table_match.group(1) != current_alias:
                tables_in_condition.append(table_match.group(1))
        
        if tables_in_condition:
            # İlgili tabloyu bul
            for table in self.current_query["tables"]:
                if table["alias"] in tables_in_condition:
                    return {"table": table["table_name"], "alias": table["alias"]}
        
        return None

    def _extract_where_conditions(self, query: str):
        """WHERE koşullarını çıkarır"""
        where_match = re.search(r'WHERE\s+(.*?)(?:\s+GROUP\s+BY|\s+HAVING|\s+ORDER\s+BY|;|$)', query, re.IGNORECASE | re.DOTALL)
        if not where_match:
            return
        
        where_clause = where_match.group(1)
        conditions = re.split(r'\s+AND\s+|\s+OR\s+', where_clause, flags=re.IGNORECASE)
        
        for condition in conditions:
            condition = condition.strip()
            if not condition:
                continue
                
            # Tablo ve sütun bilgisini çıkar
            table_match = re.match(r'(\w+)\.(\w+)\s*(.*)', condition)
            if table_match:
                table_alias = table_match.group(1)
                column = table_match.group(2)
                rest = table_match.group(3)
                
                # İlgili tabloyu bul
                table_name = None
                for table in self.current_query["tables"]:
                    if table["alias"] == table_alias:
                        table_name = table["table_name"]
                        break
                
                if table_name:
                    self.current_query["where_conditions"].append({
                        "table": table_name,
                        "alias": table_alias,
                        "column": column,
                        "condition": rest
                    })

    def _extract_group_by(self, query: str):
        """GROUP BY ifadelerini çıkarır"""
        group_match = re.search(r'GROUP\s+BY\s+(.*?)(?:\s+HAVING|\s+ORDER\s+BY|;|$)', query, re.IGNORECASE | re.DOTALL)
        if not group_match:
            return
        
        group_clause = group_match.group(1)
        items = [item.strip() for item in group_clause.split(',')]
        
        for item in items:
            # Tablo ve sütun bilgisini çıkar
            table_match = re.match(r'(\w+)\.(\w+)', item)
            if table_match:
                table_alias = table_match.group(1)
                column = table_match.group(2)
                
                # İlgili tabloyu bul
                table_name = None
                for table in self.current_query["tables"]:
                    if table["alias"] == table_alias:
                        table_name = table["table_name"]
                        break
                
                if table_name:
                    self.current_query["group_by"].append({
                        "table": table_name,
                        "alias": table_alias,
                        "column": column
                    })

    def _extract_having(self, query: str):
        """HAVING koşulunu çıkarır"""
        having_match = re.search(r'HAVING\s+(.*?)(?:\s+ORDER\s+BY|;|$)', query, re.IGNORECASE | re.DOTALL)
        if not having_match:
            return
        
        having_clause = having_match.group(1)
        self.current_query["having_condition"] = {
            "expression": having_clause,
            "aggregated_columns": self._extract_aggregated_columns(having_clause)
        }

    def _extract_aggregated_columns(self, expression: str) -> List[Dict[str, str]]:
        """Aggregate fonksiyonlarındaki sütunları çıkarır (SUM, AVG, COUNT vb.)"""
        agg_matches = re.finditer(r'(SUM|AVG|COUNT|MIN|MAX)\((\w+\.\w+)\)', expression, re.IGNORECASE)
        result = []
        
        for match in agg_matches:
            operation = match.group(1).upper()
            full_column = match.group(2)
            
            table_match = re.match(r'(\w+)\.(\w+)', full_column)
            if table_match:
                table_alias = table_match.group(1)
                column = table_match.group(2)
                
                # İlgili tabloyu bul
                table_name = None
                for table in self.current_query["tables"]:
                    if table["alias"] == table_alias:
                        table_name = table["table_name"]
                        break
                
                if table_name:
                    result.append({
                        "table": table_name,
                        "alias": table_alias,
                        "column": column,
                        "operation": operation
                    })
        
        return result

    def _extract_order_by(self, query: str):
        """ORDER BY ifadelerini çıkarır"""
        order_match = re.search(r'ORDER\s+BY\s+(.*?)(?:;|$)', query, re.IGNORECASE | re.DOTALL)
        if not order_match:
            return
        
        order_clause = order_match.group(1)
        items = [item.strip() for item in order_clause.split(',')]
        
        for item in items:
            # Yönü belirle (ASC/DESC)
            direction = "ASC"
            if re.search(r'\s+DESC$', item, re.IGNORECASE):
                direction = "DESC"
                item = re.sub(r'\s+DESC$', '', item, flags=re.IGNORECASE).strip()
            elif re.search(r'\s+ASC$', item, re.IGNORECASE):
                item = re.sub(r'\s+ASC$', '', item, flags=re.IGNORECASE).strip()
            
            # Tablo ve sütun bilgisini çıkar
            table_match = re.match(r'(\w+)\.(\w+)', item)
            if table_match:
                table_alias = table_match.group(1)
                column = table_match.group(2)
                
                # İlgili tabloyu bul
                table_name = None
                for table in self.current_query["tables"]:
                    if table["alias"] == table_alias:
                        table_name = table["table_name"]
                        break
                
                if table_name:
                    self.current_query["order_by"].append({
                        "table": table_name,
                        "alias": table_alias,
                        "column": column,
                        "direction": direction
                    })
            else:
                # Sütun adı veya alias olabilir
                self.current_query["order_by"].append({
                    "expression": item,
                    "direction": direction
                })

    def _generate_output(self) -> Dict[str, Any]:
        """Sonuçları JSON formatında oluşturur"""
        unique_tables = set()
        for query in self.queries:
            for table in query["tables"]:
                unique_tables.add(table["table_name"])
        
        return {
            "queries": self.queries,
            "metadata": {
                "total_queries": len(self.queries),
                "unique_tables": list(unique_tables),
                "timestamp": datetime.utcnow().isoformat() + "Z"
            }
        }

# Kullanım örneği
if __name__ == "__main__":
    analyzer = SQLQueryAnalyzer()
    
    # SQL dosyasını analiz et
    result = analyzer.analyze_sql_file("sql_queries.sql")  # Dosya adını değiştirin
    
    # Sonuçları JSON olarak kaydet
    with open("analysis_result.json", "w") as outfile:
        json.dump(result, outfile, indent=2)
    
    print("Analiz tamamlandı. Sonuçlar 'analysis_result.json' dosyasına kaydedildi.")
