def _process_query(self, query: str, entity_type: str, file_path: str):
    """Process complex JPQL/SQL queries with JOINs"""
    try:
        # Clean and normalize the query
        query = ' '.join(query.split()).replace('( ', '(').replace(' )', ')')
        
        # Get the main entity table name
        main_table = self.entity_map[entity_type].table
        
        # Parse JOIN clauses to build table aliases mapping
        join_matches = re.finditer(
            r'(?:JOIN|LEFT JOIN|INNER JOIN|OUTER JOIN)\s+(\w+)\s+(\w+)'
            r'(?:\s+ON\s+((?:(?!\b(?:WHERE|AND|OR)\b).)*))?',
            query,
            re.IGNORECASE | re.DOTALL
        )
        
        table_aliases = {}
        for match in join_matches:
            entity_name = match.group(1)
            alias = match.group(2)
            join_condition = match.group(3)
            table_aliases[alias] = entity_name.lower()  # Map alias to table name

        # Parse WHERE conditions with JOIN support
        where_index = query.upper().find('WHERE ')
        if where_index > 0:
            where_clause = query[where_index + 6:]
            
            # Improved pattern to handle table aliases and complex conditions
            condition_pattern = r'''
                (?:^|\s+|\))          # Start of string or whitespace or closing paren
                (?P<table_alias>\w+)  # Table alias
                \.                    # Dot separator
                (?P<column>\w+)       # Column name
                \s*                   # Optional whitespace
                (?P<operator>
                    =|!=|<>|>|<|>=|<=|   # Comparison operators
                    IS(?:\s+NOT)?\s+NULL|  # NULL checks
                    (?:NOT\s+)?(?:IN|LIKE|BETWEEN)  # Special operators
                )
                \s*
                (?P<value>
                    :\w+|            # Named parameters
                    '[^']*'|         # String literals
                    \d+\.?\d*|       # Numbers
                    \(.*?\)          # Sub-expressions
                )?
            '''
            
            for condition in re.finditer(condition_pattern, where_clause, 
                                       re.IGNORECASE | re.VERBOSE):
                alias = condition.group('table_alias')
                column = condition.group('column').lower()
                operator = condition.group('operator').upper()
                
                # Resolve the actual table name from the alias
                table_name = table_aliases.get(alias, main_table if alias == entity_type.lower() else None)
                
                if not table_name:
                    print(f"⚠️ Could not resolve table alias '{alias}' in query")
                    continue
                
                # Skip ID columns
                entity_for_table = next(
                    (k for k, v in self.entity_map.items() if v.table == table_name),
                    None
                )
                if entity_for_table and column in self.entity_map[entity_for_table].id_columns:
                    continue
                
                self.query_conditions.add(QueryCondition(
                    table_name,
                    column,
                    operator
                ))
                print(f"ℹ️ Found condition: {table_name}.{column} {operator} (from JOIN)")
                
    except Exception as e:
        print(f"⚠️ Error processing query in {file_path}: {e}")
