def _process_entity_file(self, content: str, file_path: str):
    """Process a Kotlin entity file with improved @Table annotation parsing"""
    class_match = re.search(r'class\s+(\w+)(?:\s*:\s*([^{\s]+))?', content)
    if not class_match:
        return
        
    class_name = class_match.group(1)
    parent_class = class_match.group(2)
    
    # Improved @Table annotation parsing that handles different formatting styles
    table_match = re.search(
        r'@Table\s*\([^)]*name\s*=\s*["\']([^"\']+)["\'][^)]*\)',
        content,
        re.DOTALL
    )
    
    # Fallback pattern for multi-line @Table annotations
    if not table_match:
        table_match = re.search(
            r'@Table\s*\(\s*name\s*=\s*["\']([^"\']+)["\']\s*\)',
            content,
            re.DOTALL
        )
    
    table_name = (table_match.group(1) if table_match else class_name.lower()
    
    # Clean up table name (remove whitespace, newlines, etc.)
    table_name = re.sub(r'\s+', '', table_name).lower()
    
    entity_info = EntityInfo(table_name)
    self.entity_map[class_name] = entity_info

    # Rest of the method remains the same...
    # [Previous property parsing code here...]


    # Parse all properties with more robust annotation handling
    for prop_match in re.finditer(
        r'(?:@\w+(?:\([^)]*\))?\s*)*val\s+(\w+)\s*:\s*([^\n=]+?)(?:\s*=\s*[^\n]+)?(?:\s*@[^\n]+)*',
        content,
        re.DOTALL
    ):
        prop_name = prop_match.group(1)
        prop_type = prop_match.group(2).strip().split('?')[0]  # Remove nullable
        full_match = prop_match.group(0)
        
        # Handle @Id fields with more flexible parsing
        if '@Id' in full_match:
            col_match = re.search(
                r'@Column\s*\(\s*name\s*=\s*["\']([^"\']+)["\']\s*\)',
                full_match,
                re.DOTALL
            )
            column_name = (col_match.group(1) if col_match else self._camel_to_snake(prop_name))
            # Clean up column name
            column_name = re.sub(r'\s+', '', column_name).lower()
            entity_info.id_columns.add(column_name)
            continue
        
        # Get column name with more robust parsing
        col_match = re.search(
            r'@Column\s*\(\s*name\s*=\s*["\']([^"\']+)["\']\s*\)',
            full_match,
            re.DOTALL
        )
        column_name = (col_match.group(1) if col_match else self._camel_to_snake(prop_name))
        column_name = re.sub(r'\s+', '', column_name).lower()
        
        # [Rest of relationship handling...]
