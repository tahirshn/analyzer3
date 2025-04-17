#!/bin/bash

INPUT_FILE="your-log-file.log"
OUTPUT_FILE="output.sql"

# output dosyasını sıfırla
> "$OUTPUT_FILE"

# çok satırlı sorguları birleştirip tek satıra al
awk '
BEGIN { collecting = 0; sql_line = "" }
/org\.hibernate\.SQL/ {
    collecting = 1;
    sql_line = "";  # yeni SQL için sıfırla
    next;
}
collecting {
    # Metadata kısmına (köşeli parantezle başlayan satır) geldiğimizde bitir
    if ($0 ~ /^\s*\[.*\]$/) {
        gsub(/\r/, "", sql_line);   # Windows tarzı satır sonlarını temizle
        gsub(/\n/, " ", sql_line);  # yeni satırları boşlukla değiştir
        gsub(/\s+/, " ", sql_line); # fazla boşlukları temizle
        print sql_line >> "'"$OUTPUT_FILE"'";
        collecting = 0;
        next;
    }
    # sorgu satırını ekle
    sql_line = sql_line " " $0;
}
' "$INPUT_FILE"

echo "SQL sorguları '$OUTPUT_FILE' dosyasına yazıldı."
