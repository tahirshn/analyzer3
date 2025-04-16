#!/bin/bash

# Exit immediately if a command exits with a non-zero status
set -e
set -o pipefail

# Branch and project settings
FEATURE_BRANCH=$(git rev-parse --abbrev-ref HEAD)
MASTER_BRANCH="master"
ORIGINAL_BRANCH="$FEATURE_BRANCH"
PROJECT_DIR="$(git rev-parse --show-toplevel)"

# Paths to scripts
SQL_BEAUTIFIER="$PROJECT_DIR/script/sql_index_checker/extract_sql_beautifier.py"
SQL_DIFF="$PROJECT_DIR/script/sql_index_checker/diff_sql_generator.py"

# Paths to properties files
TEST_CONFIG_FILE="$PROJECT_DIR/card-service-app/src/test/resources/application-test.properties"
CONFIG_FILE="$PROJECT_DIR/card-service-app/src/test/resources/application.properties"

# Log output files
FEATURE_SQL="$PROJECT_DIR/feature_sql.log"
MASTER_SQL="$PROJECT_DIR/master_sql.log"
SQL_DIFF_FILE="$PROJECT_DIR/diff_sql.sql"

GRADLE_COMMAND="./gradlew test --info"

# Ensure script is run on a feature branch
if [[ "$FEATURE_BRANCH" == "$MASTER_BRANCH" ]]; then
  echo "⛔ This script should be executed from a feature branch."
  exit 1
fi

# Function to collect and format SQL logs from a branch
collect_sql_log() {
  local branch=$1
  local out_log=$2

  echo ">>> Switching to branch: $branch"
  git checkout "$branch"

  echo ">>> Enabling Spring + Hibernate SQL logging..."
  cp "$CONFIG_FILE" "$TEST_CONFIG_FILE"
  cat << 'EOL' >> "$CONFIG_FILE"
# Hibernate SQL Logging
logging.level.ROOT=ERROR
logging.level.org.hibernate.SQL=DEBUG
logging.pattern.console=%d{yyyy-MM-dd HH:mm:ss} %-5level %logger{36} - %msg%n
EOL

  echo ">>> Running tests and capturing SQL logs..."
  $GRADLE_COMMAND | tee "$out_log"

  echo ">>> Filtering Hibernate SQL lines..."
  grep -E "Hibernate:|org.hibernate.SQL" "$out_log" > "${out_log}.tmp"
  mv "${out_log}.tmp" "$out_log"

  echo ">>> Formatting SQL log with Python formatter..."
  python3 "$SQL_BEAUTIFIER" "$out_log" "${out_log}.tmp"
  mv "${out_log}.tmp" "$out_log"

  echo "✔ Done: SQL log written to $out_log"

  echo ">>> Restoring original application.properties..."
  cp "$TEST_CONFIG_FILE" "$CONFIG_FILE"
  rm -f "$TEST_CONFIG_FILE"
}

# Collect logs for both branches
collect_sql_log "$ORIGINAL_BRANCH" "$FEATURE_SQL"
collect_sql_log "$MASTER_BRANCH" "$MASTER_SQL"

# Run Python diff analyzer to compare SQL logs
echo ">>> Running SQL diff analyzer..."
python3 "$SQL_DIFF" "$FEATURE_SQL" "$MASTER_SQL" "$SQL_DIFF_FILE"
echo "✔ SQL diff written to $SQL_DIFF_FILE"

# Switch back to original branch
git checkout "$ORIGINAL_BRANCH"
echo "✔ Switched back to $ORIGINAL_BRANCH"
