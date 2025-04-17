import os

def save_analysis_report(result: SQLAnalysisResult, output_dir: str):
    """
    Save detailed SQL analysis results for SELECT queries into files

    Args:
        result: SQLAnalysisResult object containing parsed query data
        output_dir: directory where reports will be saved
    """
    os.makedirs(output_dir, exist_ok=True)

    for idx, (query_name, analysis) in enumerate(result.queries.items(), 1):
        if not analysis.sql.strip().lower().startswith("select"):
            continue  # Skip non-SELECT queries

        filename = os.path.join(output_dir, f"query_{idx}.txt")
        with open(filename, "w") as f:
            f.write(f"Query Name: {query_name}\n")
            f.write(f"Original SQL:\n{analysis.sql.strip()}\n\n")

            f.write("WHERE Conditions:\n")
            if analysis.where_conditions:
                for condition in analysis.where_conditions:
                    is_indexed = condition.indexInfo is not None
                    f.write(f"- Table: {condition.table}\n")
                    f.write(f"  Column: {condition.column}\n")
                    f.write(f"  Indexed: {'YES' if is_indexed else 'NO'}\n\n")
            else:
                f.write("No WHERE conditions found.\n")

----------------------------------
    try:
        with PostgresIndexFetcher(db_config) as fetcher:
            indexes = fetcher.get_user_indexes()

            # Optional: print basic index info
            print("Table Indexes:")
            for idx in indexes:
                print(f"{idx.table_name}.{idx.index_name} ({idx.column_name}) "
                      f"{'PK' if idx.is_primary else ''}{'UQ' if idx.is_unique else ''}")

            # Perform query analysis after indexes are loaded
            analysis_results = analyze_sql_file(diff)

            # Save the report to Spring Boot's /build/sql_analysis_report directory
            report_dir = os.path.join("build", "sql_analysis_report")
            save_analysis_report(analysis_results, report_dir)

            print(f"\n✅ Analysis report saved in: {report_dir}")

    except Exception as e:
        print(f"Error: {e}")
