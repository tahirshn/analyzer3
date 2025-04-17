#!/bin/bash

input_file="log.txt"
output_file="queries.txt"

# Geçici değişken
query=""
in_query=false

# Satır satır log dosyasını oku
while IFS= read -r line; do
    # Eğer org.hibernate.SQL içeren satırsa ve ":" ile "[" içeriyorsa
    if [[ "$line" == *"org.hibernate.SQL"* && "$line" == *":"* ]]; then
        # ":" işaretinden sonrasını al
        query_part="${line#*:}"

        # Eğer "[" içeriyorsa bu satırda query bitmiş demektir
        if [[ "$query_part" == *"["* ]]; then
            query_only="${query_part%%\[*}"
            echo "$(echo "$query_only" | tr '\n' ' ' | xargs)" >> "$output_file"
        else
            # Query çok satırlıysa parçalamaya başla
            query="$query_part"
            in_query=true
        fi

    elif $in_query; then
        # Çok satırlı query'lerde satır sonu "[" ise query bitmiştir
        if [[ "$line" == *"["* ]]; then
            query="$query $line"
            query_only="${query%%\[*}"
            echo "$(echo "$query_only" | tr '\n' ' ' | xargs)" >> "$output_file"
            query=""
            in_query=false
        else
            # Sorgu devam ediyor
            query="$query $line"
        fi
    fi
done < "$input_file"
