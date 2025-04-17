import re

input_file = "your-log-file.log"
output_file = "output.sql"

with open(input_file, "r", encoding="utf-8") as f:
    lines = f.readlines()

sql_lines = []
collecting = False
current_sql = []

for line in lines:
    if "org.hibernate.SQL" in line:
        collecting = True
        current_sql = []
        continue

    # Bitiş koşulu: metadata satırı (köşeli parantezle başlayan satır)
    if collecting:
        if re.match(r'^\s*\[.*\]$', line):
            sql = " ".join(l.strip() for l in current_sql)
            sql = re.sub(r'\s+', ' ', sql).strip()
            sql_lines.append(sql)
            collecting = False
        else:
            current_sql.append(line)

# Dosyaya yaz
with open(output_file, "w", encoding="utf-8") as f:
    for sql in sql_lines:
        f.write(sql + "\n")

print(f"✅ Extracted {len(sql_lines)} SQL statements to {output_file}")
