# Konfigurasi Proxy Server (Squid) di Debian 10

Dokumentasi konfigurasi Proxy Server menggunakan Squid di Debian 10, termasuk fitur filtering domain dan kata kunci.

## Apa itu Proxy Server?

Proxy adalah server/program yang menjadi perantara antara client dengan internet, meneruskan permintaan client yang berhubungan dengan internet.

**Fungsi Proxy:**
- Perantara pengambilan data antar alamat IP
- Filtering data (blokir IP/website tertentu)
- Menyimpan cache agar koneksi ke website lebih cepat

**Squid** adalah proxy caching berfitur lengkap yang mendukung protokol jaringan populer seperti HTTP, HTTPS, FTP, dan lainnya.

## Topology

```
[ Internet ] -- Bridged Adapter -- [ Proxy Server ] -- Internal Network -- [ Client ]
                                    192.168.28.1/24        192.168.28.2/24
```

- **Proxy Server**: Adapter 1 → Bridged Adapter (ke internet), Adapter 2 → Internal Network (ke client)
- **Client**: Adapter 1 → Internal Network (sama dengan sisi server)

## Langkah-Langkah Konfigurasi

### 1. Persiapan
Siapkan 2 Sistem Operasi Debian: 1 sebagai Server, 1 sebagai Client (berbasis GUI).

### 2. Konfigurasi Network di VirtualBox

**Server:**
- Adapter 1 → Bridged Adapter
- Adapter 2 → Internal Network (nama: `LAN1`)

**Client:**
- Adapter 1 → Internal Network (nama: `LAN1`, harus sama dengan server)

### 3. Konfigurasi Network Interface di Server
```bash
nano /etc/network/interfaces
```
```
auto lo
iface lo inet loopback

# Interface ke internet
auto enp0s3
iface enp0s3 inet dhcp

# Interface ke client
auto enp0s8
iface enp0s8 inet static
    address 192.168.28.1
    netmask 255.255.255.0
```

### 4. Install Squid
```bash
apt-get install squid -y
```

### 5. Konfigurasi Squid
```bash
nano /etc/squid/squid.conf
```

Gunakan `Ctrl+W` untuk mencari, lakukan perubahan berikut:

| Cari | Aksi |
|---|---|
| `http_port 3128` | Pastikan tidak ada `#` di depannya |
| `cache_mem 256 MB` | Hapus `#` di depannya |
| `cache_mgr` | Hapus `#`, tambahkan alamat email |
| `Automatically detect the system host name` | Tambahkan baris `visible_hostname anonymous` di atasnya |
| `http_access deny all` | Tambahkan `#` di depannya (comment out) |
| `acl CONNECT` | Tambahkan script ACL di bawahnya (lihat bawah) |

**Tambahan ACL untuk filtering:**
```
acl local src 192.168.28.0/24
acl blocklist dstdomain "/etc/squid/domain.txt"
acl katakunci url_regex -i "/etc/squid/katakunci.txt"

http_access deny blocklist
http_access deny katakunci
http_access allow local
```

Simpan dengan `Ctrl+X` → `Y` → `Enter`.

### 6. Buat File Filtering

**Daftar domain yang diblokir:**
```bash
nano /etc/squid/domain.txt
```
```
facebook.com
detik.com
```

**Daftar kata kunci terkait domain yang diblokir:**
```bash
nano /etc/squid/katakunci.txt
```
```
facebook
detik
```

### 7. Restart Squid
```bash
/etc/init.d/squid restart
```
Pastikan status `OK` tanpa error.

### 8. Aktifkan IP Forwarding
```bash
nano /etc/sysctl.conf
```
Hapus `#` pada baris:
```
net.ipv4.ip_forward=1
```

### 9. Konfigurasi NAT & Redirect Port
```bash
iptables -t nat -A POSTROUTING -j MASQUERADE
iptables -t nat -A PREROUTING -s 192.168.28.0/24 -p tcp --dport 80 -j REDIRECT --to-port 3128
```
- **MASQUERADE**: agar client bisa terhubung ke internet
- **REDIRECT**: mengarahkan trafik HTTP (port 80) ke port proxy (3128)

### 10. Konfigurasi Network di Client
```bash
nano /etc/network/interfaces
```
```
auto lo
iface lo inet loopback

auto enp0s3
iface enp0s3 inet static
    address 192.168.28.2
    netmask 255.255.255.0
    gateway 192.168.28.1
    dns-nameserver 8.8.8.8
```

### 11. Uji Koneksi
Ping dari client ke server dan ke internet (google.com), pastikan berhasil.

### 12. Setting Proxy di Browser (Client)

Karena tanpa setting proxy di browser, filtering **belum berlaku** (client masih bisa akses domain yang diblokir). Untuk mengaktifkan:

1. Buka Firefox → menu (☰) → **Preferences**
2. Scroll ke **Network Settings** → klik **Settings**
3. Pilih **Manual Proxy Configuration**
4. **HTTP Proxy**: isi IP Server Squid (`192.168.28.1`), **Port**: `3128`
5. Centang **Use this proxy server for all protocols**
6. Klik **OK**

## Hasil Pengujian

- Akses `facebook.com` dan `detik.com` → **Diblokir** (muncul halaman error "Access Denied")
- Akses `youtube.com` atau `kompas.com` → **Masih bisa diakses** (tidak masuk daftar blokir)

```
ERROR
The requested URL could not be retrieved
Access Denied.
```
