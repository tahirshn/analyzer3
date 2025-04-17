grep 'org.hibernate.SQL' "$input_file" \
  | sed -E 's/^[^:]+: *//' \
  | sed -E 's/ *\[[^]]*\]$//' \
  | sed '/^$/d'
