#!/bin/bash

INPUT_FILE="file1"
OUTPUT_FILE="file2"

awk '
BEGIN {
    collecting = 0
    buffer = ""
}
/^org.hibernate.SQL:/ {
    if (collecting && buffer != "") {
        print buffer
    }
    collecting = 1
    buffer = $0
    next
}
collecting && /^[[:space:]]+/ {
    # Remove leading spaces and append with space
    sub(/^[[:space:]]+/, "", $0)
    buffer = buffer " " $0
    next
}
collecting {
    print buffer
    buffer = ""
    collecting = 0
}
END {
    if (collecting && buffer != "") {
        print buffer
    }
}
' "$INPUT_FILE" > "$OUTPUT_FILE"
