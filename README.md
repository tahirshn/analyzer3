class KotlinEntityExtractor(MetadataExtractor):
    """Extracts metadata from Kotlin entity classes"""
    
    def extract_entity_metadata(self, file_content: str, file_path: str) -> Optional[EntityMetadata]:
        if '@Entity' not in file_content:
            return None

        class_name = self._extract_class_name(file_content)
        if not class_name:
            return None

        table_name = self._extract_table_name(file_content, class_name)
        fields = self._extract_fields(file_content, class_name)
        relationships = self._extract_relationships(file_content)

        return EntityMetadata(
            table_name=table_name,
            fields=fields,
            relationships=relationships
        )

    def _extract_class_name(self, content: str) -> Optional[str]:
        match = re.search(r'class\s+(\w+)', content)
        return match.group(1) if match else None

    def _extract_table_name(self, content: str, class_name: str) -> str:
        table_match = re.search(
            r'@Table\s*\(\s*name\s*=\s*["\']([^"\']+)["\']',
            content,
            re.DOTALL
        )
        if table_match:
            return table_match.group(1).lower()
        return self._camel_to_snake(class_name)

    def _extract_column_name(self, field_content: str, field_name: str) -> str:
        """Extracts the column name from a field declaration"""
        # Check for explicit @Column annotation
        column_match = re.search(
            r'@Column\s*\(\s*name\s*=\s*["\']([^"\']+)["\']',
            field_content
        )
        if column_match:
            return column_match.group(1).lower()
        
        # Check for @JoinColumn annotation (for relationships)
        join_column_match = re.search(
            r'@JoinColumn\s*\(\s*name\s*=\s*["\']([^"\']+)["\']',
            field_content
        )
        if join_column_match:
            return join_column_match.group(1).lower()
        
        # Default to snake_case version of field name
        return self._camel_to_snake(field_name)

    def _extract_fields(self, content: str, class_name: str) -> Dict[str, EntityField]:
        fields = {}
        
        for prop_match in re.finditer(
            r'val\s+(\w+)\s*:\s*([^\n=]+?)(?:\s*=\s*[^\n]+)?(?:\s*@[^\n]+)*',
            content,
            re.DOTALL
        ):
            prop_name = prop_match.group(1)
            full_match = prop_match.group(0)
            
            if '@OneToMany' in full_match:
                continue
                
            is_id = '@Id' in full_match
            column_name = self._extract_column_name(full_match, prop_name)
            
            fields[prop_name] = EntityField(
                name=prop_name,
                column_name=column_name,
                is_id=is_id,
                is_relationship='@ManyToOne' in full_match
            )
        
        return fields

    def _extract_relationships(self, content: str) -> Dict[str, RelationshipMetadata]:
        relationships = {}
        
        for rel_match in re.finditer(
            r'@ManyToOne[^v]*val\s+(\w+)\s*:\s*(\w+)[^@]*@JoinColumn\s*\([^)]*name\s*=\s*["\']([^"\']+)["\']',
            content,
            re.DOTALL
        ):
            prop_name = rel_match.group(1)
            target_entity = rel_match.group(2)
            join_column = rel_match.group(3).lower()
            
            relationships[prop_name] = RelationshipMetadata(
                target_entity=target_entity,
                join_column=join_column,
                relationship_type='ManyToOne'
            )
        
        return relationships

    def _camel_to_snake(self, name: str) -> str:
        name = re.sub('(.)([A-Z][a-z]+)', r'\1_\2', name)
        return re.sub('([a-z0-9])([A-Z])', r'\1_\2', name).lower()
