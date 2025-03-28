#!/usr/bin/env python3
"""
Complete Missing Index Detector with Full Spring Data JPA Keyword Support
"""

import os
import re
import sys
import argparse
from pathlib import Path
from typing import Set, Dict, List, Tuple, Optional, MutableSet, MutableMapping
import sqlparse

# Define all Spring Data JPA keywords and their operators
JPA_KEYWORDS = {
    'Distinct': None,  # Handled specially
    'And': 'AND',
    'Or': 'OR',
    'Is': '=',
    'Equals': '=',
    'Between': 'BETWEEN',
    'LessThan': '<',
    'LessThanEqual': '<=',
    'GreaterThan': '>',
    'GreaterThanEqual': '>=',
    'After': '>',
    'Before': '<',
    'IsNull': 'IS NULL',
    'Null': 'IS NULL',
    'IsNotNull': 'IS NOT NULL',
    'NotNull': 'IS NOT NULL',
    'Like': 'LIKE',
    'NotLike': 'NOT LIKE',
    'StartingWith': 'LIKE',  # Parameter will have % appended
    'EndingWith': 'LIKE',   # Parameter will have % prepended
    'Containing': 'LIKE',   # Parameter will be wrapped in %
    'Not': '<>',
    'In': 'IN',
    'NotIn': 'NOT IN',
    'True': '= TRUE',
    'False': '= FALSE',
    # OrderBy is handled separately as it doesn't affect WHERE clause
}

class EntityInfo:
    def __init__(self, table: str):
        self.table = table
        self.columns: Set[str] = set()
        self.relationships: Dict[str, 'RelationshipInfo'] = {}
        self.id_columns: Set[str] = set()

class RelationshipInfo:
    def __init__(self, rel_type: str, target: str, column: str):
        self.type = rel_type
        self.target = target
        self.column = column

class QueryCondition:
    def __init__(self, table: str, column: str, operator: str):
        self.table = table
        self.column = column
        self.operator = operator
    
    def __hash__(self):
        return hash((self.table, self.column, self.operator))
    
    def __eq__(self, other):
        return (self.table, self.column, self.operator) == (other.table, other.column, other.operator)

class MissingIndexDetector:
    def __init__(self, repo_path: str, flyway_path: str):
        self.repo_path = repo_path
        self.flyway_path = flyway_path
        self.entity_map: Dict[str, EntityInfo] = {}
        self.relationship_map: Dict[Tuple[str, str], str] = {}
        self.query_conditions: Set[QueryCondition] = set()
        self.indexes: Dict[str, Set[Tuple[str, ...]]] = {}

    def run_analysis(self):
        """Run complete analysis pipeline"""
        self._analyze_entities()
        self._analyze_repositories()
        self._parse_flyway_migrations()
        self._validate_indexes()
        self._generate_report()

    def _analyze_entities(self):
        """Scan Kotlin files for entity classes and relationships"""
        print("🔍 Analyzing Kotlin entity classes...")
        
        kotlin_files = []
        for root, _, files in os.walk(os.path.join(self.repo_path, "src/main/kotlin")):
            kotlin_files.extend(
                os.path.join(root, f) 
                for f in files 
                if f.endswith('.kt') and not f.endswith('Test.kt')
            )

        for file_path in kotlin_files:
            with open(file_path, 'r', encoding='utf-8') as f:
                content = f.read()
                if '@Entity' in content:
                    self._process_entity_file(content, file_path)

        print(f"📊 Mapped {len(self.entity_map)} entities with {len(self.relationship_map)} relationships")

    def _process_entity_file(self, content: str, file_path: str):
        """Process a Kotlin entity file with all annotations"""
        class_match = re.search(r'class\s+(\w+)(?:\s*:\s*([^{\s]+))?', content)
        if not class_match:
            return
            
        class_name = class_match.group(1)
        parent_class = class_match.group(2)
        
        # Get table name
        table_match = re.search(r'@Table\s*\(.*name\s*=\s*["\']([^"\']+)["\']', content)
        table_name = (table_match.group(1) if table_match else class_name.lower()
        
        entity_info = EntityInfo(table_name.lower())
        self.entity_map[class_name] = entity_info

        # Parse all properties with annotations
        for prop_match in re.finditer(
            r'(?:@\w+(?:\([^)]*\))?\s*)*val\s+(\w+)\s*:\s*([^\n=]+?)(?:\s*=\s*[^\n]+)?(?:\s*@[^\n]+)*',
            content,
            re.DOTALL
        ):
            prop_name = prop_match.group(1)
            prop_type = prop_match.group(2).strip().split('?')[0]  # Remove nullable
            full_match = prop_match.group(0)
            
            # Handle all @Id patterns
            if '@Id' in full_match:
                col_match = re.search(r'@Column\s*\(.*name\s*=\s*["\']([^"\']+)["\']', full_match)
                column_name = (col_match.group(1) if col_match else self._camel_to_snake(prop_name)).lower()
                entity_info.id_columns.add(column_name)
                print(f"ℹ️ Found @Id field: {column_name} in {class_name}")
                continue
            
            # Get column name from @Column or convert to snake_case
            col_match = re.search(r'@Column\s*\(.*name\s*=\s*["\']([^"\']+)["\']', full_match)
            column_name = (col_match.group(1) if col_match else self._camel_to_snake(prop_name)).lower()
            
            # Handle relationships
            if '@ManyToOne' in full_match:
                join_col_match = re.search(r'@JoinColumn\s*\(.*name\s*=\s*["\']([^"\']+)["\']', full_match)
                if join_col_match:
                    column_name = join_col_match.group(1).lower()
                
                entity_info.relationships[prop_name] = RelationshipInfo(
                    rel_type='ManyToOne',
                    target=prop_type,
                    column=column_name
                )
                self.relationship_map[(table_name.lower(), column_name)] = prop_type
            else:
                # Regular column
                entity_info.columns.add(column_name)

    def _camel_to_snake(self, name: str) -> str:
        """Convert camelCase to snake_case with proper acronym handling"""
        name = re.sub('(.)([A-Z][a-z]+)', r'\1_\2', name)
        return re.sub('([a-z0-9])([A-Z])', r'\1_\2', name).lower()

    def _analyze_repositories(self):
        """Scan repository interfaces for query methods"""
        print("🔍 Analyzing Spring Data repositories...")
        
        for root, _, files in os.walk(os.path.join(self.repo_path, "src/main/kotlin")):
            for file in files:
                if file.endswith('Repository.kt'):
                    self._process_repository_file(os.path.join(root, file))

    def _process_repository_file(self, file_path: str):
        """Process a single repository interface"""
        with open(file_path, 'r', encoding='utf-8') as f:
            content = f.read()
            
            # Find repository entity type
            entity_match = re.search(
                r'interface\s+\w+Repository\s*:\s*[^\s<]+\s*<\s*(\w+)\s*,',
                content
            )
            if not entity_match:
                return
                
            entity_type = entity_match.group(1)
            if entity_type not in self.entity_map:
                print(f"⚠️ Entity type {entity_type} not found in entity map")
                return
                
            # Process all query methods
            for method_match in re.finditer(
                r'fun\s+(find|read|query|get)(Distinct)?By([A-Z]\w+)\(',
                content
            ):
                method_prefix = method_match.group(1)
                is_distinct = method_match.group(2) is not None
                method_body = method_match.group(3)
                
                # Reconstruct full method name
                method_name = f"{method_prefix}{'Distinct' if is_distinct else ''}By{method_body}"
                self._process_repository_method(method_name, entity_type)

            # Process @Query annotations
            for query_match in re.finditer(
                r'@Query\("(.*?)"\)\s*\n\s*fun\s+\w+\(', 
                content, 
                re.DOTALL
            ):
                self._process_query(query_match.group(1), entity_type, file_path)

    def _process_repository_method(self, method_name: str, entity_type: str):
        """Parse complex JPA query method names into conditions"""
        if not any(method_name.startswith(prefix) for prefix in ['findBy', 'readBy', 'queryBy', 'getBy']):
            return
        
        # Extract the condition part (after By)
        condition_part = method_name.split('By', 1)[1]
        table_name = self.entity_map[entity_type].table
        
        # Handle OrderBy separately (doesn't affect WHERE clause)
        if 'OrderBy' in condition_part:
            condition_part = condition_part.split('OrderBy')[0]
        
        # Split into individual conditions
        conditions = re.split('(And|Or)(?=[A-Z])', condition_part)
        conditions = [c for c in conditions if c and c not in ('And', 'Or')]
        
        for condition in conditions:
            field = condition
            operator = '='  # Default operator
            
            # Check all JPA keywords
            for keyword, op in JPA_KEYWORDS.items():
                if keyword == 'Distinct':
                    continue  # Already handled
                
                if condition.endswith(keyword):
                    field = condition[:-len(keyword)]
                    operator = op
                    
                    # Handle special LIKE cases
                    if keyword in ['StartingWith', 'EndingWith', 'Containing']:
                        operator = 'LIKE'
                    break
            
            self._resolve_field_with_operator(table_name, field, operator, entity_type)

    def _resolve_field_with_operator(self, table_name: str, field: str, operator: str, entity_type: str):
        """Resolve field name and store with operator"""
        column_name = self._resolve_field_name(field, entity_type)
        if column_name:
            # Skip if this is an ID column (already indexed)
            if column_name in self.entity_map[entity_type].id_columns:
                print(f"ℹ️ Skipping {table_name}.{column_name} - already indexed (@Id)")
                return
                
            self.query_conditions.add(QueryCondition(table_name, column_name, operator))
            print(f"ℹ️ Found condition: {table_name}.{column_name} {operator}")

    def _resolve_field_name(self, field: str, entity_type: str) -> Optional[str]:
        """Resolve a field name to its database column name"""
        entity = self.entity_map[entity_type]
        field = field[0].lower() + field[1:] if field else field  # PascalCase to camelCase
        
        # 1. Check exact match
        if field in entity.columns:
            return field
        
        # 2. Check relationships
        if field in entity.relationships:
            return entity.relationships[field].column
        
        # 3. Convert naming conventions
        snake_case = self._camel_to_snake(field)
        if snake_case in entity.columns:
            return snake_case
        
        # 4. Handle compound fields
        if len(field) > 1 and any(c.isupper() for c in field[1:]):
            parts = re.findall('[A-Z][a-z]*', field)
            if parts:
                snake_case = '_'.join([p.lower() for p in parts])
                if snake_case in entity.columns:
                    return snake_case
        
        # 5. Try common variations
        variations = [
            field + '_id',
            snake_case + '_id',
            field.lower(),
            field.replace('Id', 'id')
        ]
        
        for variation in variations:
            if variation in entity.columns:
                return variation
        
        print(f"⚠️ Could not resolve field '{field}' in entity {entity_type}")
        return None

    def _process_query(self, query: str, entity_type: str, file_path: str):
        """Process a JPQL/SQL query"""
        try:
            query = query.replace('\n', ' ').strip()
            if not query:
                return
                
            table_name = self.entity_map[entity_type].table
            
            # Extract conditions from WHERE clause
            where_index = query.upper().find('WHERE ')
            if where_index > 0:
                where_clause = query[where_index + 6:]
                for condition in re.finditer(r'(\w+)\.?(\w+)\s*([=<>]+|(?:IS\s+(?:NOT\s+)?NULL|(?:NOT\s+)?(?:IN|LIKE|BETWEEN))', where_clause, re.IGNORECASE):
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
            print(f"⚠️ Error processing query in $file_path: {e}")

    def _parse_flyway_migrations(self):
        """Parse Flyway migrations for index definitions"""
        print("📦 Parsing Flyway migrations...")
        
        for migration in Path(self.flyway_path).glob('*.sql'):
            with open(migration, 'r', encoding='utf-8') as f:
                content = f.read()
                parsed = sqlparse.parse(content)
                
                for statement in parsed:
                    if 'CREATE INDEX' not in statement.value.upper():
                        continue
                    
                    tokens = statement.tokens
                    table_name = None
                    columns = []
                    
                    for i, token in enumerate(tokens):
                        if token.value.upper() == 'ON' and i+1 < len(tokens):
                            table_name = tokens[i+1].value.lower()
                        elif isinstance(token, sqlparse.sql.Parenthesis):
                            columns = [
                                t.value.lower().strip('"') 
                                for t in token.tokens 
                                if t.ttype is sqlparse.tokens.Name
                            ]
                    
                    if table_name and columns:
                        if table_name not in self.indexes:
                            self.indexes[table_name] = set()
                        self.indexes[table_name].add(tuple(columns))

    def _validate_indexes(self) -> List[QueryCondition]:
        """Validate that all query conditions have indexes"""
        missing = []
        
        for condition in self.query_conditions:
            # Skip if this is an ID column (already indexed)
            entity_type = next(
                (k for k, v in self.entity_map.items() 
                 if v.table == condition.table), 
                None
            )
            if entity_type and condition.column in self.entity_map[entity_type].id_columns:
                continue
                
            if condition.table not in self.indexes:
                missing.append(condition)
                continue
                
            has_index = False
            for index in self.indexes[condition.table]:
                if condition.column in index:
                    if condition.operator in ('<', '<=', '>', '>=', 'BETWEEN', 'AFTER', 'BEFORE'):
                        if index[0] == condition.column:  # Range queries need column first
                            has_index = True
                            break
                    else:
                        has_index = True
                        break
            
            if not has_index:
                missing.append(condition)
        
        return missing

    def _generate_report(self):
        """Generate a report of missing indexes"""
        missing = self._validate_indexes()
        
        if not missing:
            print("✅ No missing indexes found!")
            return
        
        print(f"❌ Found {len(missing)} missing indexes:")
        for condition in missing:
            relationship = self.relationship_map.get((condition.table, condition.column))
            rel_note = f" (relationship to {relationship})" if relationship else ""
            print(f"  - Table '{condition.table}' column '{condition.column}'{rel_note} used with '{condition.operator}'")
            
            if condition.operator in ('<', '<=', '>', '>=', 'BETWEEN', 'AFTER', 'BEFORE'):
                print(f"    Suggested index: CREATE INDEX idx_{condition.table}_{condition.column} ON {condition.table}({condition.column} DESC);")
            elif condition.operator in ('LIKE', 'NOT LIKE'):
                print(f"    Suggested index: CREATE INDEX idx_{condition.table}_{condition.column} ON {condition.table}({condition.column});")
            elif condition.operator == 'IN':
                print(f"    Suggested index: CREATE INDEX idx_{condition.table}_{condition.column} ON {condition.table}({condition.column});")
            elif relationship:
                print(f"    Suggested index: CREATE INDEX idx_{condition.table}_{condition.column} ON {condition.table}({condition.column});")
            else:
                print(f"    Suggested index: CREATE INDEX idx_{condition.table}_{condition.column} ON {condition.table}({condition.column});")

        print("\n💡 Optimization tips:")
        print("   - Always index foreign key columns (relationships)")
        print("   - Range queries benefit most from DESC indexes")
        print("   - LIKE queries with leading wildcards may not use indexes efficiently")
        print("   - @Id fields are automatically indexed and were ignored")

def main():
    parser = argparse.ArgumentParser(
        description="Missing Index Detector for Kotlin/Spring/Hibernate Projects"
    )
    parser.add_argument("--repo-path", default=".", 
        help="Path to code repository (default: current directory)")
    parser.add_argument("--flyway-path", default="src/main/resources/db/migration",
        help="Path to Flyway migrations (default: src/main/resources/db/migration)")
    
    args = parser.parse_args()

    detector = MissingIndexDetector(args.repo_path, args.flyway_path)
    detector.run_analysis()

if __name__ == "__main__":
    main()
