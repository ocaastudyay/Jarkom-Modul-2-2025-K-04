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


Gunakan script bash dan coba gunakan tampilan yang interakttif agar lebih mudah
```bash
#!/bin/bash

# =================================================================
# Script Otomatisasi Setup DNS Master, Slave, dan Client
# DOMAIN: k16.com
# =================================================================

# --- Variabel Warna ANSI ---
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[0;33m'
BLUE='\033[0;34m'
NC='\033[0m' # No Color

# --- Variabel Konfigurasi (sesuai soal) ---
DOMAIN="k16.com"
MASTER_IP="192.213.3.3"   # IP Tirion (ns1)
SLAVE_IP="192.213.3.4"    # IP Valmar (ns2)
WEB_IP="192.213.3.2"      # IP Sirion (website)
FORWARDER_IP="192.168.122.1"

# --- Fungsi Utility ---

# Pengecekan hak akses root
check_root() {
    if [ "$(id -u)" -ne 0 ]; then
        echo -e "${RED}❌ GAGAL: Script ini harus dijalankan sebagai root (atau menggunakan sudo).${NC}"
        exit 1
    fi
}

# Fungsi restart bind9 (lebih robust)
restart_bind9() {
    echo -n "  -> Me-restart service BIND9..."
    if command -v systemctl >/dev/null 2>&1; then
        systemctl restart bind9
    elif command -v service >/dev/null 2>&1; then
        service bind9 restart
    elif [ -x /etc/init.d/bind9 ]; then
        /etc/init.d/bind9 restart
    else
        echo -e "\n${RED}❌ GAGAL: Tidak dapat me-restart bind9: perintah service/init.d/systemctl tidak tersedia.${NC}"
        exit 1
    fi
    if [ $? -eq 0 ]; then
        echo -e "${GREEN}BERHASIL!${NC}"
    else
        echo -e "${RED}GAGAL!${NC}"
    fi
}

# Fungsi Install BIND9
install_bind9() {
    echo -n "  -> Mengupdate list paket dan menginstall BIND9..."
    apt-get update > /dev/null 2>&1
    apt-get install -y bind9 dnsutils > /dev/null 2>&1
    if [ $? -eq 0 ]; then
        echo -e "${GREEN}BERHASIL!${NC}"
    else
        echo -e "${RED}GAGAL! Periksa koneksi internet atau repository.${NC}"
        exit 1
    fi
}

# --- Main Program ---
check_root

# Tampilan menu utama
echo -e "${BLUE}====================================================${NC}"
echo -e "${YELLOW}       🚀 Setup Otomatisasi DNS - Domain $DOMAIN${NC}"
echo -e "${BLUE}====================================================${NC}"
echo -e "Pilih peran mesin ini:"
echo -e "  ${GREEN}1) Setup Master DNS Server (Tirion - $MASTER_IP)${NC}"
echo -e "  ${GREEN}2) Setup Slave DNS Server (Valmar - $SLAVE_IP)${NC}"
echo -e "  ${GREEN}3) Setup Client Host (Resolver)${NC}"
echo -e "----------------------------------------------------"
read -p "Masukkan pilihan (1, 2, atau 3): " opsi

# Validasi Pilihan
if ! [[ "$opsi" =~ ^[1-3]$ ]]; then
    echo -e "${RED}❌ ERROR: Pilihan tidak valid. Harap masukkan 1, 2, atau 3.${NC}"
    exit 1
fi

case $opsi in
    1)
        # -----------------------------------------------------------------
        # Opsi 1: Setup untuk Master DNS (Tirion)
        # -----------------------------------------------------------------
        echo -e "\n${BLUE}<<< [1/3] Setup Master DNS Server (Tirion) >>>${NC}"

        # Langkah 1: Instalasi
        install_bind9

        # Langkah 2: Konfigurasi named.conf.options
        echo -e "\n${YELLOW}* Langkah 2: Konfigurasi named.conf.options${NC}"
        echo -n "  -> Mengatur forwarder dan allow-transfer..."
        cat <<EOF > /etc/bind/named.conf.options
options {
    directory "/var/cache/bind";
    listen-on { any; };
    listen-on-v6 { any; };
    forwarders {
        $FORWARDER_IP;
    };
    allow-query { any; };
    recursion yes;
};
EOF
        echo -e "${GREEN}BERHASIL! (Forwarder: $FORWARDER_IP)${NC}"

        # Langkah 3: Mendaftarkan zona master di named.conf.local
        echo -e "\n${YELLOW}* Langkah 3: Konfigurasi named.conf.local${NC}"
        echo -n "  -> Mendaftarkan zona master $DOMAIN..."
        cat <<EOF > /etc/bind/named.conf.local
// Konfigurasi Zona Master $DOMAIN
zone "$DOMAIN" {
    type master;
    file "/etc/bind/k16/$DOMAIN";
    notify yes;
    allow-transfer { $SLAVE_IP; }; // Izinkan transfer ke Valmar
};
EOF
        echo -e "${GREEN}BERHASIL! (Slave IP: $SLAVE_IP)${NC}"

        # Langkah 4: Membuat file zona
        echo -e "\n${YELLOW}* Langkah 4: Membuat File Zona ($DOMAIN)${NC}"
        mkdir -p /etc/bind/k16
        echo -n "  -> Membuat entri A record (ns1, ns2, @)..."
        cat <<EOF > /etc/bind/k16/$DOMAIN
\$TTL    604800
@       IN      SOA     ns1.$DOMAIN. root.$DOMAIN. (
                        2025102201 ; Serial (YYYYMMDDNN)
                        604800     ; Refresh
                        86400      ; Retry
                        2419200    ; Expire
                        604800 )   ; Negative Cache TTL

; Name Servers
@       IN      NS      ns1.$DOMAIN.
@       IN      NS      ns2.$DOMAIN.

; A Records
@       IN      A       $WEB_IP     ; Apex domain (@) menunjuk ke Sirion
ns1     IN      A       $MASTER_IP  ; ns1 menunjuk ke Tirion
ns2     IN      A       $SLAVE_IP   ; ns2 menunjuk ke Valmar
www     IN      A       $WEB_IP
EOF
        echo -e "${GREEN}BERHASIL! (Web IP: $WEB_IP)${NC}"
        
        # Langkah 5: Verifikasi dan Restart
        echo -e "\n${YELLOW}* Langkah 5: Verifikasi dan Restart Service${NC}"
        echo -n "  -> Memeriksa file konfigurasi..."
        named-checkconf 2>/dev/null
        if [ $? -eq 0 ]; then
            echo -e "${GREEN}named.conf OK!${NC}"
        else
            echo -e "${RED}named-checkconf GAGAL!${NC}"
            exit 1
        fi
        
        echo -n "  -> Memeriksa zona file..."
        named-checkzone $DOMAIN /etc/bind/k16/$DOMAIN 2>/dev/null
        if [ $? -eq 0 ]; then
            echo -e "${GREEN}Zone file OK!${NC}"
        else
            echo -e "${RED}named-checkzone GAGAL!${NC}"
            exit 1
        fi
        
        restart_bind9
        
        echo -e "\n${GREEN}====================================================${NC}"
        echo -e "${GREEN}✅ Setup Master DNS Server (Tirion) SELESAI.${NC}"
        echo -e "${GREEN}====================================================${NC}"
        ;;

    2)
        # -----------------------------------------------------------------
        # Opsi 2: Setup untuk Slave DNS (Valmar)
        # -----------------------------------------------------------------
        echo -e "\n${BLUE}<<< [2/3] Setup Slave DNS Server (Valmar) >>>${NC}"

        # Langkah 1: Instalasi
        install_bind9
        
        # Langkah 2: Buat symlink jika diperlukan (agar fungsi restart_bind9 berjalan)
        if [ ! -e /etc/init.d/bind9 ] && [ -x /etc/init.d/named ]; then
             echo -n "  -> Membuat symlink bind9 ke named..."
             ln -s /etc/init.d/named /etc/init.d/bind9
             echo -e "${GREEN}BERHASIL!${NC}"
        fi

        # Langkah 3: Mendaftarkan zona slave di named.conf.local
        echo -e "\n${YELLOW}* Langkah 3: Konfigurasi named.conf.local${NC}"
        echo -n "  -> Mendaftarkan zona slave $DOMAIN..."
        cat <<EOF > /etc/bind/named.conf.local
// Konfigurasi Zona Slave $DOMAIN
zone "$DOMAIN" {
    type slave;
    file "/var/cache/bind/$DOMAIN.zone"; // Lokasi default cache di Debian/Ubuntu
    masters { $MASTER_IP; }; // Menarik data dari Tirion
};
EOF
        echo -e "${GREEN}BERHASIL! (Master IP: $MASTER_IP)${NC}"

        # Langkah 4: Restart Service
        echo -e "\n${YELLOW}* Langkah 4: Restart Service${NC}"
        restart_bind9
        
        echo -e "\n${YELLOW}💡 INFO: Periksa log untuk transfer zona!${NC}"
        echo -e "   ${BLUE}journalctl -u bind9 -f${NC}"

        echo -e "\n${GREEN}====================================================${NC}"
        echo -e "${GREEN}✅ Setup Slave DNS Server (Valmar) SELESAI.${NC}"
        echo -e "${GREEN}====================================================${NC}"
        ;;

    3)
        # -----------------------------------------------------------------
        # Opsi 3: Setup untuk Client (Resolver)
        # -----------------------------------------------------------------
        echo -e "\n${BLUE}<<< [3/3] Setup Client Host (Resolver) >>>${NC}"
        
        # Langkah 1: Konfigurasi /etc/resolv.conf
        echo -e "\n${YELLOW}* Langkah 1: Mengkonfigurasi /etc/resolv.conf${NC}"
        echo -n "  -> Menulis nameserver..."
        cat <<EOF > /etc/resolv.conf
nameserver $MASTER_IP
nameserver $SLAVE_IP
nameserver $FORWARDER_IP
EOF
        echo -e "${GREEN}BERHASIL!${NC}"
        
        echo -e "\n${YELLOW}* Status Resolver:${NC}"
        echo -e "  1. Tirion (Master): ${GREEN}$MASTER_IP${NC}"
        echo -e "  2. Valmar (Slave): ${GREEN}$SLAVE_IP${NC}"
        echo -e "  3. Fallback: ${GREEN}$FORWARDER_IP${NC}"

        # Langkah 2: Setup Persisten untuk /root/.bashrc (Opsional, tapi ini salah)
        # LOGIKA ASLI DIBAWAH INI TIDAK BENAR. resolv.conf TIDAK PERLU DITIMPA SETIAP LOGIN.
        # Saya asumsikan Anda ingin MENGINSTALL `dnsutils` untuk testing.
        
        echo -e "\n${YELLOW}* Langkah 2: Instalasi DNS Utility${NC}"
        echo -n "  -> Menginstall dnsutils (dig, nslookup)..."
        apt-get update > /dev/null 2>&1
        apt-get install -y dnsutils > /dev/null 2>&1
        if [ $? -eq 0 ]; then
             echo -e "${GREEN}BERHASIL!${NC}"
        else
             echo -e "${YELLOW}WARNING: dnsutils gagal diinstall. Lanjut...${NC}"
        fi
        
        # Menghapus perbaikan yang SALAH di resolv.conf dari .bashrc
        sed -i '/nameserver 192\.213\.3\.3/d' /root/.bashrc
        sed -i '/nameserver 192\.213\.3\.4/d' /root/.bashrc
        sed -i '/nameserver 192\.168\.122\.1/d' /root/.bashrc

        echo -e "\n${GREEN}====================================================${NC}"
        echo -e "${GREEN}✅ Setup Client Host SELESAI.${NC}"
        echo -e "${GREEN}Anda dapat menguji dengan 'dig $DOMAIN'${NC}"
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
