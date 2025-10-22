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
