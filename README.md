from pathlib import Path
import argparse
import re


def extract_sql_queries(input_path: Path) -> set[str]:
    with input_path.open("r", encoding="utf-8") as f:
        lines = f.readlines()

    sql_lines = set()
    index = 0
    total_lines = len(lines)

    while index < total_lines:
        line = lines[index]

        if "org.hibernate.SQL" in line:
            index += 1
            parts = re.split("org.hibernate.SQL", line)[1]
            raw_sql_split = re.split(r'[:\[]', parts)
            if len(raw_sql_split) > 1 and raw_sql_split[1].strip():
                sql_lines.add(raw_sql_split[1].strip())
            else:
                sql_parts = []
                while index < total_lines:
                    next_line = lines[index]
                    if "[" in next_line:
                        sql_parts.append(next_line.split("[")[0].strip())
                        index += 1
                        break
                    else:
                        sql_parts.append(next_line.strip())
                        index += 1
                joined_sql = ' '.join(sql_parts).strip()
                if joined_sql:
                    sql_lines.add(joined_sql)
        else:
            index += 1

    return sql_lines


def write_output(output_path: Path, sql_queries: set[str]) -> None:
    with output_path.open("w", encoding="utf-8") as f:
        for sql in sorted(sql_queries):
            f.write(sql + "\n")


def main():
    parser = argparse.ArgumentParser(description="Extract unique SQL queries from Hibernate logs.")
    parser.add_argument("input_file", type=Path, help="Path to input log file.")
    parser.add_argument("output_file", type=Path, help="Path to output SQL file.")
    args = parser.parse_args()

    sql_queries = extract_sql_queries(args.input_file)
    write_output(args.output_file, sql_queries)

    print(f"✅ Found {len(sql_queries)} unique SQL queries.")
    print(f"📄 Output written to: {args.output_file}")


if __name__ == "__main__":
    main()
