import re

input_file = "your-log-file.log"
output_file = "output.sql"

with open(input_file, "r", encoding="utf-8") as f:
    lines = f.readlines()

sql_lines = set()
collecting = False
current_sql = []

for line in lines:
    # org.hibernate.SQL ile başlayan satır, bir SQL bloğunun başlangıcı olabilir
    if "org.hibernate.SQL" in line:
        collecting = True
        current_sql = []
        continue

    if collecting:
        # Köşeli parantez ile başlayan log metadata'sı ise, SQL tamamlandı
        if re.match(r'^\s*\[.*\]\s*$', line.strip()):
            if current_sql:
                sql = " ".join(l.strip() for l in current_sql)
                sql = re.sub(r'\s+', ' ', sql).strip()
                if sql:  # Boş değilse ve tekrar değilse ekle
                    sql_lines.add(sql)
            collecting = False
        else:
            current_sql.append(line)

# Dosyaya yaz
with open(output_file, "w", encoding="utf-8") as f:
    for sql in sorted(sql_lines):  # İstersen sıralamayı kaldırabilirim
        f.write(sql + "\n")

print(f"✅ Found {len(sql_lines)} unique SQL queries.")
print(f"📄 Output written to: {output_file}")
