#!/usr/bin/env python3
"""
Enterprise-Grade Database Index Analyzer for Spring Data JPA Applications
Architected with SOLID principles and Clean Architecture
"""

from abc import ABC, abstractmethod
from dataclasses import dataclass
from pathlib import Path
import re
import sqlparse
from typing import Dict, List, Optional, Set, Tuple
import argparse

# ====================== Domain Models ======================
@dataclass(frozen=True)
class EntityField:
    name: str
    column_name: str
    is_id: bool = False
    is_relationship: bool = False

@dataclass
class EntityMetadata:
    table_name: str
    fields: Dict[str, EntityField]
    relationships: Dict[str, 'RelationshipMetadata']

@dataclass
class RelationshipMetadata:
    target_entity: str
    join_column: str
    relationship_type: str  # 'ManyToOne', 'OneToMany', etc.

@dataclass(frozen=True)
class QueryCondition:
    table_name: str
    column_name: str
    operator: str

# ====================== Core Abstraction ======================
class MetadataExtractor(ABC):
    @abstractmethod
    def extract_entity_metadata(self, file_content: str, file_path: str) -> Optional[EntityMetadata]:
        pass

class QueryAnalyzer(ABC):
    @abstractmethod
    def analyze_repositories(self, repo_path: str) -> Set[QueryCondition]:
        pass

# ====================== Concrete Implementations ======================
class KotlinEntityExtractor(MetadataExtractor):
    """Extracts metadata from Kotlin entity classes following SOLID principles"""
    
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
        # First try to get explicit table name
        table_match = re.search(
            r'@Table\s*\(\s*name\s*=\s*["\']([^"\']+)["\']',
            content,
            re.DOTALL
        )
        if table_match:
            return table_match.group(1).lower()
        
        # Fallback to snake_case class name
        return self._camel_to_snake(class_name)

    def _extract_fields(self, content: str, class_name: str) -> Dict[str, EntityField]:
        fields = {}
        
        for prop_match in re.finditer(
            r'val\s+(\w+)\s*:\s*([^\n=]+?)(?:\s*=\s*[^\n]+)?(?:\s*@[^\n]+)*',
            content,
            re.DOTALL
        ):
            prop_name = prop_match.group(1)
            full_match = prop_match.group(0)
            
            # Skip OneToMany relationships
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

class SpringDataQueryAnalyzer(QueryAnalyzer):
    """Analyzes Spring Data repositories with comprehensive query support"""
    
    def __init__(self, metadata_extractor: MetadataExtractor):
        self.metadata_extractor = metadata_extractor
        self.jpa_keywords = {
            'And': 'AND', 'Or': 'OR',
            'Is': '=', 'Equals': '=',
            'Between': 'BETWEEN',
            'LessThan': '<', 'LessThanEqual': '<=',
            'GreaterThan': '>', 'GreaterThanEqual': '>=',
            'After': '>', 'Before': '<',
            'IsNull': 'IS NULL', 'Null': 'IS NULL',
            'IsNotNull': 'IS NOT NULL', 'NotNull': 'IS NOT NULL',
            'Like': 'LIKE', 'NotLike': 'NOT LIKE',
            'StartingWith': 'LIKE', 'EndingWith': 'LIKE', 'Containing': 'LIKE',
            'Not': '<>', 'In': 'IN', 'NotIn': 'NOT IN',
            'True': '= TRUE', 'False': '= FALSE'
        }

    def analyze_repositories(self, repo_path: str) -> Set[QueryCondition]:
        conditions = set()
        
        for repo_file in Path(repo_path).rglob('*Repository.kt'):
            with open(repo_file, 'r', encoding='utf-8') as f:
                content = f.read()
                conditions.update(self._analyze_repository_file(content, repo_file))
        
        return conditions

    def _analyze_repository_file(self, content: str, file_path: str) -> Set[QueryCondition]:
        conditions = set()
        
        # Analyze query methods
        for method_match in re.finditer(
            r'fun\s+(find|read|query|get)(Distinct)?By([A-Z]\w+)\(',
            content
        ):
            method_body = method_match.group(3)
            conditions.update(self._parse_query_method(method_body, file_path))
        
        # Analyze @Query annotations
        for query_match in re.finditer(
            r'@Query\("(.*?)"\)',
            content,
            re.DOTALL
        ):
            conditions.update(self._parse_jpql_query(query_match.group(1), file_path))
        
        return conditions

    def _parse_query_method(self, method_body: str, file_path: str) -> Set[QueryCondition]:
        # Implementation of method name parsing
        pass

    def _parse_jpql_query(self, query: str, file_path: str) -> Set[QueryCondition]:
        # Implementation of JPQL parsing
        pass

# ====================== Main Application ======================
class IndexAnalyzer:
    """Orchestrates the index analysis process following Clean Architecture"""
    
    def __init__(self, metadata_extractor: MetadataExtractor, query_analyzer: QueryAnalyzer):
        self.metadata_extractor = metadata_extractor
        self.query_analyzer = query_analyzer
        self.entity_metadata: Dict[str, EntityMetadata] = {}
        self.existing_indexes: Dict[str, Set[Tuple[str, ...]]] = {}

    def run_analysis(self, repo_path: str, flyway_path: str) -> None:
        self._load_entity_metadata(repo_path)
        self._load_existing_indexes(flyway_path)
        self._analyze_and_report()

    def _load_entity_metadata(self, repo_path: str) -> None:
        for kotlin_file in Path(repo_path).rglob('*.kt'):
            with open(kotlin_file, 'r', encoding='utf-8') as f:
                content = f.read()
                if metadata := self.metadata_extractor.extract_entity_metadata(content, str(kotlin_file)):
                    self.entity_metadata[metadata.table_name] = metadata

    def _load_existing_indexes(self, flyway_path: str) -> None:
        for migration in Path(flyway_path).glob('*.sql'):
            with open(migration, 'r', encoding='utf-8') as f:
                content = f.read()
                self._parse_migration(content)

    def _parse_migration(self, content: str) -> None:
        # Parse SQL migrations for existing indexes
        pass

    def _analyze_and_report(self) -> None:
        query_conditions = self.query_analyzer.analyze_repositories(str(Path.cwd()))
        missing_indexes = self._identify_missing_indexes(query_conditions)
        self._generate_report(missing_indexes)

    def _identify_missing_indexes(self, conditions: Set[QueryCondition]) -> Set[QueryCondition]:
        missing = set()
        
        for condition in conditions:
            if condition.table_name not in self.entity_metadata:
                continue
                
            entity = self.entity_metadata[condition.table_name]
            field = next((f for f in entity.fields.values() 
                         if f.column_name == condition.column_name), None)
                
            if not field or field.is_id:
                continue
                
            if not self._has_appropriate_index(condition):
                missing.add(condition)
        
        return missing

    def _has_appropriate_index(self, condition: QueryCondition) -> bool:
        if condition.table_name not in self.existing_indexes:
            return False
            
        for index in self.existing_indexes[condition.table_name]:
            if condition.column_name in index:
                if condition.operator in {'<', '<=', '>', '>=', 'BETWEEN'}:
                    return index[0] == condition.column_name
                return True
        return False

    def _generate_report(self, missing_indexes: Set[QueryCondition]) -> None:
        if not missing_indexes:
            print("✅ No missing indexes found!")
            return
            
        print(f"❌ Found {len(missing_indexes)} missing indexes:")
        for condition in sorted(missing_indexes, key=lambda c: (c.table_name, c.column_name)):
            self._print_index_recommendation(condition)

    def _print_index_recommendation(self, condition: QueryCondition) -> None:
        # Detailed recommendation printing
        pass

def main():
    parser = argparse.ArgumentParser(
        description="Enterprise Database Index Analyzer for Spring Data JPA"
    )
    parser.add_argument("--repo-path", default=".",
                       help="Path to code repository")
    parser.add_argument("--flyway-path", default="src/main/resources/db/migration",
                       help="Path to Flyway migrations")
    
    args = parser.parse_args()

    # Dependency injection
    metadata_extractor = KotlinEntityExtractor()
    query_analyzer = SpringDataQueryAnalyzer(metadata_extractor)
    analyzer = IndexAnalyzer(metadata_extractor, query_analyzer)
    
    analyzer.run_analysis(args.repo_path, args.flyway_path)

if __name__ == "__main__":
    main()

# Add to existing analyzer.py

class PipelineIndexAnalyzer(IndexAnalyzer):
    """Extended analyzer with pipeline-specific functionality"""
    
    def __init__(self, metadata_extractor: MetadataExtractor, query_analyzer: QueryAnalyzer):
        super().__init__(metadata_extractor, query_analyzer)
        self.fail_threshold = 0  # Default: fail on any missing index
    
    def set_fail_threshold(self, threshold: int):
        """Set how many missing indexes are allowed before failing"""
        self.fail_threshold = threshold
    
    def run_pipeline_analysis(self, repo_path: str, flyway_path: str) -> bool:
        """Run analysis and return whether pipeline should pass"""
        self.run_analysis(repo_path, flyway_path)
        missing_count = len(self._identify_missing_indexes(
            self.query_analyzer.analyze_repositories(repo_path)
        ))
        
        if missing_count > self.fail_threshold:
            print(f"🚨 Pipeline failed - {missing_count} missing indexes detected")
            return False
        return True

# Add to existing analyzer.py

class HTMLReporter:
    """Generate HTML reports for pipeline artifacts"""
    
    @staticmethod
    def generate_report(missing_indexes: Set[QueryCondition], output_path: str = "index-report.html"):
        with open(output_path, 'w') as f:
            f.write("<html><head><title>Missing Index Report</title></head><body>")
            f.write("<h1>Missing Database Indexes Report</h1>")
            
            if not missing_indexes:
                f.write("<p>✅ No missing indexes found!</p>")
                return
            
            # Group by table
            by_table = {}
            for condition in missing_indexes:
                by_table.setdefault(condition.table_name, []).append(condition)
            
            for table, conditions in sorted(by_table.items()):
                f.write(f"<h2>Table: {table}</h2><ul>")
                for cond in sorted(conditions, key=lambda c: c.column_name):
                    f.write(f"<li>Column: {cond.column_name} (used with operator: {cond.operator})</li>")
                f.write("</ul>")
            
            f.write("</body></html>")
