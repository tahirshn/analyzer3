def _process_query(self, query: str, entity_type: str, file_path: str):
    """Process JPQL/SQL queries with proper alias resolution"""
    try:
        # Clean and normalize the query
        query = ' '.join(query.split()).replace('( ', '(').replace(' )', ')')
        
        # Get the main entity table name
        main_table = self.entity_map[entity_type].table
        
        # Parse FROM clause to get the main entity alias
        from_match = re.search(
            r'FROM\s+(\w+)\s+(\w+)',
            query,
            re.IGNORECASE
        )
        
        # Build table aliases mapping (alias -> table_name)
        table_aliases = {}
        if from_match:
            table_name = from_match.group(1).lower()
            alias = from_match.group(2)
            table_aliases[alias] = table_name
        
        # Parse JOIN clauses
        join_matches = re.finditer(
            r'(?:JOIN|LEFT JOIN|INNER JOIN|OUTER JOIN)\s+(\w+)\s+(\w+)'
            r'(?:\s+ON\s+((?:(?!\b(?:WHERE|AND|OR)\b).)*))?',
            query,
            re.IGNORECASE | re.DOTALL
        )
        
        for match in join_matches:
            entity_name = match.group(1).lower()
            alias = match.group(2)
            table_aliases[alias] = entity_name

        # If no explicit alias, use the entity name as default alias
        if not table_aliases and entity_type.lower() in query.lower():
            table_aliases[entity_type.lower()] = main_table

        # Parse WHERE conditions with alias support
        where_index = query.upper().find('WHERE ')
        if where_index > 0:
            where_clause = query[where_index + 6:]
            
            condition_pattern = r'''
                (?:^|\s+|\))          # Start of string or whitespace
                (?P<table_alias>\w+)  # Table alias
                \.                    # Dot separator
                (?P<column>\w+)       # Column name
                \s*                   # Optional whitespace
                (?P<operator>
                    =|!=|<>|>|<|>=|<=|   # Comparison operators
                    IS(?:\s+NOT)?\s+NULL|  # NULL checks
                    (?:NOT\s+)?(?:IN|LIKE|BETWEEN)| # Special operators
                    \s*=\s*:\w+       # Parameter binding
                )
            '''
            
            for condition in re.finditer(condition_pattern, where_clause, 
                                       re.IGNORECASE | re.VERBOSE):
                alias = condition.group('table_alias')
                column = condition.group('column').lower()
                operator = condition.group('operator').strip().upper()
                
                # Resolve the actual table name
                table_name = None
                if alias in table_aliases:
                    table_name = table_aliases[alias]
                elif alias == entity_type.lower():
                    table_name = main_table
                else:
                    # Try to find entity by alias (for simple queries)
                    table_name = next(
                        (v.table for k, v in self.entity_map.items() 
                         if k.lower() == alias),
                        None
                    )
                
                if not table_name:
                    print(f"⚠️ Could not resolve table alias '{alias}' in query. "
                          f"Available aliases: {table_aliases}")
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
                print(f"ℹ️ Found condition: {table_name}.{column} {operator} (alias: {alias})")
                
    except Exception as e:
        print(f"⚠️ Error processing query in {file_path}: {e}")
