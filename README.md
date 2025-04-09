#!/bin/bash

# Konfigürasyon
TEST_CONFIG_FILE="src/test/resources/application-test.properties"
SQL_LOG_FILE="build/sql_logs/sql_queries.log"
GRADLE_COMMAND="./gradlew test --info"

# Log dizinini oluştur
mkdir -p "build/sql_logs"

# Önceki log dosyasını temizle
> "$SQL_LOG_FILE"

# Test için geçici log ayarlarını oluştur
echo "Spring ve Hibernate log ayarları etkinleştiriliyor..."
cat > "$TEST_CONFIG_FILE" << 'EOL'
# Hibernate SQL Logging
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.properties.hibernate.use_sql_comments=true

# Logging Levels
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.type=TRACE
logging.level.org.hibernate.stat=DEBUG
logging.level.org.hibernate.engine.QueryPlan=DEBUG

# Log output to console
logging.pattern.console=%d{yyyy-MM-dd HH:mm:ss} %-5level %logger{36} - %msg%n
EOL

echo "Testler çalıştırılıyor ve SQL sorguları loglanıyor..."
$GRADLE_COMMAND | tee -a "$SQL_LOG_FILE"

# Logları filtrele ve formatla
echo "SQL sorguları filtreleniyor..."
grep -E "Hibernate:|org.hibernate.SQL" "$SQL_LOG_FILE" > "${SQL_LOG_FILE}.tmp"
sed -i -e 's/^.*Hibernate: //' \
       -e 's/^.*org.hibernate.SQL.* - //' \
       -e '/^[[:space:]]*$/d' \
       "${SQL_LOG_FILE}.tmp"

mv "${SQL_LOG_FILE}.tmp" "$SQL_LOG_FILE"

# Geçici config dosyasını temizle (opsiyonel)
# rm "$TEST_CONFIG_FILE"

echo -e "\nİşlem tamamlandı!"
echo "SQL sorguları şu dosyada kaydedildi: $SQL_LOG_FILE"
