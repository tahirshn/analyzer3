class ReportGenerator:
    """Generates reports in multiple formats (text, JSON, HTML)"""
    
    @staticmethod
    def generate_report(missing_indexes: Set[QueryCondition], 
                       output_path: str = "index-report",
                       formats: List[str] = ["text", "json", "html"]) -> None:
        """Generate reports in specified formats"""
        
        # Group by table for better organization
        by_table = defaultdict(list)
        for condition in missing_indexes:
            by_table[condition.table_name].append(condition)
        
        # Generate each requested format
        for fmt in formats:
            if fmt == "text":
                ReportGenerator._generate_text_report(by_table, f"{output_path}.txt")
            elif fmt == "json":
                ReportGenerator._generate_json_report(by_table, f"{output_path}.json")
            elif fmt == "html":
                ReportGenerator._generate_html_report(by_table, f"{output_path}.html")

    @staticmethod
    def _generate_text_report(by_table: Dict[str, List[QueryCondition]], output_path: str) -> None:
        """Generate plain text report"""
        with open(output_path, 'w') as f:
            if not by_table:
                f.write("✅ No missing indexes found!\n")
                return
            
            f.write("Missing Database Indexes Report\n")
            f.write("=" * 40 + "\n\n")
            
            for table in sorted(by_table.keys()):
                f.write(f"Table: {table}\n")
                f.write("-" * 40 + "\n")
                
                for cond in sorted(by_table[table], key=lambda c: c.column_name):
                    f.write(f"  Column: {cond.column_name}\n")
                    f.write(f"    Used with operator: {cond.operator}\n")
                    f.write(f"    Recommended index:\n")
                    f.write(f"      CREATE INDEX idx_{table}_{cond.column_name} ON {table}({cond.column_name});\n\n")
                
                f.write("\n")

    @staticmethod
    def _generate_json_report(by_table: Dict[str, List[QueryCondition]], output_path: str) -> None:
        """Generate JSON report"""
        import json
        
        report_data = {
            "timestamp": datetime.datetime.now().isoformat(),
            "missing_indexes": []
        }
        
        for table, conditions in sorted(by_table.items()):
            for cond in sorted(conditions, key=lambda c: c.column_name):
                report_data["missing_indexes"].append({
                    "table": table,
                    "column": cond.column_name,
                    "operator": cond.operator,
                    "recommendation": {
                        "sql": f"CREATE INDEX idx_{table}_{cond.column_name} ON {table}({cond.column_name})",
                        "priority": "high" if cond.operator in {'<', '<=', '>', '>=', 'BETWEEN'} else "medium"
                    }
                })
        
        with open(output_path, 'w') as f:
            json.dump(report_data, f, indent=2)

    @staticmethod
    def _generate_html_report(by_table: Dict[str, List[QueryCondition]], output_path: str) -> None:
        """Generate HTML report"""
        with open(output_path, 'w') as f:
            f.write("""<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Missing Indexes Report</title>
    <style>
        body { font-family: Arial, sans-serif; line-height: 1.6; margin: 20px; }
        h1 { color: #333; }
        h2 { color: #444; margin-top: 30px; border-bottom: 1px solid #eee; padding-bottom: 5px; }
        .index { background: #f9f9f9; padding: 10px; margin: 10px 0; border-left: 4px solid #ccc; }
        .sql { font-family: monospace; background: #f0f0f0; padding: 5px; display: inline-block; }
        .priority-high { border-left-color: #e74c3c; }
        .priority-medium { border-left-color: #f39c12; }
        .summary { background: #f5f5f5; padding: 15px; margin-bottom: 20px; }
    </style>
</head>
<body>
    <h1>Missing Database Indexes Report</h1>
    <div class="summary">
        Generated: {timestamp}<br>
        Total missing indexes found: {count}
    </div>
""".format(
    timestamp=datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
    count=sum(len(conditions) for conditions in by_table.values())
))

            if not by_table:
                f.write("""<div style="color: green; font-size: 1.2em;">
                    ✅ No missing indexes found!
                </div>""")
            else:
                for table in sorted(by_table.keys()):
                    f.write(f'<h2>Table: {table}</h2>\n')
                    
                    for cond in sorted(by_table[table], key=lambda c: c.column_name):
                        priority = "high" if cond.operator in {'<', '<=', '>', '>=', 'BETWEEN'} else "medium"
                        f.write(f"""<div class="index priority-{priority}">
    <strong>Column:</strong> {cond.column_name}<br>
    <strong>Operator:</strong> {cond.operator}<br>
    <strong>Recommended index:</strong><br>
    <div class="sql">CREATE INDEX idx_{table}_{cond.column_name} ON {table}({cond.column_name});</div>
</div>
""")

            f.write("""</body>
</html>""")

# Update the main function to support multiple output formats
def main():
    parser = argparse.ArgumentParser(
        description="Enterprise Database Index Analyzer for Spring Data JPA (Python 3.6)"
    )
    parser.add_argument("--repo-path", default=".",
                       help="Path to code repository")
    parser.add_argument("--flyway-path", default="src/main/resources/db/migration",
                       help="Path to Flyway migrations")
    parser.add_argument("--fail-threshold", type=int, default=0,
                       help="Maximum allowed missing indexes before failing")
    parser.add_argument("--formats", nargs="+", default=["text"],
                       choices=["text", "json", "html"],
                       help="Output formats for the report")
    parser.add_argument("--output-prefix", default="index-report",
                       help="Prefix for output files")
    
    args = parser.parse_args()

    # Dependency injection
    metadata_extractor = KotlinEntityExtractor()
    query_analyzer = SpringDataQueryAnalyzer(metadata_extractor)
    analyzer = PipelineIndexAnalyzer(metadata_extractor, query_analyzer)
    
    analyzer.set_fail_threshold(args.fail_threshold)
    success = analyzer.run_pipeline_analysis(args.repo_path, args.flyway_path)
    
    # Generate reports
    query_conditions = query_analyzer.analyze_repositories(args.repo_path)
    missing_indexes = analyzer._identify_missing_indexes(query_conditions)
    ReportGenerator.generate_report(
        missing_indexes,
        output_path=args.output_prefix,
        formats=args.formats
    )
    
    if not success:
        exit(1)
