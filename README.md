#!/bin/bash

# Check if input file is provided
if [ $# -eq 0 ]; then
    echo "Usage: $0 <input_file>"
    echo "Output will be printed to stdout"
    exit 1
fi

input_file="$1"

# Process the file to merge multi-line log entries
awk '
# Pattern for new log entry (timestamp at start of line)
/^[0-9]{4}-[0-9]{2}-[0-9]{2} [0-9]{2}:[0-9]{2}:[0-9]{2}/ {
    if (buffer != "") {
        # Print the previous entry
        print buffer
        buffer = ""
    }
    # Start new buffer with current line
    buffer = $0
    next
}

# For continuation lines (not starting with timestamp)
{
    # Trim leading whitespace and add to buffer
    sub(/^[[:space:]]+/, "")
    buffer = buffer " " $0
}

END {
    # Print the last entry
    if (buffer != "") {
        print buffer
    }
}
' "$input_file"
