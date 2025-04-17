#!/bin/bash

# Check if input file is provided
if [ $# -eq 0 ]; then
    echo "Usage: $0 <input_file>"
    echo "Output will be printed to stdout"
    exit 1
fi

input_file="$1"

# Process the file to filter and merge SQL query logs
awk '
# Pattern for new log entry with SQL (timestamp + "org.hibernate.SQL")
/^[0-9]{4}-[0-9]{2}-[0-9]{2} [0-9]{2}:[0-9]{2}:[0-9]{2}.*org\.hibernate\.SQL/ {
    if (buffer != "") {
        # Print the previous SQL entry
        print buffer
    }
    # Start new buffer with current line
    buffer = $0
    next
}

# For continuation lines when we have an active SQL buffer
buffer != "" && /^[[:space:]]+/ {
    # Trim leading whitespace and add to buffer
    sub(/^[[:space:]]+/, "")
    buffer = buffer " " $0
    next
}

# For any other line (non-SQL logs)
{
    # If we encounter a non-SQL log while we have a buffer,
    # print the buffer and reset it
    if (buffer != "") {
        print buffer
        buffer = ""
    }
}

END {
    # Print the last SQL entry if exists
    if (buffer != "") {
        print buffer
    }
}
' "$input_file"
