#!/bin/bash

# Konfigürasyon
TEST_CONFIG_FILE="src/test/resources/application-test.properties"
RAW_LOG_FILE="build/sql_logs/raw_sql_output.log"
FINAL_SQL_FILE="build/sql_logs/cleaned_sql_queries.log"
GRADLE_COMMAND="./gradlew test --info"

# Dizinleri oluştur
mkdir -p "build/sql_logs"

# Önceki log dosyalarını temizle
> "$RAW_LOG_FILE"
> "$FINAL_SQL_FILE"

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
$GRADLE_COMMAND > "$RAW_LOG_FILE" 2>&1

echo "SQL sorguları temizleniyor ve parse ediliyor..."
python3 - <<END
import re
from collections import OrderedDict

input_file = "build/sql_logs/raw_sql_output.log"
output_file = "build/sql_logs/cleaned_sql_queries.log"

# SQL sorgularını tanımlayan regex pattern
sql_pattern = re.compile(r'(?:Hibernate:|org\.hibernate\.SQL.*-)\s*(.*)')

# Tekrarsız SQL sorgularını saklamak için
unique_sqls = OrderedDict()

with open(input_file, 'r') as f:
    for line in f:
        match = sql_pattern.search(line)
        if match:
            sql = match.group(1).strip()
            if sql and not sql.startswith(('select', 'insert', 'update', 'delete', 'call', 'alter', 'create', 'drop')): 
                continue
            if sql:
                # Parametreleri (?), :param gibi ifadelerle standartlaştır
                sql = re.sub(r'\?\d*', '?', sql)
                sql = re.sub(r':[a-zA-Z0-9_]+', '?', sql)
                sql = re.sub(r'\s+', ' ', sql).strip()
                unique_sqls[sql] = None

# Numaralandırılmış ve temizlenmiş SQL sorgularını yaz
with open(output_file, 'w') as f:
    for i, sql in enumerate(unique_sqls.keys(), 1):
        f.write(f"{i}. {sql}\n\n")

print(f"Toplam {len(unique_sqls)} benzersiz SQL sorgusu bulundu.")
END

echo -e "\nİşlem tamamlandı!"
echo "Ham loglar: $RAW_LOG_FILE"
echo "Temizlenmiş SQL sorguları: $FINAL_SQL_FILE"
