import psycopg2
from psycopg2 import sql
from tabulate import tabulate

def get_index_info(connection_params, schema_name):
    """
    PostgreSQL'de belirtilen şemadaki tüm tabloların indeks bilgilerini getirir
    
    Args:
        connection_params (dict): PostgreSQL bağlantı parametreleri
        schema_name (str): İncelecek şema adı
    
    Returns:
        list: İndeks bilgilerini içeren sözlük listesi
    """
    query = """
    SELECT
        t.relname AS table_name,
        i.relname AS index_name,
        a.attname AS column_name,
        CASE WHEN indisprimary THEN TRUE ELSE FALSE END AS is_primary,
        CASE WHEN indisunique THEN TRUE ELSE FALSE END AS is_unique
    FROM
        pg_class t,
        pg_class i,
        pg_index ix,
        pg_attribute a,
        pg_namespace n
    WHERE
        t.oid = ix.indrelid
        AND i.oid = ix.indexrelid
        AND n.oid = t.relnamespace
        AND a.attrelid = t.oid
        AND a.attnum = ANY(ix.indkey)
        AND t.relkind = 'r'  -- Sadece tablolar
        AND n.nspname = %s  -- Şema adı
    ORDER BY
        t.relname,
        i.relname;
    """
    
    try:
        # PostgreSQL bağlantısı oluştur
        conn = psycopg2.connect(**connection_params)
        cursor = conn.cursor()
        
        # Sorguyu çalıştır
        cursor.execute(query, (schema_name,))
        
        # Sonuçları al
        columns = [desc[0] for desc in cursor.description]
        results = cursor.fetchall()
        
        # Sonuçları sözlük listesine çevir
        index_info = []
        for row in results:
            index_info.append(dict(zip(columns, row)))
        
        return index_info
        
    except Exception as e:
        print(f"Hata oluştu: {e}")
        return []
    finally:
        if 'conn' in locals():
            conn.close()

# Kullanım örneği
if __name__ == "__main__":
    # PostgreSQL bağlantı bilgileri
    connection_params = {
        "host": "localhost",
        "database": "your_database",
        "user": "your_username",
        "password": "your_password",
        "port": "5432"
    }
    
    # İncelemek istediğiniz şema adı
    schema_name = "public"  # Veya başka bir şema adı
    
    # İndeks bilgilerini al
    indexes = get_index_info(connection_params, schema_name)
    
    # Sonuçları tablo halinde göster
    if indexes:
        print(tabulate(indexes, headers="keys", tablefmt="grid"))
    else:
        print("Belirtilen şemada indeks bulunamadı.")
