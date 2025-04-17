awk '
BEGIN {
    collecting = 0
    buffer = ""
}
/^[[:space:]]*org.hibernate.SQL:/ {
    if (collecting && buffer != "") {
        print buffer
    }
    collecting = 1
    buffer = $0
    sub(/^[[:space:]]*/, "", buffer)
    next
}
collecting && /^[[:space:]]+/ {
    sub(/^[[:space:]]+/, "", $0)
    buffer = buffer " " $0
    next
}
collecting {
    print buffer
    buffer = ""
    collecting = 0
}
END {
    if (collecting && buffer != "") {
        print buffer
    }
}
'
