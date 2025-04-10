import re

# Define the file paths
input_file = 'input_file.txt'
output_file = 'output_file.txt'

def process_log_line(line):
    # Use regex to extract the SQL query part from the log line
    # Also remove any trailing "[]" characters
    line = line.strip()
    line = re.sub(r"\s*\[\s*\]\s*$", "", line)  # Remove empty "[]" at the end of the line
    # Match SQL statements (e.g., select, insert, update, etc.)
    match = re.search(r"(select|insert|update|delete|merge|alter|create|drop|truncate|rename|grant|revoke).*", line, re.IGNORECASE)
    if match:
        return match.group(0).strip()  # Return the matched SQL query
    return None

def process_file(input_file, output_file):
    # Create a set to store unique SQL queries
    unique_queries = set()
    
    # Open input and output files
    with open(input_file, 'r') as infile, open(output_file, 'w') as outfile:
        for line in infile:
            # Process each line
            sql_query = process_log_line(line)
            if sql_query:
                # If the query is not already in the set, add it and write to output file
                if sql_query not in unique_queries:
                    unique_queries.add(sql_query)
                    outfile.write(sql_query + '\n')

# Process the input file and write unique queries to the output file
process_file(input_file, output_file)

print(f"SQL queries have been successfully written to {output_file}.")
