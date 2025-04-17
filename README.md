#!/bin/bash

# Çıktı dosyasını tanımla
OUTPUT_FILE="extracted_sql_$(date +'%Y%m%d_%H%M%S').log"

echo "Spring Boot test loglarından SQL sorguları çıkarılıyor..."
echo "Çıktı dosyası: $OUTPUT_FILE"
echo "Çıkmak için CTRL+C tuşlarına basın..."

# Temiz çıkış fonksiyonu
cleanup() {
  echo -e "\nSQL sorguları $OUTPUT_FILE dosyasına kaydedildi."
  exit 0
}
trap cleanup SIGINT

# Logları işleme fonksiyonu
process_logs() {
  awk '
  BEGIN { buffer = ""; sql_found = 0 }
  
  # SQL log satırını tespit et
  /^[0-9]{4}-[0-9]{2}-[0-9]{2} [0-9]{2}:[0-9]{2}:[0-9]{2}\.[0-9]{3}.*org\.hibernate\.SQL/ {
    if (buffer != "") {
      print buffer >> "'"$OUTPUT_FILE"'"
    }
    buffer = $0
    sql_found = 1
    next
  }
  
  # SQL devam satırları (girintili)
  sql_found && /^[[:space:]]+/ {
    sub(/^[[:space:]]+/, "")
    buffer = buffer " " $0
    next
  }
  
  # SQL olmayan satırlar
  {
    if (sql_found && buffer != "") {
      print buffer >> "'"$OUTPUT_FILE"'"
      buffer = ""
      sql_found = 0
    }
  }
  
  END {
    if (sql_found && buffer != "") {
      print buffer >> "'"$OUTPUT_FILE"'"
    }
  }
  '
}

# Ana işlem
if [ "$1" == "-f" ] && [ -f "$2" ]; then
  # Dosyadan oku
  cat "$2" | process_logs
else
  # Canlı logları dinle
  mvn test | process_logs
fi

cleanup
