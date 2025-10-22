# Jarkom-Modul-2-2025-K-04

Anggota :
| Nama | NRP |
| -------------------- | ---------- |
| Diva Aulia Rosa | 5027241003 |

## Question 1

> Di tepi Beleriand yang porak-poranda, Eonwe merentangkan tiga jalur: Barat untuk Earendil dan Elwing, Timur untuk Círdan, Elrond, Maglor, serta pelabuhan DMZ bagi Sirion, Tirion, Valmar, Lindon, Vingilot. Tetapkan alamat dan default gateway tiap tokoh sesuai glosarium yang sudah diberikan.

<img width="1614" height="867" alt="Screenshot 2025-10-22 194045" src="https://github.com/user-attachments/assets/b32d2395-14ff-4f3b-b52e-46942d11ea7d" />

Edit Network Configuration disetiap client, dan switch.


## Question 2

> Angin dari luar mulai berhembus ketika Eonwe membuka jalan ke awan NAT. Pastikan jalur WAN di router aktif dan NAT meneruskan trafik keluar bagi seluruh alamat internal sehingga host di dalam dapat mencapai layanan di luar menggunakan IP address.

Agar host internal dapat akses internet maka lakukan

```bash
apt update && apt install iptables -y
```
```bash
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE -s 192.213.0.0/16
```
```bash
echo nameserver 192.168.122.1 > /etc/resolv.conf
```
```bash
cat /etc/resolv.conf
```
```bash
ping google.com -c 5
```
<img width="1592" height="765" alt="Screenshot 2025-10-22 195302" src="https://github.com/user-attachments/assets/7db32b46-11aa-4a8e-b3f6-56e953950e5b" />


## Question 3

> Kabar dari Barat menyapa Timur. Pastikan kelima klien dapat saling berkomunikasi lintas jalur (routing internal via Eonwe berfungsi), lalu pastikan setiap host non-router menambahkan resolver 192.168.122.1 saat interfacenya aktif agar akses paket dari internet tersedia sejak awal.

Tambahkan configuration di bawah ini disetiap klien.
```bash
echo "nameserver 192.168.122.1" > /etc/resolv.conf
```
<img width="1111" height="706" alt="image" src="https://github.com/user-attachments/assets/35079043-1e2a-49a8-92fa-8b4f89b2532e" />

Untuk memastikan agar semua klien bisa saling berkomunikasi atau tersambung ke internet melalui Eonwe menggunakan ping ataupun telnet.

- ping (Aerendil address 192.213.1.2)
  <img width="1059" height="281" alt="Screenshot 2025-10-22 200220" src="https://github.com/user-attachments/assets/e7135cec-81e2-496c-a526-4670197edcad" />
- telnet (Cirdan)
  <img width="968" height="267" alt="Screenshot 2025-10-22 200416" src="https://github.com/user-attachments/assets/89403286-2d6c-44f5-b599-25105307d4e6" />


## Question 4

> Para penjaga nama naik ke menara, di Tirion (ns1/master) bangun zona <xxxx>.com sebagai authoritative dengan SOA yang menunjuk ke ns1.<xxxx>.com dan catatan NS untuk ns1.<xxxx>.com dan ns2.<xxxx>.com. Buat A record untuk ns1.<xxxx>.com dan ns2.<xxxx>.com yang mengarah ke alamat Tirion dan Valmar sesuai glosarium, serta A record apex <xxxx>.com yang mengarah ke alamat Sirion (front door), aktifkan notify dan allow-transfer ke Valmar, set forwarders ke 192.168.122.1. Di Valmar (ns2/slave) tarik zona <xxxx>.com dari Tirion dan pastikan menjawab authoritative. pada seluruh host non-router ubah urutan resolver menjadi IP dari ns1.<xxxx>.com → ns2.<xxxx>.com → 192.168.122.1. Verifikasi query ke apex dan hostname layanan dalam zona dijawab melalui ns1/ns2.

maksut ringkasnya adalah mengkonfigurasi DNS rekursif dan otoritatif terdistribusi di dua server, Tirion (master/ns1) dan Valmar (slave/ns2)

Gunakan script bash dan coba gunakan tampilan yang interakttif agar lebih mudah
```bash
#!/bin/bash

# =================================================================
# Script Otomatisasi Setup DNS Master, Slave, dan Client (FINAL PRODUCTION READY)
# DOMAIN: k16.com
# FIX: Mengatasi SERVFAIL (masalah kepemilikan/izin zona) dan menguji ulang named-checkzone.
# =================================================================

# --- Variabel Warna ANSI ---
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[0;33m'
BLUE='\033[0;34m'
NC='\033[0m' 

# --- Variabel Konfigurasi ---
DOMAIN="k16.com"
MASTER_IP="192.213.3.3"
SLAVE_IP="192.213.3.4"
WEB_IP="192.213.3.2"
FORWARDER_IP="192.168.122.1"
ZONE_DIR="/etc/bind" 

# --- Fungsi Utility ---

check_root() {
    if [ "$(id -u)" -ne 0 ]; then
        echo -e "${RED}❌ GAGAL: Script ini harus dijalankan sebagai root.${NC}"
        exit 1
    fi
}

# Fungsi restart bind9 dengan FALLBACK MANUAL yang terkuat
restart_bind9() {
    echo -n "  -> Me-restart service BIND9..."
    
    local status=1
    
    # 1. Pastikan tidak ada proses named lama yang mengganggu
    killall named 2>/dev/null 
    sleep 1 

    # 2. Coba jalankan named secara manual (fallback untuk non-systemd/non-init.d)
    /usr/sbin/named -c /etc/bind/named.conf & 2>/dev/null
    sleep 2 
    status=$? 

    # 3. Final Check: Pastikan Port 53 digunakan
    if [ $status -eq 0 ]; then
        if netstat -tuln | grep -q ':53'; then
            echo -e "${GREEN}BERHASIL!${NC}"
            return 0
        else
            status=1
        fi
    fi

    # Hasil Akhir
    if [ $status -eq 0 ]; then
        echo -e "${GREEN}BERHASIL!${NC}"
    else
        echo -e "${RED}GAGAL!${NC}"
        echo -e "${YELLOW}💡 INFO: BIND gagal memulai. Cek konflik Port 53: ${BLUE}netstat -tuln | grep ':53'${NC}${NC}"
        exit 1
    fi
}

install_bind9() {
    echo -n "  -> Mengupdate list paket dan menginstall BIND9 & dnsutils..."
    apt-get update > /dev/null 2>&1
    apt-get install -y bind9 dnsutils net-tools telnet > /dev/null 2>&1
    if [ $? -eq 0 ]; then
        echo -e "${GREEN}BERHASIL!${NC}"
    else
        echo -e "${RED}GAGAL! Periksa koneksi internet atau repository.${NC}"
        exit 1
    fi
}

# --- Main Program ---
check_root

echo -e "${BLUE}====================================================${NC}"
echo -e "${YELLOW}        🚀 Setup Otomatisasi DNS - Domain $DOMAIN (FINAL)${NC}"
echo -e "${BLUE}====================================================${NC}"
echo -e "Pilih peran mesin ini:"
echo -e "  ${GREEN}1) Setup Master DNS Server (Tirion - $MASTER_IP)${NC}"
echo -e "  ${GREEN}2) Setup Slave DNS Server (Valmar - $SLAVE_IP)${NC}"
echo -e "  ${GREEN}3) Setup Client Host (Resolver)${NC}"
echo -e "----------------------------------------------------"
read -p "Masukkan pilihan (1, 2, atau 3): " opsi

if ! [[ "$opsi" =~ ^[1-3]$ ]]; then
    echo -e "${RED}❌ ERROR: Pilihan tidak valid. Harap masukkan 1, 2, atau 3.${NC}"
    exit 1
fi

case $opsi in
    1)
        # Opsi 1: Setup untuk Master DNS (Tirion)
        echo -e "\n${BLUE}<<< [1/7] Setup Master DNS Server (Tirion) >>>${NC}"
        install_bind9

        # Langkah 2: Konfigurasi named.conf.options 
        echo -e "\n${YELLOW}* Langkah 2: Konfigurasi named.conf.options${NC}"
        echo -n "  -> Mengatur forwarder ($FORWARDER_IP)..."
        cat <<EOF > /etc/bind/named.conf.options
options {
    directory "/var/cache/bind";
    listen-on-v6 { none; };
    forwarders {
        $FORWARDER_IP;
    };
    allow-query { any; };
    recursion yes;
};
EOF
        echo -e "${GREEN}BERHASIL!${NC}"
        
        # FIX A: Membuat named.conf.default-zones kosong
        echo -n "  -> Membuat named.conf.default-zones kosong..."
        touch /etc/bind/named.conf.default-zones
        chmod 644 /etc/bind/named.conf.default-zones
        echo -e "${GREEN}BERHASIL!${NC}"


        # Langkah 3: Konfigurasi named.conf (File Utama)
        echo -e "\n${YELLOW}* Langkah 3: Konfigurasi named.conf (File Utama) [FINAL]${NC}"
        echo -n "  -> Membuat entry point named.conf..."
        # FIX B: Mengembalikan include default-zones karena file sudah dibuat
        cat <<EOF > /etc/bind/named.conf
include "/etc/bind/named.conf.options";
include "/etc/bind/named.conf.local";
include "/etc/bind/named.conf.default-zones";
EOF
        echo -e "${GREEN}BERHASIL!${NC}"
        
        # Langkah 4: Mendaftarkan zona master di named.conf.local
        echo -e "\n${YELLOW}* Langkah 4: Konfigurasi named.conf.local${NC}"
        echo -n "  -> Mendaftarkan zona otoritatif master $DOMAIN..."
        cat <<EOF > /etc/bind/named.conf.local
// Konfigurasi Zona Master Otoritatif $DOMAIN
zone "$DOMAIN" {
    type master;
    file "$ZONE_DIR/$DOMAIN.zone";
    notify yes;
    allow-transfer { $SLAVE_IP; };
};
EOF
        echo -e "${GREEN}BERHASIL!${NC}"

        # Langkah 5: Membuat file zona
        echo -e "\n${YELLOW}* Langkah 5: Membuat File Zona ($DOMAIN) dan Izin${NC}"
        echo -n "  -> Membuat entri SOA, NS, dan A record..."

        cat <<EOF > $ZONE_DIR/$DOMAIN.zone
\$TTL    604800
@       IN      SOA     ns1.$DOMAIN. root.$DOMAIN. (
                        2025102311 ; Serial (Dinaikkan untuk reload)
                        604800     ; Refresh
                        86400      ; Retry
                        2419200    ; Expire
                        604800 )   ; Negative Cache TTL

; Name Servers (NS)
@       IN      NS      ns1.$DOMAIN.
@       IN      NS      ns2.$DOMAIN.

; A Records
@       IN      A       $WEB_IP     
ns1     IN      A       $MASTER_IP 
ns2     IN      A       $SLAVE_IP 
www     IN      A       $WEB_IP
EOF
        # FIX C: Memastikan Kepemilikan dan Izin Benar untuk BIND
        chown root:bind $ZONE_DIR/$DOMAIN.zone
        chmod 644 $ZONE_DIR/$DOMAIN.zone
        echo -e "${GREEN}FILE ZONA & IZIN BERHASIL!${NC}"

        # Langkah 6: Verifikasi Konfigurasi
        echo -e "\n${YELLOW}* Langkah 6: Verifikasi Konfigurasi${NC}"
        echo -n "  -> Memeriksa named.conf..."
        named-checkconf /etc/bind/named.conf
        if [ $? -ne 0 ]; then echo -e "${RED}named-checkconf GAGAL!${NC}"; exit 1; else echo -e "${GREEN}named.conf OK!${NC}"; fi
        
        echo -n "  -> Memeriksa zona file..."
        named-checkzone $DOMAIN $ZONE_DIR/$DOMAIN.zone
        if [ $? -ne 0 ]; then echo -e "${RED}named-checkzone GAGAL!${NC}"; exit 1; else echo -e "${GREEN}Zone file OK!${NC}"; fi
        
        # Langkah 7: Restart Service (FALLBACK)
        echo -e "\n${YELLOW}* Langkah 7: Restart Service (FALLBACK MANUAL)...${NC}"
        restart_bind9 
        
        echo -e "\n${GREEN}====================================================${NC}"
        echo -e "${GREEN}✅ Setup Master DNS Server (Tirion) SELESAI.${NC}" 
        echo -e "${GREEN}   VERIFIKASI: dig @127.0.0.1 k16.com SOA${NC}" 
        echo -e "${GREEN}====================================================${NC}"
        ;;

    2)
        # Opsi 2: Setup untuk Slave DNS (Valmar)
        echo -e "\n${BLUE}<<< [2/3] Setup Slave DNS Server (Valmar) >>>${NC}"
        install_bind9
        
        # Langkah 2: Konfigurasi named.conf.local
        echo -e "\n${YELLOW}* Langkah 2: Konfigurasi named.conf.local${NC}"
        echo -n "  -> Mendaftarkan zona slave $DOMAIN..."
        
        # named.conf di Slave (Membuat default-zones kosong dan meng-include)
        touch /etc/bind/named.conf.default-zones
        chmod 644 /etc/bind/named.conf.default-zones
        cat <<EOF > /etc/bind/named.conf
include "/etc/bind/named.conf.options";
include "/etc/bind/named.conf.local";
include "/etc/bind/named.conf.default-zones";
EOF

        cat <<EOF > /etc/bind/named.conf.options
options {
    directory "/var/cache/bind";
    listen-on-v6 { none; }; 
    allow-query { any; };
    recursion yes;
};
EOF

        cat <<EOF > /etc/bind/named.conf.local
// Konfigurasi Zona Slave $DOMAIN
zone "$DOMAIN" {
    type slave;
    file "/var/cache/bind/$DOMAIN.zone";
    masters { $MASTER_IP; };
};
EOF
        echo -e "${GREEN}BERHASIL! (Master IP: $MASTER_IP)${NC}"

        # Langkah 3: Restart Service dan Verifikasi
        echo -e "\n${YELLOW}* Langkah 3: Restart Service${NC}"
        restart_bind9 
        
        echo -e "\n${YELLOW}* Verifikasi Zone Transfer${NC}"
        echo -e "💡 INFO: Valmar akan menarik zona dari Tirion. Gunakan ${BLUE}dig k16.com @127.0.0.1${NC} untuk memverifikasi."

        echo -e "\n${GREEN}====================================================${NC}"
        echo -e "${GREEN}✅ Setup Slave DNS Server (Valmar) SELESAI.${NC}"
        echo -e "${GREEN}====================================================${NC}"
        ;;

    3)
        # Opsi 3: Setup untuk Client (Resolver)
        echo -e "\n${BLUE}<<< [3/3] Setup Client Host (Resolver) >>>${NC}"
        
        # Langkah 1: Konfigurasi /etc/resolv.conf
        echo -e "\n${YELLOW}* Langkah 1: Mengkonfigurasi /etc/resolv.conf${NC}"
        echo -n "  -> Menulis nameserver (ns1 -> ns2 -> Forwarder)..."
        
        cat <<EOF > /etc/resolv.conf
nameserver $MASTER_IP
nameserver $SLAVE_IP
nameserver $FORWARDER_IP
search $DOMAIN
EOF
        echo -e "${GREEN}BERHASIL!${NC}"
        
        # Langkah 2: Instalasi DNS Utility
        echo -e "\n${YELLOW}* Langkah 2: Instalasi DNS Utility${NC}"
        echo -n "  -> Menginstall dnsutils, telnet, dan net-tools..."
        apt-get update > /dev/null 2>&1
        apt-get install -y dnsutils net-tools telnet > /dev/null 2>&1
        if [ $? -eq 0 ]; then
             echo -e "${GREEN}BERHASIL!${NC}"
        else
             echo -e "${YELLOW}WARNING: Utility gagal diinstall. Lanjut...${NC}"
        fi
        
        echo -e "\n${GREEN}====================================================${NC}"
        echo -e "${GREEN}✅ Setup Client Host SELESAI.${NC}"
        echo -e "${GREEN}====================================================${NC}"
        ;;
esac

echo -e "\n${BLUE}====================================================${NC}"
echo -e "${YELLOW}Terima kasih! Skrip selesai dijalankan. ✨${NC}"
echo -e "${BLUE}====================================================${NC}"
exit 0
```
<img width="1011" height="408" alt="Screenshot 2025-10-22 202718" src="https://github.com/user-attachments/assets/9e52597f-3a28-44ad-97d9-017f826a8b70" />

Ketika Setup Master DNS Server (Tirion - 192.213.3.3) berhasil dilakukan
<img width="1376" height="956" alt="image" src="https://github.com/user-attachments/assets/d1e0e2b1-cdae-4a0a-a223-ab15aa6c7ffb" />
Untuk memastikan bahwa setup berjalan dan berhasil bisa menggunakan 
```bash
/etc/init.d/bind9 status
named-checkconf
named-checkzone k16.com /etc/bind/k16/k16.com
```
<img width="896" height="269" alt="Screenshot 2025-10-22 204332" src="https://github.com/user-attachments/assets/712f0641-87ba-4f2d-9fe5-83cab3ec5506" />

Ketika Setup Slave DNS Server (Valmar - 192.213.3.4) berhasil dilakukan
<img width="1405" height="696" alt="Screenshot 2025-10-22 203536" src="https://github.com/user-attachments/assets/1a50b9e3-32da-4d44-9ef3-f50c687c5772" />

Ketika Setup Client Host (Resolver) berhasil
<img width="998" height="660" alt="Screenshot 2025-10-22 203800" src="https://github.com/user-attachments/assets/08675ce2-ee6e-4c2b-8ac7-469eb7c1270a" />

Untuk memastikan semuanya berjalan dengan lancar coba cek dengan beberapa bash di bawah ini pada valmar
```bash
/etc/init.d/bind9 status
ls -l /var/cache/bind/k16.com.zone
dig k16.com @127.0.0.1
```
<img width="1267" height="857" alt="Screenshot 2025-10-22 212423" src="https://github.com/user-attachments/assets/b08e6ec2-6bb2-4520-992d-70b1fcc87911" />


## Question 5

> “Nama memberi arah,” kata Eonwe. Namai semua tokoh (hostname) sesuai glosarium, eonwe, earendil, elwing, cirdan, elrond, maglor, sirion, tirion, valmar, lindon, vingilot, dan verifikasi bahwa setiap host mengenali dan menggunakan hostname tersebut secara system-wide. Buat setiap domain untuk masing masing node sesuai dengan namanya (contoh: eru.<xxxx>.com) dan assign IP masing-masing juga. Lakukan pengecualian untuk node yang bertanggung jawab atas ns1 dan ns2


Gunakan Bash dibawah ini
```bash
#!/bin/bash

# =================================================================
# Script untuk Menambahkan A Records ke Zona k16.com (FINAL & INTERAKTIF)
# Dijalankan di: TIRION (DNS Master)
# =================================================================

# --- Variabel Warna ANSI ---
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[0;33m'
BLUE='\033[0;34m'
PURPLE='\033[0;35m'
NC='\033[0m' # No Color

# --- Variabel Konfigurasi ---
ZONE_FILE="/etc/bind/k16/k16.com"

# --- Fungsi Utility ---

# Pengecekan hak akses root
check_root() {
    if [ "$(id -u)" -ne 0 ]; then
        echo -e "${RED}❌ GAGAL: Script ini harus dijalankan sebagai root (atau menggunakan sudo).${NC}"
        exit 1
    fi
}

# Fungsi restart bind9 (menggunakan logika robust)
restart_bind9() {
    echo -n "  -> Me-restart service BIND9..."
    # Mencoba systemctl, service, lalu init.d
    if command -v systemctl >/dev/null 2>&1; then
        systemctl restart bind9
    elif command -v service >/dev/null 2>&1; then
        service bind9 restart
    elif [ -x /etc/init.d/bind9 ]; then
        /etc/init.d/bind9 restart
    else
        echo -e "\n${RED}❌ GAGAL: Tidak dapat me-restart bind9. Harap restart manual.${NC}"
        return 1
    fi
    if [ $? -eq 0 ]; then
        echo -e "${GREEN}BERHASIL!${NC}"
    else
        echo -e "${RED}GAGAL!${NC}"
        return 1
    fi
}

# --- Main Program ---
check_root

# Tampilan menu utama
echo -e "${BLUE}====================================================${NC}"
echo -e "${PURPLE}       💾 Script Penambah A Record DNS Master${NC}"
echo -e "${BLUE}====================================================${NC}"
echo -e "Hostname baru akan ditambahkan ke zona ${YELLOW}k16.com${NC}."
echo -e "File Zona: ${BLUE}$ZONE_FILE${NC}"
echo -e "IP yang digunakan: ${GREEN}192.213.x.x${NC}"
echo -e "----------------------------------------------------"

read -p "Tekan [Y/y] untuk MENAMBAH RECORD atau [N/n] untuk batal: " confirm

if [[ "$confirm" != "y" && "$confirm" != "Y" ]]; then
    echo -e "${YELLOW}Operasi dibatalkan oleh pengguna.${NC}"
    exit 0
fi

# --- Proses Penambahan Record ---

# 1. Cek apakah file zona ada
echo -e "\n${YELLOW}* Langkah 1: Pengecekan File Zona${NC}"
if [ ! -f "$ZONE_FILE" ]; then
    echo -e "${RED}❌ GAGAL: File zona $ZONE_FILE tidak ditemukan.${NC}"
    echo -e "Pastikan Anda menjalankan script ini di Tirion (Master) setelah setup awal."
    exit 1
fi
echo -e "${GREEN}BERHASIL: File zona ditemukan.${NC}"


# 2. Tambahkan A records baru
echo -e "\n${YELLOW}* Langkah 2: Menambahkan Record Baru${NC}"
echo -e "  -> Menulis A records ke akhir file... (Menggunakan 192.213.x.x)"
cat <<EOF >> "$ZONE_FILE"

; Penambahan host baru (Sesuai Glosarium: eonwe, earendil, elwing, cirdan, elrond, maglor, lindon, vingilot)
eonwe    IN      A       192.213.1.2
earendil IN      A       192.213.1.3
elwing   IN      A       192.213.2.2
cirdan   IN      A       192.213.2.3
elrond   IN      A       192.213.2.4
maglor   IN      A       192.213.3.5
lindon   IN      A       192.213.3.6
vingilot IN      A       192.213.3.7
EOF
echo -e "${GREEN}BERHASIL: 8 Hostname baru berhasil ditambahkan.${NC}"


# 3. Naikkan nomor serial secara otomatis (LOGIKA DIPERKUAT: ANTI-WHITESPACE)
echo -e "\n${YELLOW}* Langkah 3: Menaikkan Nomor Serial${NC}"

# Mencari baris yang berisi 'Serial' dan mengekstrak angka 10 digit pertama
CURRENT_SERIAL=$(grep -i 'Serial' "$ZONE_FILE" | grep -oE '[0-9]{10,}' | head -1)

if [[ ! "$CURRENT_SERIAL" =~ ^[0-9]+$ ]]; then
    echo -e "${RED}❌ GAGAL: Tidak dapat membaca Nomor Serial. Pastikan format di file zona benar.${NC}"
    exit 1
fi

NEW_SERIAL=$((CURRENT_SERIAL + 1))

# Mengganti angka serial lama dengan yang baru di file zona
sed -i "s/$CURRENT_SERIAL/$NEW_SERIAL/" "$ZONE_FILE"

echo -e "${GREEN}BERHASIL! Serial diperbarui dari $CURRENT_SERIAL menjadi $NEW_SERIAL.${NC}"


# 4. Restart layanan BIND9 dan Verifikasi
echo -e "\n${YELLOW}* Langkah 4: Restart Service & Verifikasi${NC}"

# 4a. Verifikasi Sintaks sebelum Restart
echo -n "  -> Memeriksa sintaks file zona..."
named-checkzone k16.com "$ZONE_FILE" 2>/dev/null
if [ $? -ne 0 ]; then
    echo -e "${RED}GAGAL! named-checkzone GAGAL. Harap periksa sintaks file $ZONE_FILE.${NC}"
    exit 1
fi
echo -e "${GREEN}Sintaks OK!${NC}"


# 4b. Restart
restart_bind9
if [ $? -ne 0 ]; then
    echo -e "${RED}Proses dihentikan karena GAGAL me-restart BIND9.${NC}"
    exit 1
fi


# --- Selesai ---
echo -e "\n${GREEN}====================================================${NC}"
echo -e "${GREEN}✅ Script Selesai Dijalankan (Tirion Updated).${NC}"
echo -e "${YELLOW}BIND9 akan mengirim notifikasi Zone Transfer. ${BLUE}Langkah selanjutnya adalah VERIFIKASI di Valmar dan Client!${NC}"
echo -e "${GREEN}====================================================${NC}"
exit 0
```
<img width="1736" height="954" alt="image" src="https://github.com/user-attachments/assets/87c82354-b034-4ec5-bb34-4c886b10e983" />
Untuk memastikan bahwa sudah ditambahkan bisa menggunakan
```bash
ls -l /var/cache/bind/k16.com.zone
```
jika berhasil maka ukuran file akan berubah


## Question 6

> Lonceng Valmar berdentang mengikuti irama Tirion. Pastikan zone transfer berjalan, Pastikan Valmar (ns2) telah menerima salinan zona terbaru dari Tirion (ns1). Nilai serial SOA di keduanya harus sama

Gunakan script bash di bawah ini
```bash
#!/bin/bash

# =================================================================
# Script Verifikasi KONSISTENSI Serial SOA (Interaktif)
# Digunakan untuk mengecek apakah Zone Transfer berhasil.
# IP KOREKSI: Tirion=192.213.3.3, Valmar=192.213.3.4
# =================================================================

# --- Variabel Warna ANSI ---
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[0;33m'
BLUE='\033[0;34m'
PURPLE='\033[0;35m'
NC='\033[0m' # No Color

# --- Variabel Konfigurasi ---
MASTER_IP="192.213.3.3"  # Tirion
SLAVE_IP="192.213.3.4"   # Valmar
DOMAIN="k16.com"

# --- Pengecekan Hak Akses Root ---
if [ "$(id -u)" -ne 0 ]; then
    echo -e "${RED}❌ GAGAL: Script ini harus dijalankan sebagai root atau dengan sudo.${NC}"
    exit 1
fi

# --- Fungsi Utility ---
cleanup() {
    rm -f /tmp/serial_valmar.txt /tmp/serial_tirion.txt
}

# --- Main Program ---
cleanup # Bersihkan file lama jika ada

# Tampilan menu utama
echo -e "${BLUE}====================================================${NC}"
echo -e "${PURPLE}       🔍 Script Verifikasi Konsistensi Serial SOA${NC}"
echo -e "${BLUE}====================================================${NC}"
echo -e "Script ini akan membandingkan Nomor Serial SOA:"
echo -e "  - ${YELLOW}Master (Tirion): ${MASTER_IP}${NC}"
echo -e "  - ${YELLOW}Slave (Valmar): ${SLAVE_IP}${NC}"
echo -e "----------------------------------------------------"

read -p "Tekan [Y/y] untuk MEMULAI PENGECKAN: " confirm

if [[ "$confirm" != "y" && "$confirm" != "Y" ]]; then
    echo -e "${YELLOW}Operasi dibatalkan oleh pengguna.${NC}"
    cleanup
    exit 0
fi

# --- Proses Cek Serial ---
echo -e "\n${YELLOW}* Langkah 1: Mengambil Serial dari Tirion (Master)${NC}"

# Mengambil serial dari Tirion menggunakan IP 192.213.3.3
dig "@$MASTER_IP" "$DOMAIN" SOA +short 2>/dev/null | awk '{print $3}' > /tmp/serial_tirion.txt

# Pengecekan status kueri
if [ $? -ne 0 ]; then
    echo -e "${RED}❌ GAGAL: Kueri ke Tirion gagal. Cek koneksi atau layanan BIND9 di Master.${NC}"
    cleanup
    exit 1
fi

echo -e "${YELLOW}* Langkah 2: Mengambil Serial dari Valmar (Slave)${NC}"

# Mengambil serial dari Valmar menggunakan IP 192.213.3.4
dig "@$SLAVE_IP" "$DOMAIN" SOA +short 2>/dev/null | awk '{print $3}' > /tmp/serial_valmar.txt

# Pengecekan status kueri
if [ $? -ne 0 ]; then
    echo -e "${RED}❌ GAGAL: Kueri ke Valmar gagal. Cek koneksi atau layanan BIND9 di Slave.${NC}"
    cleanup
    exit 1
fi


# --- Analisis dan Hasil ---
echo -e "\n${YELLOW}* Langkah 3: Analisis Hasil${NC}"

serial_tirion=$(cat /tmp/serial_tirion.txt)
serial_valmar=$(cat /tmp/serial_valmar.txt)

echo -e "  Serial Master (Tirion) : ${BLUE}$serial_tirion${NC}"
echo -e "  Serial Slave (Valmar)  : ${BLUE}$serial_valmar${NC}"

if [ -z "$serial_tirion" ] || [ -z "$serial_valmar" ]; then
    echo -e "${RED}❌ GAGAL: Tidak dapat mengambil Nomor Serial (Kemungkinan SERVFAIL).${NC}"
elif [ "$serial_valmar" == "$serial_tirion" ]; then
    echo -e "\n${GREEN}✅ SUKSES! Serial SOA di Tirion dan Valmar SAMA.${NC}"
    echo -e "${GREEN}Ini mengonfirmasi Zone Transfer telah berhasil!${NC}"
else
    echo -e "\n${RED}⚠️ PERINGATAN! Serial SOA di Tirion dan Valmar BERBEDA.${NC}"
    echo -e "  - Ini menandakan Zone Transfer BELUM berhasil atau tertunda."
    echo -e "  - Coba ${YELLOW}rndc retransfer k16.com${NC} di Valmar untuk memaksa transfer."
fi

# --- Selesai ---
cleanup
echo -e "\n${BLUE}====================================================${NC}"
echo -e "Pengecekan Serial Selesai."
echo -e "${BLUE}====================================================${NC}"
exit 0
```
Hasilnya :
<img width="1036" height="665" alt="Screenshot 2025-10-22 221803" src="https://github.com/user-attachments/assets/06b893de-c745-44fc-a125-dae9db5c6b79" />


## Question 7

> Peta kota dan pelabuhan dilukis. Sirion sebagai gerbang, Lindon sebagai web statis, Vingilot sebagai web dinamis. Tambahkan pada zona <xxxx>.com A record untuk sirion.<xxxx>.com (IP Sirion), lindon.<xxxx>.com (IP Lindon), dan vingilot.<xxxx>.com (IP Vingilot). Tetapkan CNAME :
 - www.<xxxx>.com → sirion.<xxxx>.com, 
 - static.<xxxx>.com → lindon.<xxxx>.com, dan 
 - app.<xxxx>.com → vingilot.<xxxx>.com.
Verifikasi dari dua klien berbeda bahwa seluruh hostname tersebut ter-resolve ke tujuan yang benar dan konsisten.

Gunakan script bash di bawah ini:
```bash
#!/bin/bash

# =================================================================
# Script FINAL Perbaikan CNAME: FOKUS pada SERIAL & RESTART
# Dijalankan di: TIRION (DNS Master)
# CATATAN: Konflik CNAME (GANDA) harus diperbaiki MANUAL di file zona.
# =================================================================

# --- Variabel Warna ANSI ---
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[0;33m'
BLUE='\033[0;34m'
PURPLE='\033[0;35m'
NC='\033[0m' # No Color

# --- Variabel Konfigurasi ---
ZONE_FILE="/etc/bind/k16/k16.com"

# --- Fungsi Utility ---

# Pengecekan hak akses root
check_root() {
    if [ "$(id -u)" -ne 0 ]; then
        echo -e "${RED}❌ GAGAL: Script ini harus dijalankan sebagai root (atau menggunakan sudo).${NC}"
        exit 1
    fi
}

# Fungsi restart bind9 (menggunakan logika robust)
restart_bind9() {
    echo -n "  -> Me-restart service BIND9..."
    # Mencoba systemctl, service, lalu init.d
    if command -v systemctl >/dev/null 2>&1; then
        systemctl restart bind9
    elif command -v service >/dev/null 2>&1; then
        service bind9 restart
    elif [ -x /etc/init.d/bind9 ]; then
        /etc/init.d/bind9 restart
    else
        echo -e "\n${RED}❌ GAGAL: Tidak dapat me-restart bind9. Harap restart manual.${NC}"
        return 1
    fi
    if [ $? -eq 0 ]; then
        echo -e "${GREEN}BERHASIL!${NC}"
    else
        echo -e "${RED}GAGAL!${NC}"
        return 1
    fi
}

# --- Main Program ---
check_root

# Tampilan menu utama
echo -e "${BLUE}====================================================${NC}"
echo -e "${PURPLE}       🔧 Script Perbaikan CNAME dan Peningkatan Serial${NC}"
echo -e "${BLUE}====================================================${NC}"
echo -e "Script ini akan menaikkan Serial Number dan me-restart BIND9."
echo -e "${RED}PASTIKAN Anda SUDAH memperbaiki konflik CNAME di file zona secara manual!${NC}"
echo -e "File Zona: ${BLUE}$ZONE_FILE${NC}"
echo -e "----------------------------------------------------"

read -p "Tekan [Y/y] untuk MELANJUTKAN: " confirm

if [[ "$confirm" != "y" && "$confirm" != "Y" ]]; then
    echo -e "${YELLOW}Operasi dibatalkan oleh pengguna.${NC}"
    exit 0
fi

# --- Proses Perbaikan & Restart ---

# 1. Cek apakah file zona ada
echo -e "\n${YELLOW}* Langkah 1: Pengecekan File Zona${NC}"
if [ ! -f "$ZONE_FILE" ]; then
    echo -e "${RED}❌ GAGAL: File zona $ZONE_FILE tidak ditemukan.${NC}"
    exit 1
fi
echo -e "${GREEN}BERHASIL: File zona ditemukan.${NC}"


# 2. Naikkan nomor serial secara otomatis (LOGIKA DIPERKUAT)
echo -e "\n${YELLOW}* Langkah 2: Menaikkan Nomor Serial${NC}"

# Mencari baris yang berisi 'Serial' dan mengekstrak angka 10 digit pertama
CURRENT_SERIAL=$(grep -i 'Serial' "$ZONE_FILE" | grep -oE '[0-9]{10,}' | head -1)

if [[ ! "$CURRENT_SERIAL" =~ ^[0-9]+$ ]]; then
    echo -e "${RED}❌ GAGAL: Tidak dapat membaca Nomor Serial. Harap periksa format di file zona.${NC}"
    exit 1
fi

NEW_SERIAL=$((CURRENT_SERIAL + 1))

# Mengganti angka serial lama dengan yang baru di file zona
sed -i "s/$CURRENT_SERIAL/$NEW_SERIAL/" "$ZONE_FILE"

echo -e "${GREEN}BERHASIL! Serial diperbarui dari $CURRENT_SERIAL menjadi $NEW_SERIAL.${NC}"


# 3. Restart layanan BIND9 dan Verifikasi
echo -e "\n${YELLOW}* Langkah 3: Restart Service & Verifikasi${NC}"

# 3a. Verifikasi Sintaks sebelum Restart
echo -n "  -> Memeriksa sintaks file zona..."
named-checkzone k16.com "$ZONE_FILE" 2>/dev/null
if [ $? -ne 0 ]; then
    echo -e "${RED}GAGAL! named-checkzone GAGAL. Perbaiki konflik CNAME di file zona dan jalankan lagi.${NC}"
    exit 1
fi
echo -e "${GREEN}Sintaks OK!${NC}"


# 3b. Restart
restart_bind9
if [ $? -ne 0 ]; then
    echo -e "${RED}Proses dihentikan karena GAGAL me-restart BIND9.${NC}"
    exit 1
fi


# --- Selesai ---
echo -e "\n${GREEN}====================================================${NC}"
echo -e "${GREEN}✅ Script Selesai Dijalankan (Tirion Diperbarui).${NC}"
echo -e "${YELLOW}Zone Transfer akan terpicu ke Valmar.${NC}"
echo -e "${BLUE}Langkah Selanjutnya: Uji CNAME (www.k16.com) di Slave!${NC}"
echo -e "${GREEN}====================================================${NC}"
exit 0
```
<img width="1600" height="855" alt="image" src="https://github.com/user-attachments/assets/4685857e-fce5-4f3e-a6f7-91b91ae707d0" />


## Question 8

> Setiap jejak harus bisa diikuti. Di Tirion (ns1) deklarasikan satu reverse zone untuk segmen DMZ tempat Sirion, Lindon, Vingilot berada. Di Valmar (ns2) tarik reverse zone tersebut sebagai slave, isi PTR untuk ketiga hostname itu agar pencarian balik IP address mengembalikan hostname yang benar, lalu pastikan query reverse untuk alamat Sirion, Lindon, Vingilot dijawab authoritative.

