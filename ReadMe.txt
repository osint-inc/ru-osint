## OSINT of recon.sh (cleaned up)

Raw: https://github.com/derlemue/osint

### Script:

#!/bin/bash

# Automatisiertes Recon-Skript mit Tor-Absicherung
# Liest verwundbare Hosts aus attack_hosts.txt und führt gezielte Tests durch

# Datei mit den verwundbaren Hosts
HOST_FILE="attack_hosts.txt"

# Überprüfen, ob die Datei existiert
if [ ! -f "$HOST_FILE" ]; then
    echo "[ERROR] Die Datei $HOST_FILE wurde nicht gefunden!"
    exit 1
fi

# Tools für Tests definieren
PROXYCHAINS="proxychains"
TORIFY="torify"
NMAP_CMD="$PROXYCHAINS nmap -sT -PN -n -p 80,443"
NIKTO_CMD="$PROXYCHAINS nikto -h"
CURL_CMD="$PROXYCHAINS curl -s -I"
WPSCAN_CMD="$PROXYCHAINS wpscan --url"

# Ergebnisse speichern
OUTPUT_DIR="recon_results"
mkdir -p "$OUTPUT_DIR"

# Funktion zur Erneuerung des Tor-Circuits
tor_new_circuit() {
    echo "[*] Wechsle Tor Exit-Node..."
    echo -e 'AUTHENTICATE ""\nSIGNAL NEWNYM\nQUIT' | nc localhost 9050
    sleep 10  # Warten, damit neuer Exit-Node aktiv ist
}

# Scans durchführen
while read -r HOST; do
    echo "[+] Scanning $HOST over Tor/Proxychains..."

    # Tor Circuit erneuern
    tor_new_circuit

    # Nmap-Scan für Web-Schwachstellen
    echo "  -> Running Nmap scan..."
    $NMAP_CMD "$HOST" -oN "$OUTPUT_DIR/nmap_$HOST.txt"

    # Nikto Webserver-Security-Scan
    echo "  -> Running Nikto scan..."
    $NIKTO_CMD "$HOST" > "$OUTPUT_DIR/nikto_$HOST.txt"

    # HTTP-Header analysieren
    echo "  -> Checking HTTP headers..."
    $CURL_CMD "http://$HOST" > "$OUTPUT_DIR/headers_$HOST.txt"

    # WordPress Scans (falls WP gefunden)
    if grep -q "wp-admin" "$OUTPUT_DIR/nikto_$HOST.txt"; then
        echo "  -> WordPress detected, running WPScan..."
        $WPSCAN_CMD "http://$HOST" > "$OUTPUT_DIR/wpscan_$HOST.txt"
    fi

    echo "[+] Scan for $HOST completed!"
done < "$HOST_FILE"

# Ergebnisse zusammenfassen
echo "[+] Recon abgeschlossen! Ergebnisse gespeichert in $OUTPUT_DIR"
