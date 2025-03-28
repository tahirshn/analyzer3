def _process_entity_file(self, content: str, file_path: str):
    """Process entity file with correct table name extraction"""
    class_match = re.search(r'class\s+(\w+)(?:\s*:\s*([^{\s]+))?', content)
    if not class_match:
        return
        
    class_name = class_match.group(1)
    
    # Get table name - first look for @Table(name = "...")
    table_match = re.search(
        r'@Table\s*\(\s*name\s*=\s*["\']([^"\']+)["\']',
        content,
        re.DOTALL
    )
    
    # If no explicit name, use class name converted to snake_case
    table_name = (table_match.group(1) if table_match else self._camel_to_snake(class_name)).lower()
    
    # Now process the entity (rest of the method remains the same)
    entity_info = EntityInfo(table_name)
    self.entity_map[class_name] = entity_info
    
    # ... rest of your entity processing code ...
