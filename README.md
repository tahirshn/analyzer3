#!/bin/bash

INPUT_FILE="your-log-file.log"
OUTPUT_FILE="output.sql"

> "$OUTPUT_FILE"  # output dosyasını sıfırla

# Çok satırlı SQL'leri toplayıp tek satır haline getiren awk script
awk '
BEGIN { collecting = 0; sql_line = "" }

# SQL log başlangıcı
/^\S+ \S+ +DEBUG +[0-9]+ ---.*org\.hibernate\.SQL/ {
    collecting = 1;
    sql_line = "";
    next;
}

collecting {
    # Metadata gibi köşeli parantezle başlayan satır -> SQL bitti
    if ($0 ~ /^\s*\[.*\]$/) {
        # tüm whitespace karakterlerini tek boşluğa indir
        gsub(/[\r\n]/, " ", sql_line);
        gsub(/[ \t]+/, " ", sql_line);
        gsub(/^ +| +$/, "", sql_line);
        print sql_line >> "'"$OUTPUT_FILE"'";
        collecting = 0;
        next;
    }

    # SQL satırlarını topluyoruz
    sql_line = sql_line " " $0;
}
' "$INPUT_FILE"

echo "Extracted SQLs written to: $OUTPUT_FILE"
