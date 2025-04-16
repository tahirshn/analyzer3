import psycopg2
from psycopg2 import sql
from dataclasses import dataclass
from typing import List, Dict

@dataclass
class IndexInfo:
    """Represents basic index information from a PostgreSQL database"""
    table_name: str
    index_name: str
    column_name: str
    is_primary: bool
    is_unique: bool

class PostgresIndexFetcher:
    """Fetches index information from PostgreSQL database"""
    
    def __init__(self, connection_params: Dict[str, str]):
        """Initialize with database connection parameters"""
        self.connection_params = connection_params
        self.connection = None

    def __enter__(self):
        """Context manager entry point"""
        self.connect()
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        """Context manager exit point"""
        self.disconnect()

    def connect(self):
        """Establish connection to PostgreSQL database"""
        try:
            self.connection = psycopg2.connect(**self.connection_params)
        except Exception as e:
            raise ConnectionError(f"PostgreSQL connection error: {e}")

    def disconnect(self):
        """Close database connection"""
        if self.connection and not self.connection.closed:
            self.connection.close()

    def get_user_indexes(self, schema_name: str = 'public') -> List[IndexInfo]:
        """
        Retrieve index information for user-created tables in specified schema
        
        Args:
            schema_name: Name of the schema to analyze (default: 'public')
            
        Returns:
            List of IndexInfo objects containing basic index information
        """
        if not self.connection or self.connection.closed:
            raise ConnectionError("No active database connection")

        query = """
        SELECT
            t.relname AS table_name,
            i.relname AS index_name,
            a.attname AS column_name,
            ix.indisprimary AS is_primary,
            ix.indisunique AS is_unique
        FROM
            pg_class t
            JOIN pg_index ix ON t.oid = ix.indrelid
            JOIN pg_class i ON i.oid = ix.indexrelid
            JOIN pg_namespace n ON n.oid = t.relnamespace
            JOIN pg_attribute a ON a.attrelid = t.oid AND a.attnum = ANY(ix.indkey)
        WHERE
            t.relkind = 'r'  -- Regular tables only
            AND n.nspname = %s  -- Specific schema
            AND t.relname NOT LIKE 'pg\\_%'  -- Exclude system tables
            AND t.relname NOT LIKE 'sql\\_%'  -- Exclude system tables
        ORDER BY
            t.relname,
            i.relname;
        """

        try:
            with self.connection.cursor() as cursor:
                cursor.execute(query, (schema_name,))
                return [
                    IndexInfo(
                        table_name=row[0],
                        index_name=row[1],
                        column_name=row[2],
                        is_primary=row[3],
                        is_unique=row[4]
                    )
                    for row in cursor.fetchall()
                ]

        except Exception as e:
            raise RuntimeError(f"Error fetching index information: {e}")

# Example usage
if __name__ == "__main__":
    # Configure your database connection
    db_config = {
        "host": "localhost",
        "database": "your_db",
        "user": "your_user",
        "password": "your_password",
        "port": "5432"
    }

    try:
        with PostgresIndexFetcher(db_config) as fetcher:
            indexes = fetcher.get_user_indexes()
            
            # Print basic index information
            print("Table Indexes:")
            for idx in indexes:
                print(f"{idx.table_name}.{idx.index_name} ({idx.column_name}) "
                      f"{'PK' if idx.is_primary else ''}{'UQ' if idx.is_unique else ''}")
                
    except Exception as e:
        print(f"Error: {e}")
