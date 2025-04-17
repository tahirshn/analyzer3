import re
from dataclasses import dataclass
from typing import List, Dict


@dataclass
class WhereCondition:
    """
    Represents a single WHERE condition with its table and column.
    """
    column: str
    table: str


@dataclass
class QueryAnalysis:
    """
    Represents the analysis of a single SQL query, storing the query and its WHERE conditions.
    """
    sql: str
    where_conditions: List[WhereCondition]


@dataclass
class SQLAnalysisResult:
    """
    Represents the overall result of SQL analysis, containing all query analyses.
    """
    queries: Dict[str, QueryAnalysis]  # Mapping of query name to its analysis


def extract_where_conditions(query: str) -> List[WhereCondition]:
    """
    Extracts all table.column pairs from the WHERE clause of a SQL query.
    """
    query = remove_comments(query)  # Clean up query from comments
    where_clause = find_where_clause(query)
    
    if where_clause:
        table_column_pairs = extract_table_column_pairs(where_clause)
        return table_column_pairs
    return []


def remove_comments(query: str) -> str:
    """
    Removes SQL comments (single-line and block comments).
    """
    query = re.sub(r"--.*?$", "", query, flags=re.MULTILINE)  # Remove single-line comments
    query = re.sub(r"/\*.*?\*/", "", query, flags=re.DOTALL)  # Remove block comments
    return query


def find_where_clause(query: str) -> str:
    """
    Finds the WHERE clause in the SQL query.
    """
    where_clause_match = re.search(r"WHERE\s+(.*)", query, re.IGNORECASE)
    if where_clause_match:
        return where_clause_match.group(1)
    return ""


def extract_table_column_pairs(where_clause: str) -> List[WhereCondition]:
    """
    Extracts table.column pairs from the WHERE clause.
    """
    pattern = r"(\w+)\.(\w+)"
    matches = re.findall(pattern, where_clause)
    
    table_column_pairs = [WhereCondition(table, column) for table, column in matches]
    return table_column_pairs


def analyze_sql_file(file_path: str) -> SQLAnalysisResult:
    """
    Analyzes all SQL queries in a file, extracting WHERE conditions and returning them in structured format.
    """
    results = {}
    
    try:
        with open(file_path, 'r', encoding='utf-8') as f:
            queries = [query.strip() for query in f]
    except Exception as e:
        return SQLAnalysisResult(queries={"error": f"File reading error: {str(e)}"})
    
    # Process each query
    for i, query in enumerate(queries, 1):
        where_conditions = extract_where_conditions(query)
        query_analysis = QueryAnalysis(sql=query, where_conditions=where_conditions)
        results[f"Query {i}"] = query_analysis
    
    return SQLAnalysisResult(queries=results)


if __name__ == "__main__":
    # Example usage - replace with your SQL file path
    analysis_results = analyze_sql_file("./feature_sql.log")
    
    # Print analysis results
    if "error" in analysis_results.queries:
        print(f"Error: {analysis_results.queries['error']}")
    else:
        for query_name, analysis in analysis_results.queries.items():
            print(f"--- {query_name} ---")
            print(f"SQL: {analysis.sql}")
            print("WHERE Conditions:")
            for condition in analysis.where_conditions:
                print(f"  Table: {condition.table}, Column: {condition.column}")
