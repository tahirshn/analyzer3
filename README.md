# diff_sql_generator.py

def load_queries(file_path):
    with open(file_path, "r") as f:
        return set(line.strip() for line in f if line.strip())

def write_diff(diff_queries, output_path):
    with open(output_path, "w") as f:
        for query in sorted(diff_queries):
            f.write(query + "\n")

if __name__ == "__main__":
    feature_path = "feature_branch.log"
    master_path = "master_branch.log"
    output_path = "diff_sql.log"

    feature_queries = load_queries(feature_path)
    master_queries = load_queries(master_path)

    diff = feature_queries - master_queries
    write_diff(diff, output_path)

    print(f"Farklı sorgular {output_path} dosyasına yazıldı.")
