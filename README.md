import json
from dataclasses import dataclass
from pathlib import Path
from typing import List, Dict


@dataclass
class ColumnInfo:
    name: str


@dataclass
class IndexInfo:
    name: str
    columns: List[ColumnInfo]
    is_unique: bool
    is_primary: bool


@dataclass
class TableIndexInfo:
    table_name: str
    indexes: List[IndexInfo]


def read_indexes_from_json(file_path: Path) -> List[TableIndexInfo]:
    if not file_path.exists():
        raise FileNotFoundError(f"File not found: {file_path}")

    with file_path.open("r", encoding="utf-8") as f:
        flat_data = json.load(f)

    # Grup: table -> index -> IndexInfo
    table_map: Dict[str, Dict[str, IndexInfo]] = {}

    for row in flat_data:
        table = row["table"]
        index = row["index"]
        column_name = row["column"]
        is_unique = row["isUnique"]
        is_primary = row["isPrimary"]

        if table not in table_map:
            table_map[table] = {}

        if index not in table_map[table]:
            table_map[table][index] = IndexInfo(
                name=index,
                columns=[],
                is_unique=is_unique,
                is_primary=is_primary
            )

        # Aynı kolonu tekrar eklememek için kontrol
        if not any(c.name == column_name for c in table_map[table][index].columns):
            table_map[table][index].columns.append(ColumnInfo(name=column_name))

    # Sonuç listesini oluştur
    result: List[TableIndexInfo] = [
        TableIndexInfo(table_name=table, indexes=list(indexes.values()))
        for table, indexes in table_map.items()
    ]

    return result


def main():
    json_path = Path("build/sql_logs/indexes.json")

    try:
        tables = read_indexes_from_json(json_path)
        print(f"Loaded index metadata for {len(tables)} tables.")
    except Exception as e:
        print(f"Error: {e}")


if __name__ == "__main__":
    main()
