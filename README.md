input_file="log.txt"
temp_file="log_normalized.txt"

in_sql_block=false
sql_block=""

> "$temp_file"  # temp dosyayı temizle

while IFS= read -r line; do
    if [[ "$line" == *"org.hibernate.SQL"* ]]; then
        # org.hibernate.SQL satırı başlıyorsa yeni SQL bloğu başlıyor
        in_sql_block=true
        sql_block="$line"
    elif $in_sql_block; then
        # SQL bloğu devam ediyor
        sql_block="$sql_block $line"

        # Satırın sonunda "[" varsa (log context başlıyorsa) query bitmiştir
        if [[ "$line" == *"["* ]]; then
            echo "$sql_block" >> "$temp_file"
            in_sql_block=false
            sql_block=""
        fi
    else
        # Normal satır, direkt yaz
        echo "$line" >> "$temp_file"
    fi
done < "$input_file"

# Geçici dosyayı orijinal dosya ile değiştir
mv "$temp_file" "$input_file
