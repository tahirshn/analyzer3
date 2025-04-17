import re

# Helper function to extract table.column pairs
def extract_table_column_pairs(query):
    # Regular expression to match patterns like table.column, even with aliases
    pattern = r"(\w+(\.\w+)?)\.(\w+)"
    
    matches = re.findall(pattern, query)

    # Create a set to avoid duplicates and return the table.column pairs
    table_column_pairs = set()
    for match in matches:
        table_column_pairs.add(match[0])
    
    return table_column_pairs

# Main function to extract WHERE clause conditions
def extract_where_conditions(query):
    # Clean up the query by removing comments and unnecessary spaces
    query = re.sub(r"--.*?$", "", query, flags=re.MULTILINE)  # Remove single line comments
    query = re.sub(r"/\*.*?\*/", "", query, flags=re.DOTALL)  # Remove block comments
    
    # Find the WHERE clause (case insensitive)
    where_clause_match = re.search(r"WHERE\s+(.*)", query, re.IGNORECASE)
    
    if where_clause_match:
        where_clause = where_clause_match.group(1)
        
        # Find all table.column pairs in the WHERE clause
        table_column_pairs = extract_table_column_pairs(where_clause)
        
        return table_column_pairs
    else:
        return set()

# Example usage
sql_query = """
SELECT 
    a.name, b.address 
FROM 
    users a
JOIN 
    orders b ON a.id = b.user_id
WHERE 
    a.name = 'John Doe' AND b.amount > 1000 AND a.status = 'active'
"""

# Extracting the table.column pairs from the WHERE clause
table_column_pairs = extract_where_conditions(sql_query)
print("Extracted table.column pairs:", table_column_pairs)
