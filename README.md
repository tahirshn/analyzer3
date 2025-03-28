class SpringDataQueryAnalyzer(QueryAnalyzer):
    # ... (keep existing __init__ and other methods)
    
    def _analyze_repository_file(self, content: str, file_path: str) -> Set[QueryCondition]:
        """Comprehensively analyze repository file for query conditions"""
        conditions = set()
        
        # 1. Analyze derived query methods
        conditions.update(self._parse_derived_queries(content, file_path))
        
        # 2. Analyze @Query annotations (both JPQL and native SQL)
        conditions.update(self._parse_query_annotations(content, file_path))
        
        # 3. Analyze custom implementation methods if needed
        conditions.update(self._parse_custom_impl(content, file_path))
        
        return conditions

    def _parse_derived_queries(self, content: str, file_path: str) -> Set[QueryCondition]:
        """Parse Spring Data derived query methods"""
        conditions = set()
        table_name = self._guess_table_name(file_path)
        
        # Match method patterns like findByLastNameAndFirstName
        for method_match in re.finditer(
            r'fun\s+(?:find|read|get|query|count)(?:[A-Z]\w+)?By([A-Z]\w+)\([^)]*\)',
            content
        ):
            method_body = method_match.group(1)
            
            # Split conditions (And/Or)
            parts = re.split('(And|Or)', method_body)
            field_ops = []
            
            # First part is always a field
            field_ops.append((parts[0], '='))
            
            # Process remaining parts in pairs (connector, field)
            for i in range(1, len(parts), 2):
                if i + 1 >= len(parts):
                    break
                connector = parts[i]
                field_part = parts[i + 1]
                
                # Extract operator from field part
                field_name, operator = self._parse_field_with_operator(field_part)
                field_ops.append((field_name, operator))
            
            # Create conditions for each field
            for field_name, operator in field_ops:
                snake_case_field = self._camel_to_snake(field_name)
                conditions.add(QueryCondition(
                    table_name=table_name,
                    column_name=snake_case_field,
                    operator=operator
                ))
        
        return conditions

    def _parse_field_with_operator(self, field_part: str) -> Tuple[str, str]:
        """Extract field name and operator from method part"""
        # Check for operators in the field part
        for op_pattern, operator in [
            ('(LessThan|Before)', '<'),
            ('(LessThanEqual)', '<='),
            ('(GreaterThan|After)', '>'),
            ('(GreaterThanEqual)', '>='),
            ('(Not)', '<>'),
            ('(Like)', 'LIKE'),
            ('(In)', 'IN'),
            ('(Between)', 'BETWEEN'),
            ('(IsNull|Null)', 'IS NULL'),
            ('(IsNotNull|NotNull)', 'IS NOT NULL')
        ]:
            match = re.search(f'(.+?){op_pattern}', field_part)
            if match:
                return match.group(1), operator
        
        # Default to equality if no operator found
        return field_part, '='

    def _parse_query_annotations(self, content: str, file_path: str) -> Set[QueryCondition]:
        """Parse both JPQL and native SQL queries from @Query annotations"""
        conditions = set()
        table_name = self._guess_table_name(file_path)
        
        for query_match in re.finditer(
            r'@Query\s*\(\s*value\s*=\s*["\'](.*?)["\']',
            content,
            re.DOTALL
        ):
            query = query_match.group(1)
            
            # Clean up query string
            query = re.sub(r'[\n\r\t]', ' ', query).strip()
            
            if query.lower().startswith('select'):
                # JPQL/HQL query
                conditions.update(self._parse_jpql_query(query, table_name))
            else:
                # Native SQL query
                conditions.update(self._parse_sql_query(query, table_name))
        
        return conditions

    def _parse_jpql_query(self, query: str, table_name: str) -> Set[QueryCondition]:
        """Parse JPQL/HQL query conditions"""
        conditions = set()
        
        # Extract WHERE clause (simplified)
        where_match = re.search(
            r'where\s+(.*?)(?:\s+group\s+by|\s+order\s+by|\s*$)', 
            query, 
            re.IGNORECASE
        )
        if not where_match:
            return conditions
            
        where_clause = where_match.group(1)
        
        # Parse conditions (simplified pattern matching)
        for cond_match in re.finditer(
            r'(\w+)\.(\w+)\s*(=|!=|>|<|>=|<=|like|in|is\s+null|is\s+not\s+null)\b',
            where_clause,
            re.IGNORECASE
        ):
            entity_alias = cond_match.group(1)
            column = cond_match.group(2)
            operator = cond_match.group(3).upper()
            
            conditions.add(QueryCondition(
                table_name=table_name,
                column_name=column,
                operator=operator
            ))
        
        return conditions

    def _parse_sql_query(self, query: str, table_name: str) -> Set[QueryCondition]:
        """Parse native SQL query conditions"""
        conditions = set()
        
        try:
            parsed = sqlparse.parse(query)
            for stmt in parsed:
                for token in stmt.tokens:
                    if isinstance(token, sqlparse.sql.Where):
                        for identifier in token.get_identifiers():
                            # Find comparison operators near the identifier
                            next_token = token.token_next(token.token_index(identifier))[1]
                            operator = '='
                            if next_token and next_token.value.upper() in ('=', '!=', '>', '<', '>=', '<=', 'LIKE', 'IN'):
                                operator = next_token.value.upper()
                            
                            conditions.add(QueryCondition(
                                table_name=table_name,
                                column_name=identifier.get_real_name(),
                                operator=operator
                            ))
        except Exception as e:
            logger.warning(f"{LogIcons.WARNING.value} Failed to parse SQL query: {str(e)}")
        
        return conditions

    def _parse_custom_impl(self, content: str, file_path: str) -> Set[QueryCondition]:
        """Parse custom implementation methods if needed"""
        # This can be expanded to analyze custom query builder methods
        return set()

    def _camel_to_snake(self, name: str) -> str:
        """Convert CamelCase to snake_case"""
        name = re.sub('(.)([A-Z][a-z]+)', r'\1_\2', name)
        return re.sub('([a-z0-9])([A-Z])', r'\1_\2', name).lower()
