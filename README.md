awk '
/org.hibernate.SQL/ {
    # SQL sorgusunun olduğu kısmı çıkart
    if ($0 ~ /select|insert|update|delete/) {
        match($0, /: (.*) \[/, arr)  # Sorgu kısmını al
        print arr[1]                  # Yalnızca SQL sorgusunu yazdır
    }
}' "$input_file" > "$output_file"

echo "SQL queries extracted and saved to $output_file"
