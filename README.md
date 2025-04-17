# Geçici değişken
query=""

# SQL sorgularını ayıklayıp yalnızca sorguları dosyaya yaz
while IFS= read -r line; do
    # Satırda org.hibernate.SQL içeren bir şey varsa
    if [[ "$line" =~ "org.hibernate.SQL" ]]; then
        # : ile [ arasında olan kısmı çek
        if [[ "$line" =~ :\ (.*)\ \[ ]]; then
            query="${BASH_REMATCH[1]}"
        fi
    elif [[ -n "$query" && "$line" =~ "^\s" ]]; then
        # Eğer sorgu var ve satır boşlukla başlıyorsa, sorguya ekle
        query="$query $line"
    elif [[ -n "$query" && -z "$line" ]]; then
        # Boş satır geldiğinde sorguyu kaydet
        echo "$query" >> "$output_file"
        query=""
    fi
done < "$input_file"

# Eğer son satırda bir sorgu varsa, onu kaydet
if [[ -n "$query" ]]; then
    echo "$query" >> "$output_file"
fi

echo "SQL queries extracted and saved to $output_file"
