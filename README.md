#!/bin/bash

# SQL log dosyasının adı ve yeri
SQL_LOG_FILE="sql_queries.log"
OUTPUT_DIR="build/sql_logs"

# Log dosyası için dizin oluştur
mkdir -p "$OUTPUT_DIR"
FULL_PATH="$OUTPUT_DIR/$SQL_LOG_FILE"

# Önceki log dosyasını temizle
> "$FULL_PATH"

echo "SQL sorguları $FULL_PATH dosyasına kaydedilecek..."

# Hibernate SQL log ayarlarını geçici olarak etkinleştirerek testleri çalıştır
./gradlew test \
  -Dspring.jpa.show-sql=true \
  -Dspring.jpa.properties.hibernate.format_sql=true \
  -Dspring.jpa.properties.hibernate.use_sql_comments=true \
  -Dlogging.level.org.hibernate.SQL=DEBUG \
  -Dlogging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE \
  2>&1 | tee -a "$FULL_PATH"

# Sadece SQL sorgularını filtrele ve log dosyasına yaz
grep -h -E "Hibernate:|org.hibernate.SQL" "$FULL_PATH" > "${FULL_PATH}.tmp" && mv "${FULL_PATH}.tmp" "$FULL_PATH"

# Format the SQL file for better readability
sed -i 's/^.*Hibernate: //g' "$FULL_PATH"
sed -i 's/^.*org.hibernate.SQL.* - //g' "$FULL_PATH"

echo "Testler tamamlandı. SQL sorguları $FULL_PATH dosyasına kaydedildi."
