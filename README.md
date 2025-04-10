#!/bin/bash

# Konfigürasyon
TEST_CONFIG_FILE="src/test/resources/application-test.properties"
RAW_SQL_LOG="build/sql_logs/raw_sql_output.log"
CLEAN_SQL_FILE="build/sql_logs/clean_sql_queries.log"
GRADLE_COMMAND="./gradlew test --info"

# Dizinleri oluştur
mkdir -p "build/sql_logs"

# Önceki log dosyalarını temizle
> "$RAW_SQL_LOG"
> "$CLEAN_SQL_FILE"

# Test için geçici log ayarlarını oluştur
echo "Spring ve Hibernate log ayarları etkinleştiriliyor..."
cat > "$TEST_CONFIG_FILE" << 'EOL'
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.properties.hibernate.use_sql_comments=true
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.type=TRACE
logging.pattern.console=%d{yyyy-MM-dd HH:mm:ss} %-5level %logger{36} - %msg%n
EOL

# Testleri çalıştır ve raw logları kaydet
echo "Testler çalıştırılıyor ve SQL sorguları toplanıyor..."
$GRADLE_COMMAND > "$RAW_SQL_LOG" 2>&1

# SQL sorgularını temizle ve formatla
echo "SQL sorguları temizleniyor ve numaralandırılıyor..."

# 1. Hibernate SQL loglarını çıkar
# 2. Duplikasyonları kaldır
# 3. Numaralandır
grep -E "Hibernate:|org.hibernate.SQL" "$RAW_SQL_LOG" | \
  sed -E 's/^.*Hibernate: //;s/^.*org.hibernate.SQL.* - //' | \
  sort | uniq | \
  awk '{printf "%d. %s\n", NR, $0}' > "$CLEAN_SQL_FILE"

# Geçici config dosyasını temizle (opsiyonel)
# rm "$TEST_CONFIG_FILE"

echo -e "\nİşlem tamamlandı!"
echo "Ham loglar: $RAW_SQL_LOG"
echo "Temizlenmiş SQL sorguları: $CLEAN_SQL_FILE"
echo "Toplam benzersiz sorgu sayısı: $(wc -l < "$CLEAN_SQL_FILE")"
