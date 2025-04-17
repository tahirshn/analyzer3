awk '/org.hibernate.SQL/ { 
    # Sadece sorgu kısmını al
    if ($0 ~ /select|insert|update|delete/) {
        gsub(/^[^:]+: */, "");  # Satırın başındaki log kısmını temizle
        gsub(/\[.*\]$/, "");     # Sonundaki parametre kısmını temizle
        print $0                # Sadece SQL sorgusunu yazdır
    }
