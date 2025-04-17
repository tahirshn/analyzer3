#!/bin/bash

# Start timing
START_TIME=$(date +%s)

# Color codes
RED='\033[0;31m'
GREEN='\033[0;32m'
BLUE='\033[1;34m'
NC='\033[0m' # No Color

# Exit immediately if any command fails
set -e
set -o pipefail

# Determine branches and paths
FEATURE_BRANCH=$(git rev-parse --abbrev-ref HEAD)
MASTER_BRANCH="master"
ORIGINAL_BRANCH="$FEATURE_BRANCH"
PROJECT_DIR="$(git rev-parse --show-toplevel)"

# Script and config paths
SQL_BEAUTIFIER="$PROJECT_DIR/script/sql_index_checker/extract_sql_beautifier.py"
SQL_DIFF="$PROJECT_DIR/script/sql_index_checker/diff_sql_generator.py"
TEST_CONFIG_FILE="$PROJECT_DIR/card-service-app/src/test/resources/application-test.properties"
CONFIG_FILE="$PROJECT_DIR/card-service-app/src/test/resources/application.properties"

# Output files
FEATURE_SQL="$PROJECT_DIR/feature_sql.log"
MASTER_SQL="$PROJECT_DIR/master_sql.log"
SQL_DIFF_FILE="$PROJECT_DIR/diff_sql.log"

GRADLE_COMMAND="./gradlew test --info"

# Ensure we're not on master branch
if [[ "$FEATURE_BRANCH" == "$MASTER_BRANCH" ]]; then
  echo -e "${RED}⛔ This script should be executed from a feature branch.${NC}"
  exit 1
fi

# Function to collect SQL logs
collect_sql_log() {
  local branch=$1
  local out_log=$2

  echo -e "${BLUE}ℹ Checking out branch: $branch${NC}"
  git checkout "$branch"

  echo -e "${BLUE}ℹ Enabling SQL logging...${NC}"
  cp "$CONFIG_FILE" "$TEST_CONFIG_FILE"
  cat << 'EOL' >> "$CONFIG_FILE"
# Hibernate SQL Logging (temporary for log extraction)
logging.level.ROOT=ERROR
logging.level.org.hibernate.SQL=DEBUG
logging.pattern.console=%d{yyyy-MM-dd HH:mm:ss} %-5level %logger{36} - %msg%n
EOL

  echo -e "${BLUE}ℹ Running tests to collect SQL logs...${NC}"
  $GRADLE_COMMAND | tee "$out_log"

  echo -e "${BLUE}ℹ Formatting SQL with Python formatter...${NC}"
  python3 "$SQL_BEAUTIFIER" "$out_log" "${out_log}.tmp"
  mv "${out_log}.tmp" "$out_log"

  echo -e "${GREEN}✔ SQL log written: $out_log${NC}"

  echo -e "${BLUE}ℹ Restoring application.properties...${NC}"
  cp "$TEST_CONFIG_FILE" "$CONFIG_FILE"
  rm -f "$TEST_CONFIG_FILE"
}

# Main execution
collect_sql_log "$ORIGINAL_BRANCH" "$FEATURE_SQL"
collect_sql_log "$MASTER_BRANCH" "$MASTER_SQL"

echo -e "${BLUE}ℹ Generating SQL diff...${NC}"
python3 "$SQL_DIFF" "$FEATURE_SQL" "$MASTER_SQL" "$SQL_DIFF_FILE"
echo -e "${GREEN}✔ SQL diff created at: $SQL_DIFF_FILE${NC}"

git checkout "$ORIGINAL_BRANCH"
echo -e "${GREEN}✔ Switched back to: $ORIGINAL_BRANCH${NC}"

# Show total runtime
END_TIME=$(date +%s)
DURATION=$((END_TIME - START_TIME))
echo -e "${GREEN}✔ Script completed in $DURATION seconds.${NC}"



-------------------------dockerfile-----------------------------
FROM python:3.10-slim

# Install Java and Gradle
RUN apt-get update && \
    apt-get install -y openjdk-17-jdk curl unzip git && \
    curl -s "https://get.sdkman.io" | bash && \
    bash -c "source $HOME/.sdkman/bin/sdkman-init.sh && sdk install gradle 8.5"

# Set workdir
WORKDIR /app

# Copy project files
COPY . /app

# Set environment
ENV JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
ENV PATH=$PATH:/root/.sdkman/candidates/gradle/current/bin

# Default command
CMD ["/bin/bash"]

--------------------------------
.PHONY: run check clean

# Run script inside Docker
run:
	docker build -t sql-checker .
	docker run -it --rm -v $$PWD:/app sql-checker bash script/sql_index_checker/check_sql_diff.sh

# Run check script without Docker
check:
	bash script/sql_index_checker/check_sql_diff.sh

# Cleanup logs
clean:
	rm -f feature_sql.log master_sql.log diff_sql.log

-----------------------------------
Docker ile çalıştırmak için:
make run

Normal çalıştırmak için (sistemde Python, Java, Gradle yüklü olmalı):
make check

Log temizliği için:
make clean

