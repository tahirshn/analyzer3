import psycopg2
from psycopg2 import sql
from tabulate import tabulate

def get_indexes_from_user_tables(connection_params, schema_name='public'):
    """
    Kullanıcı tarafından oluşturulan tablolardaki indeksleri getirir
    
    Args:
        connection_params (dict): PostgreSQL bağlantı parametreleri
        schema_name (str): İncelecek şema adı (varsayılan: 'public')
    
    Returns:
        list: İndeks bilgilerini içeren sözlük listesi
    """
    query = """
    WITH user_tables AS (
        SELECT c.oid, c.relname
        FROM pg_class c
        JOIN pg_namespace n ON n.oid = c.relnamespace
        WHERE c.relkind = 'r'  -- Sadece tablolar
        AND n.nspname = %s    -- Belirtilen şema
        AND c.relname NOT LIKE 'pg_%'  -- Sistem tablolarını hariç tut
        AND c.relname NOT LIKE 'sql_%' -- Sistem tablolarını hariç tut
        AND c.relname NOT IN ('spatial_ref_sys', 'geometry_columns', 'geography_columns')  -- PostGIS tablolarını hariç tut
    )
    SELECT
        ut.relname AS table_name,
        i.relname AS index_name,
        array_to_string(array_agg(a.attname), ', ') AS column_names,
        CASE WHEN ix.indisprimary THEN TRUE ELSE FALSE END AS is_primary,
        CASE WHEN ix.indisunique THEN TRUE ELSE FALSE END AS is_unique,
        am.amname AS index_type,
        pg_get_indexdef(i.oid) AS index_definition
    FROM
        pg_index ix
        JOIN user_tables ut ON ut.oid = ix.indrelid
        JOIN pg_class i ON i.oid = ix.indexrelid
        JOIN pg_attribute a ON a.attrelid = ut.oid AND a.attnum = ANY(ix.indkey)
        JOIN pg_am am ON i.relam = am.oid
    GROUP BY
        ut.relname,
        i.relname,
        ix.indisprimary,
        ix.indisunique,
        am.amname,
        i.oid
    ORDER BY
        ut.relname,
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
    
    # İndeks bilgilerini al
    indexes = get_indexes_from_user_tables(connection_params)
    
    # Sonuçları tablo halinde göster
    if indexes:
        print(tabulate(indexes, headers="keys", tablefmt="grid"))
    else:
        print("Kullanıcı tablolarında indeks bulunamadı.")
