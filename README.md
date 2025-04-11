import psycopg2
import os

# Ortam değişkenlerinden alabiliriz, ya da doğrudan buraya yazabilirsin
DB_CONFIG = {
    "host": os.getenv("DB_HOST", "localhost"),
    "port": os.getenv("DB_PORT", "5432"),
    "dbname": os.getenv("DB_NAME", "test_db"),
    "user": os.getenv("DB_USER", "test_user"),
    "password": os.getenv("DB_PASSWORD", "test_pass")
}

def fetch_indexes():
    try:
        conn = psycopg2.connect(**DB_CONFIG)
        cursor = conn.cursor()

        cursor.execute("""
            SELECT 
                t.relname AS table_name,
                i.relname AS index_name,
                a.attname AS column_name,
                ix.indisunique AS is_unique,
                ix.indisprimary AS is_primary
            FROM 
                pg_class t,
                pg_class i,
                pg_index ix,
                pg_attribute a
            WHERE 
                t.oid = ix.indrelid
                AND i.oid = ix.indexrelid
                AND a.attrelid = t.oid
                AND a.attnum = ANY(ix.indkey)
                AND t.relkind = 'r'
            ORDER BY
                t.relname, i.relname;
        """)

        indexes = cursor.fetchall()
        if not indexes:
            print("No indexes found.")
        else:
            print("Index list:")
            for row in indexes:
                print(f"Table: {row[0]}, Index: {row[1]}, Column: {row[2]}, Unique: {row[3]}, Primary: {row[4]}")
        
        cursor.close()
        conn.close()
    except Exception as e:
        print(f"Error while fetching indexes: {e}")

if __name__ == "__main__":
    fetch_indexes()
