"""
Improved version of the original script.
Cleaned and modularized as per software engineering best practices.
"""

#!/usr/bin/env python3

import re
import sys
from dataclasses import dataclass
from typing import List, Dict, Optional
import psycopg2

# -----------------------------
# Data Classes
# -----------------------------

@dataclass
class IndexInfo:
    """Basic representation of a database index."""
    table_name: str
    index_name: str
    column_name: str
    is_primary: bool
    is_unique: bool

@dataclass
class WhereCondition:
    table: str
    column: str
    index_info: Optional[IndexInfo]

@dataclass
class QueryAnalysis:
    sql: str
    where_conditions: List[WhereCondition]

@dataclass
class SQLAnalysisResult:
    queries: Dict[str, QueryAnalysis]

# -----------------------------
# Global State
# -----------------------------

index_cache: List[IndexInfo] = []

# -----------------------------
# Database Interaction
# -----------------------------

class PostgresIndexFetcher:
    """Handles fetching index metadata from a PostgreSQL database."""

    def __init__(self, connection_params: Dict[str, str]):
        self.connection_params = connection_params
        self.connection = None

    def __enter__(self):
        self.connection = psycopg2.connect(**self.connection_params)
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        if self.connection:
            self.connection.close()

    def fetch_indexes(self, schema: str = 'public') -> List[IndexInfo]:
        query = """
            SELECT
                t.relname AS table_name,
                i.relname AS index_name,
                a.attname AS column_name,
                ix.indisprimary AS is_primary,
                ix.indisunique AS is_unique
            FROM pg_class t
            JOIN pg_index ix ON t.oid = ix.indrelid
            JOIN pg_class i ON i.oid = ix.indexrelid
            JOIN pg_namespace n ON n.oid = t.relnamespace
            JOIN pg_attribute a ON a.attrelid = t.oid AND a.attnum = ANY(ix.indkey)
            WHERE
                t.relkind = 'r'
                AND n.nspname = %s
                AND t.relname NOT LIKE 'pg\\_%'
                AND t.relname NOT LIKE 'sql\\_%'
            ORDER BY t.relname, i.relname;
        """
        with self.connection.cursor() as cursor:
            cursor.execute(query, (schema,))
            return [
                IndexInfo(*row) for row in cursor.fetchall()
            ]

# -----------------------------
# Index Utility
# -----------------------------

def find_index(table: str, column: str) -> Optional[IndexInfo]:
    for index in index_cache:
        if index.table_name == table and index.column_name == column:
            return index
    return None

# -----------------------------
# SQL Analysis Logic
# -----------------------------

def extract_where_clause(query: str) -> str:
    match = re.search(r"WHERE\s+(.*)", query, re.IGNORECASE)
    return match.group(1) if match else ""

def extract_conditions(where_clause: str) -> List[WhereCondition]:
    matches = re.findall(r"(\w+)\.(\w+)", where_clause)
    return [
        WhereCondition(table, column, find_index(table, column))
        for table, column in matches
    ]

def analyze_queries(queries: List[str]) -> SQLAnalysisResult:
    results = {}
    for i, query in enumerate(queries, start=1):
        where_clause = extract_where_clause(query)
        conditions = extract_conditions(where_clause)
        results[f"Query {i}"] = QueryAnalysis(sql=query, where_conditions=conditions)
    return SQLAnalysisResult(queries=results)

# -----------------------------
# File I/O
# -----------------------------

def load_queries(path: str) -> List[str]:
    with open(path, "r", encoding="utf-8") as f:
        return [line.strip() for line in f if line.strip()]

def write_queries(queries: List[str], path: str):
    with open(path, "w", encoding="utf-8") as f:
        for query in sorted(queries):
            f.write(query + "\n")

# -----------------------------
# Entry Point
# -----------------------------

def main():
    if len(sys.argv) != 4:
        print("Usage: python3 diff_sql_generator.py <feature_file> <master_file> <output_file>")
        sys.exit(1)

    feature_file, master_file, output_file = sys.argv[1:4]

    feature_queries = set(load_queries(feature_file))
    master_queries = set(load_queries(master_file))
    diff_queries = feature_queries - master_queries

    write_queries(list(diff_queries), output_file)
    print(f"✅ {len(diff_queries)} new SQL queries written to: {output_file}")

    # Replace with actual DB credentials
    db_config = {
        # 'host': 'localhost',
        # 'dbname': 'mydb',
        # 'user': 'username',
        # 'password': 'password'
    }

    if not db_config:
        print("⚠️ Skipping index analysis due to missing DB configuration.")
        return

    try:
        with PostgresIndexFetcher(db_config) as fetcher:
            global index_cache
            index_cache = fetcher.fetch_indexes()

            print("📌 Index Summary:")
            for idx in index_cache:
                flags = []
                if idx.is_primary: flags.append("PK")
                if idx.is_unique: flags.append("UQ")
                print(f"  - {idx.table_name}.{idx.index_name} ({idx.column_name}) [{' '.join(flags)}]")

    except Exception as e:
        print(f"❌ Database error: {e}")
        return

    analysis = analyze_queries(list(diff_queries))

    for query_name, result in analysis.queries.items():
        print(f"\n🔍 {query_name}")
        print(f"SQL: {result.sql}")
        print("WHERE Conditions:")
        for cond in result.where_conditions:
            print(f"  - Table: {cond.table}, Column: {cond.column}, Indexed: {bool(cond.index_info)}")

if __name__ == "__main__":
    main()
