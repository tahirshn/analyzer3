import os
import re

# Regex patterns
create_index_pattern = re.compile(r'CREATE\s+(UNIQUE\s+)?INDEX\s+([^\s]+)\s+ON\s+([^\s(]+)\s*\(([^)]+)\)', re.IGNORECASE)
drop_index_pattern = re.compile(r'DROP\s+INDEX\s+(IF\s+EXISTS\s+)?([^\s;]+)', re.IGNORECASE)
pk_inline_pattern = re.compile(r'PRIMARY\s+KEY\s*\(([^)]+)\)', re.IGNORECASE)
alter_add_pk_pattern = re.compile(r'ALTER\s+TABLE\s+([^\s]+)\s+ADD\s+CONSTRAINT\s+([^\s]+)\s+PRIMARY\s+KEY\s*\(([^)]+)\)', re.IGNORECASE)
alter_drop_constraint_pattern = re.compile(r'ALTER\s+TABLE\s+([^\s]+)\s+DROP\s+CONSTRAINT\s+([^\s;]+)', re.IGNORECASE)

def normalize_sql(sql: str) -> str:
    """Cleans up SQL by removing line breaks and extra spaces."""
    return re.sub(r'\s+', ' ', sql).strip()

def analyze_indexes_and_primary_keys(sql_directory: str):
    indexes = {}

    for root, _, files in os.walk(sql_directory):
        for file in sorted(files):
            if file.endswith(".sql"):
                full_path = os.path.join(root, file)
                with open(full_path, 'r', encoding='utf-8') as f:
                    sql_content = f.read()
                    statements = sql_content.split(";")

                    for statement in statements:
                        statement_clean = normalize_sql(statement)

                        # CREATE INDEX
                        create_match = create_index_pattern.search(statement_clean)
                        if create_match:
                            is_unique = bool(create_match.group(1))
                            index_name = create_match.group(2)
                            table_name = create_match.group(3)
                            columns = create_match.group(4).replace(" ", "")
                            indexes[index_name] = {
                                'index_name': index_name,
                                'table': table_name,
                                'columns': columns,
                                'type': 'UNIQUE INDEX' if is_unique else 'INDEX',
                                'created_in': file,
                                'dropped_in': None,
                            }

                        # PRIMARY KEY inline
                        pk_inline = pk_inline_pattern.search(statement_clean)
                        if pk_inline and "ALTER TABLE" not in statement_clean:
                            columns = pk_inline.group(1).replace(" ", "")
                            index_name = f"{file}_inline_pk_{statements.index(statement)+1}"
                            indexes[index_name] = {
                                'index_name': index_name,
                                'table': '?',
                                'columns': columns,
                                'type': 'PRIMARY KEY',
                                'created_in': file,
                                'dropped_in': None,
                            }

                        # ALTER TABLE ADD CONSTRAINT ... PRIMARY KEY
                        alter_add_pk = alter_add_pk_pattern.search(statement_clean)
                        if alter_add_pk:
                            table_name = alter_add_pk.group(1)
                            constraint_name = alter_add_pk.group(2)
                            columns = alter_add_pk.group(3).replace(" ", "")
                            indexes[constraint_name] = {
                                'index_name': constraint_name,
                                'table': table_name,
                                'columns': columns,
                                'type': 'PRIMARY KEY',
                                'created_in': file,
                                'dropped_in': None,
                            }

                        # DROP INDEX
                        drop_match = drop_index_pattern.search(statement_clean)
                        if drop_match:
                            index_name = drop_match.group(2)
                            if index_name in indexes:
                                indexes[index_name]['dropped_in'] = file

                        # ALTER TABLE DROP CONSTRAINT
                        drop_constraint = alter_drop_constraint_pattern.search(statement_clean)
                        if drop_constraint:
                            constraint_name = drop_constraint.group(2)
                            if constraint_name in indexes:
                                indexes[constraint_name]['dropped_in'] = file

    # Print result table
    print(f"{'Index Name':30} | {'Table':15} | {'Columns':20} | {'Type':13} | {'Status':10} | {'Created In':20} | {'Dropped In':20}")
    print("-" * 140)
    for info in indexes.values():
        status = "Dropped" if info['dropped_in'] else "Active"
        print(f"{info['index_name']:30} | {info['table']:15} | {info['columns']:20} | {info['type']:13} | {status:10} | {info['created_in']:20} | {info['dropped_in'] or '-':20}")
