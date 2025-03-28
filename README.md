def _process_query(self, query: str, entity_type: str, file_path: str):
    """Process a JPQL/SQL query with fixed regex pattern"""
    try:
        query = query.replace('\n', ' ').strip()
        if not query:
            return
            
        table_name = self.entity_map[entity_type].table
        
        # Extract conditions from WHERE clause
        where_index = query.upper().find('WHERE ')
        if where_index > 0:
            where_clause = query[where_index + 6:]
            
            # Fixed regex pattern with balanced parentheses
            pattern = r'''
                (\w+)\.?                # table reference or alias
                (\w+)                   # column name
                \s*                     # optional whitespace
                (
                    [=<>]+              # basic comparison operators
                    |                   # OR
                    IS\s+(?:NOT\s+)?NULL  # IS NULL or IS NOT NULL
                    |                   # OR
                    (?:NOT\s+)?(?:IN|LIKE|BETWEEN)  # NOT IN, LIKE, BETWEEN etc.
                )
            '''
            
            for condition in re.finditer(pattern, where_clause, re.IGNORECASE | re.VERBOSE):
                table_ref = condition.group(1)
                column = condition.group(2).lower()
                operator = condition.group(3).upper()
                
                # Skip ID columns
                if column in self.entity_map[entity_type].id_columns:
                    continue
                    
                if table_ref.lower() in ('entity', ''):
                    self.query_conditions.add(QueryCondition(
                        table_name, 
                        column, 
                        operator
                    ))
                else:
                    # Handle joins - would need more sophisticated parsing
                    pass
    except Exception as e:
        print(f"⚠️ Error processing query in {file_path}: {e}")
