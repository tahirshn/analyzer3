import os
import re
from typing import Dict, Any

# Regex patterns with improved accuracy
create_index_pattern = re.compile(
    r'CREATE\s+(UNIQUE\s+)?(CONCURRENTLY\s+)?(INDEX\s+)(IF\s+NOT\s+EXISTS\s+)?([^\s]+)\s+ON\s+([^\s(]+)\s*\(([^)]+)\)',
    re.IGNORECASE | re.DOTALL
)

drop_index_pattern = re.compile(
    r'DROP\s+INDEX\s+(CONCURRENTLY\s+)?(IF\s+EXISTS\s+)?([^\s;]+)',
    re.IGNORECASE | re.DOTALL
)

pk_inline_pattern = re.compile(
    r'PRIMARY\s+KEY\s*\(([^)]+)\)',
    re.IGNORECASE | re.DOTALL
)

# Improved column-level PK pattern - matches column definition ending with PRIMARY KEY
pk_column_constraint_pattern = re.compile(
    r'(\b\w+\b)\s+.*?(?<!\S)PRIMARY\s+KEY\b',
    re.IGNORECASE | re.DOTALL
)

alter_add_pk_pattern = re.compile(
    r'ALTER\s+TABLE\s+([^\s]+)\s+ADD\s+(CONSTRAINT\s+([^\s]+)\s+)?PRIMARY\s+KEY\s*\(([^)]+)\)',
    re.IGNORECASE | re.DOTALL
)

alter_drop_constraint_pattern = re.compile(
    r'ALTER\s+TABLE\s+([^\s]+)\s+DROP\s+CONSTRAINT\s+([^\s;]+)',
    re.IGNORECASE | re.DOTALL
)

def normalize_sql(sql: str) -> str:
    """Cleans up SQL by removing comments and normalizing whitespace."""
    # Remove SQL comments
    sql = re.sub(r'--.*?$', '', sql, flags=re.MULTILINE)
    sql = re.sub(r'/\*.*?\*/', '', sql, flags=re.DOTALL)
    # Normalize whitespace
    sql = re.sub(r'\s+', ' ', sql)
    # Ensure spaces around parentheses and commas
    sql = re.sub(r'\s*([(),])\s*', r'\1 ', sql)
    return sql.strip()

def analyze_indexes_and_primary_keys(sql_directory: str):
    indexes: Dict[str, Dict[str, Any]] = {}

    for root, _, files in os.walk(sql_directory):
        for file in sorted(files):
            if file.endswith(".sql"):
                full_path = os.path.join(root, file)
                with open(full_path, 'r', encoding='utf-8') as f:
                    sql_content = f.read()
                    # Split statements while handling semicolons in strings
                    statements = [stmt.strip() for stmt in re.split(r';(?![^(]*\))', sql_content) if stmt.strip()]

                    for statement in statements:
                        statement_clean = normalize_sql(statement)

                        # CREATE INDEX variations
                        create_match = create_index_pattern.search(statement_clean)
                        if create_match:
                            is_unique = bool(create_match.group(1))
                            concurrently = bool(create_match.group(2))
                            if_not_exists = bool(create_match.group(4))
                            index_name = create_match.group(5)
                            table_name = create_match.group(6)
                            columns = create_match.group(7).replace(" ", "")
                            
                            index_type = []
                            if is_unique:
                                index_type.append("UNIQUE")
                            if concurrently:
                                index_type.append("CONCURRENTLY")
                            index_type.append("INDEX")
                            if if_not_exists:
                                index_type.append("IF NOT EXISTS")
                            
                            indexes[index_name] = {
                                'index_name': index_name,
                                'table': table_name,
                                'columns': columns,
                                'type': ' '.join(index_type),
                                'created_in': file,
                                'dropped_in': None,
                            }

                        # PRIMARY KEY detection
                        if "CREATE TABLE" in statement_clean:
                            # Case 1: Column-level PRIMARY KEY (column type PRIMARY KEY)
                            column_pk_matches = pk_column_constraint_pattern.finditer(statement_clean)
                            for match in column_pk_matches:
                                column_name = match.group(1)
                                # Only proceed if this is actually a column name (not CREATE or other keywords)
                                if column_name.upper() not in ['CREATE', 'TABLE', 'IF', 'NOT', 'EXISTS']:
                                    table_match = re.search(r'CREATE\s+TABLE\s+(?:IF\s+NOT\s+EXISTS\s+)?([^\s(]+)', 
                                                          statement_clean, re.IGNORECASE)
                                    table_name = table_match.group(1) if table_match else '?'
                                    index_name = f"{table_name}_pkey"
                                    indexes[index_name] = {
                                        'index_name': index_name,
                                        'table': table_name,
                                        'columns': column_name,
                                        'type': 'PRIMARY KEY (column)',
                                        'created_in': file,
                                        'dropped_in': None,
                                    }

                            # Case 2: Table-level PRIMARY KEY (PRIMARY KEY (columns))
                            pk_inline = pk_inline_pattern.search(statement_clean)
                            if pk_inline and "ALTER TABLE" not in statement_clean:
                                table_match = re.search(r'CREATE\s+TABLE\s+(?:IF\s+NOT\s+EXISTS\s+)?([^\s(]+)', 
                                                      statement_clean, re.IGNORECASE)
                                table_name = table_match.group(1) if table_match else '?'
                                columns = pk_inline.group(1).replace(" ", "")
                                index_name = f"{table_name}_pkey"
                                indexes[index_name] = {
                                    'index_name': index_name,
                                    'table': table_name,
                                    'columns': columns,
                                    'type': 'PRIMARY KEY (table)',
                                    'created_in': file,
                                    'dropped_in': None,
                                }

                        # Case 3: ALTER TABLE ADD PRIMARY KEY
                        alter_add_pk = alter_add_pk_pattern.search(statement_clean)
                        if alter_add_pk:
                            table_name = alter_add_pk.group(1)
                            constraint_name = alter_add_pk.group(3) or f"{table_name}_pkey"
                            columns = alter_add_pk.group(4).replace(" ", "")
                            indexes[constraint_name] = {
                                'index_name': constraint_name,
                                'table': table_name,
                                'columns': columns,
                                'type': 'PRIMARY KEY (alter)',
                                'created_in': file,
                                'dropped_in': None,
                            }

                        # DROP INDEX variations
                        drop_match = drop_index_pattern.search(statement_clean)
                        if drop_match:
                            index_name = drop_match.group(3)
                            if index_name in indexes:
                                indexes[index_name]['dropped_in'] = file

                        # ALTER TABLE DROP CONSTRAINT
                        drop_constraint = alter_drop_constraint_pattern.search(statement_clean)
                        if drop_constraint:
                            constraint_name = drop_constraint.group(2)
                            if constraint_name in indexes:
                                indexes[constraint_name]['dropped_in'] = file

    # Print result table
    print(f"{'Index Name':40} | {'Table':20} | {'Columns':30} | {'Type':20} | {'Status':10} | {'Created In':20} | {'Dropped In':20}")
    print("-" * 180)
    for info in sorted(indexes.values(), key=lambda x: (x['table'], x['index_name'])):
        status = "Dropped" if info['dropped_in'] else "Active"
        print(f"{info['index_name']:40} | {info['table']:20} | {info['columns']:30} | {info['type']:20} | {status:10} | {info['created_in']:20} | {info['dropped_in'] or '-':20}")

if __name__ == "__main__":
    analyze_indexes_and_primary_keys("path/to/your/sql/files")
