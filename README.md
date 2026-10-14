# Praktikum Jaringan Komputer — Modul 3

## Konfigurasi Jaringan, Routing, dan Layanan Infrastruktur

**Nama:** Zidny Ilman Nafi'an
**Nim:** 5027221072
**Kelompok:** K-65
**Mata Kuliah:** Praktikum Jaringan Komputer
**Topologi:** Jujutsu High — Shibuya Incident

---

# Bagian 1: DHCP & Konektivitas Jaringan (Soal 1–10)

## Soal 1 — Konfigurasi Router dan Konektivitas Dasar

> **Dikerjakan Oleh:** Zidny Ilman Nafi'an

### Soal

> Ketika Curtain menutup Shibuya, jalur komunikasi antar zona Jujutsu High terputus. Pulihkan konektivitas dasar dengan mengonfigurasi `jichi`: `eth0` memperoleh WAN dari NAT, sedangkan `eth1–eth4` menjadi gateway Network 1–4 sesuai tabel pemetaan. Aktifkan IPv4 forwarding dan Source NAT, lalu tetapkan IP statis host infrastruktur sesuai tabel spesifikasi. Buktikan konektivitas setiap host ke gateway dan akses eksternal dari host yang relevan.

### Langkah Pengerjaan dan Implementasi

#### 1. Pembangunan Topologi dan Wiring GNS3

Router **`jichi`** menggunakan image `ardhptr21/debinet:latest` dan dilengkapi dengan lima adapter jaringan yang memiliki fungsi sebagai berikut:

| Interface | Terhubung ke | Fungsi                          | Alamat IP      |
| --------- | ------------ | ------------------------------- | -------------- |
| `eth0`    | NAT1         | WAN / Internet                  | DHCP dari NAT  |
| `eth1`    | Switch1      | Gateway Network 1 / DMZ         | `10.96.1.1/24` |
| `eth2`    | Switch2      | Gateway Network 2 / Divisi A    | `10.96.2.1/24` |
| `eth3`    | Switch4      | Gateway Network 3 / Divisi B    | `10.96.3.1/24` |
| `eth4`    | Switch6      | Gateway Network 4 / Data Center | `10.96.4.1/24` |

Node infrastruktur dihubungkan ke segmen jaringan masing-masing:

- **Network 1 (Switch1):**

  - `utahime` — DNS Master.
  - `shoko` — DNS Slave.
  - `gojo` — Public Reverse Proxy.

- **Network 4 (Switch6 dan cabangnya):**

  - `tengen` — DHCP Server.
  - `yaga` — Database Master MariaDB.
  - `inumaki` — Database Slave MariaDB.
  - `geto` — Internal Load Balancer.
  - `maki` dan `panda` — Worker Frontend.
  - `nanami` dan `todo` — Worker API Utama.
  - `kirara` — Worker API Cadangan.

Seluruh node infrastruktur tersebut menggunakan image `ardhptr21/debinet:latest`, sedangkan node klien menggunakan `ardhptr21/alpinet:latest`.

#### 2. Konfigurasi Interface dan Routing Router `jichi`

Konfigurasi interface disimpan pada berkas `/etc/network/interfaces` di node `jichi`.

```text
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet dhcp
    post-up iptables -t nat -C POSTROUTING -o eth0 -j MASQUERADE 2>/dev/null || iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
    post-up sysctl -w net.ipv4.ip_forward=1

auto eth1
iface eth1 inet static
    address 10.96.1.1
    netmask 255.255.255.0

auto eth2
iface eth2 inet static
    address 10.96.2.1
    netmask 255.255.255.0

auto eth3
iface eth3 inet static
    address 10.96.3.1
    netmask 255.255.255.0

auto eth4
iface eth4 inet static
    address 10.96.4.1
    netmask 255.255.255.0
```

Konfigurasi tersebut memiliki fungsi sebagai berikut:

- `eth0` memperoleh alamat IP WAN secara otomatis melalui DHCP dari NAT1.
- `eth1–eth4` dikonfigurasi dengan alamat IP statis sebagai gateway setiap subnet internal.
- `iptables -t nat ... MASQUERADE` menerapkan Source NAT untuk trafik yang keluar melalui `eth0`.
- `sysctl -w net.ipv4.ip_forward=1` mengaktifkan IPv4 forwarding agar router dapat meneruskan paket antarjaringan.

Untuk mempertahankan IPv4 forwarding setelah container dimulai kembali, konfigurasi juga ditambahkan pada `/etc/sysctl.conf`:

```text
net.ipv4.ip_forward=1
```

Kemudian konfigurasi diterapkan menggunakan perintah:

```bash
sysctl -w net.ipv4.ip_forward=1
```

Aturan Source NAT diperiksa dengan:

```bash
iptables -t nat -L POSTROUTING -n -v
```

#### 3. Konfigurasi Alamat IP Statis Host Infrastruktur

Setiap host infrastruktur dikonfigurasi menggunakan berkas `/etc/network/interfaces`. Alamat IP ditetapkan secara statis dengan subnet mask `/24`, default gateway mengarah ke interface `jichi` yang sesuai, dan nameserver awal menggunakan `192.168.122.1` untuk persiapan instalasi paket.

**a. Network 1 — DMZ (`10.96.1.0/24`)**

| Host      | Peran                | Interface | IP Address     | Default Gateway |
| --------- | -------------------- | --------- | -------------- | --------------- |
| `utahime` | DNS Master           | `eth0`    | `10.96.1.2/24` | `10.96.1.1`     |
| `shoko`   | DNS Slave            | `eth0`    | `10.96.1.3/24` | `10.96.1.1`     |
| `gojo`    | Public Reverse Proxy | `eth0`    | `10.96.1.4/24` | `10.96.1.1`     |

Contoh konfigurasi `/etc/network/interfaces` pada `utahime`:

```text
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
    address 10.96.1.2
    netmask 255.255.255.0
    gateway 10.96.1.1
    dns-nameservers 192.168.122.1
```

Pada `shoko` dan `gojo`, format konfigurasi sama, tetapi nilai `address` disesuaikan dengan tabel pengalamatan.

**b. Network 4 — Data Center (`10.96.4.0/24`)**

| Host      | Peran                   | Interface | IP Address      | Default Gateway |
| --------- | ----------------------- | --------- | --------------- | --------------- |
| `tengen`  | DHCP Server             | `eth0`    | `10.96.4.2/24`  | `10.96.4.1`     |
| `yaga`    | Database Master MariaDB | `eth0`    | `10.96.4.3/24`  | `10.96.4.1`     |
| `inumaki` | Database Slave MariaDB  | `eth0`    | `10.96.4.4/24`  | `10.96.4.1`     |
| `geto`    | Internal Load Balancer  | `eth0`    | `10.96.4.5/24`  | `10.96.4.1`     |
| `maki`    | Worker Frontend 1       | `eth0`    | `10.96.4.6/24`  | `10.96.4.1`     |
| `panda`   | Worker Frontend 2       | `eth0`    | `10.96.4.7/24`  | `10.96.4.1`     |
| `nanami`  | Worker API Utama 1      | `eth0`    | `10.96.4.8/24`  | `10.96.4.1`     |
| `todo`    | Worker API Utama 2      | `eth0`    | `10.96.4.9/24`  | `10.96.4.1`     |
| `kirara`  | Worker API Cadangan     | `eth0`    | `10.96.4.10/24` | `10.96.4.1`     |

Contoh konfigurasi `/etc/network/interfaces` pada `tengen`:

```text
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
    address 10.96.4.2
    netmask 255.255.255.0
    gateway 10.96.4.1
    dns-nameservers 192.168.122.1
```

Konfigurasi host lainnya mengikuti format yang sama dengan menyesuaikan alamat IP berdasarkan tabel pengalamatan.

**Catatan DNS:** Direktif `dns-nameservers` memerlukan dukungan `resolvconf` atau integrasi resolver yang sesuai. Jika dukungan tersebut tidak tersedia, DNS dapat diatur melalui `/etc/resolv.conf` selama persiapan awal. Nameserver `192.168.122.1` digunakan hanya jika alamat tersebut dapat dijangkau dari node terkait.

#### 4. Penyimpanan Konfigurasi dan Script pada Direktori `/root`

Untuk memenuhi ketentuan praktikum, salinan konfigurasi dan script pengerjaan disimpan di direktori `/root`.

Contoh penyimpanan konfigurasi router:

```bash
mkdir -p /root/soal1
cp /etc/network/interfaces /root/soal1/interfaces-jichi
```

Pada setiap host infrastruktur, salinan konfigurasi disimpan menggunakan nama host masing-masing. Contohnya pada `utahime`:

```bash
mkdir -p /root/soal1
cp /etc/network/interfaces /root/soal1/interfaces-utahime
```

Penyimpanan salinan tersebut digunakan sebagai dokumentasi dan cadangan konfigurasi. Berkas yang diterapkan oleh sistem tetap berada di `/etc/network/interfaces`.

#### 5. Pengujian Konektivitas Jaringan

Setelah konfigurasi interface diterapkan, dilakukan pengujian untuk memastikan setiap interface aktif dan alamat IP sesuai dengan rancangan.

**a. Pemeriksaan interface dan routing pada `jichi`**

```bash
ip -br addr
ip route
sysctl net.ipv4.ip_forward
iptables -t nat -L POSTROUTING -n -v
```

Hal yang diperiksa:

- `eth0` memperoleh alamat WAN melalui DHCP.
- `eth1–eth4` memiliki alamat gateway yang sesuai.
- IPv4 forwarding bernilai `1`.
- Terdapat aturan `MASQUERADE` pada tabel NAT.
- Tabel routing memiliki rute untuk subnet internal dan default route menuju WAN.

**b. Pengujian gateway Network 1**

Pengujian dilakukan dari `utahime`, `shoko`, dan `gojo`:

```bash
ping -c 4 10.96.1.1
```

**c. Pengujian gateway Network 4**

Pengujian dilakukan dari `tengen`, `yaga`, `inumaki`, `geto`, `maki`, `panda`, `nanami`, `todo`, dan `kirara`:

```bash
ping -c 4 10.96.4.1
```

**d. Pengujian akses eksternal**

Dari `jichi`, periksa konektivitas WAN:

```bash
ping -c 4 8.8.8.8
```

Kemudian, dari host internal yang relevan, misalnya `utahime` dan `tengen`, lakukan pengujian:

```bash
ping -c 4 8.8.8.8
curl -I --connect-timeout 5 https://example.com
```

Jika koneksi eksternal gagal, pemeriksaan dilanjutkan pada default route, konfigurasi NAT, kebijakan forwarding, dan DNS. Kegagalan ping saja tidak selalu menunjukkan kerusakan jaringan karena ICMP dapat dibatasi oleh jaringan tujuan.

### Bukti dan Hasil

> **Catatan:** Tambahkan tautan screenshot sesuai berkas bukti yang benar-benar diambil dari lingkungan praktikum. Bagian berikut merupakan template dokumentasi yang dapat dilengkapi setelah pengujian.

1. **Topologi Jaringan Lengkap di GNS3**

   - Bukti: `bukti/soal-1/topologi.png`
   - Keterangan: Topologi Jujutsu High dirancang sesuai skema soal, dengan `jichi` sebagai router utama yang terhubung ke NAT1 dan empat segmen jaringan internal.

2. **Verifikasi Alamat IP, Tabel Routing Router, IPv4 Forwarding dan Source NAT  `jichi`**

   - Bukti: `bukti/soal-1/ip-route-jichi.png`
   - Keterangan: Output `ip -br addr` dan `ip route` digunakan untuk membuktikan bahwa interface WAN memperoleh alamat IP melalui DHCP, sedangkan interface `eth1–eth4` menggunakan alamat gateway statis sesuai Network 1–4. 
   Output `sysctl net.ipv4.ip_forward` menunjukkan status forwarding, sedangkan output `iptables -t nat -L POSTROUTING -n -v` digunakan untuk memeriksa aturan Source NAT.

3. **Uji Konektivitas Host Network 1**

   - Bukti: `bukti/soal-1/ping-utahime.png`
   - Keterangan: Hasil pengujian ping dari `utahime` menuju gateway `10.96.1.1` digunakan untuk memverifikasi konektivitas dasar Network 1.

4. **Uji Konektivitas Host Network 4**

   - Bukti: `bukti/soal-1/ping-tengen.png`
   - Keterangan: Hasil pengujian ping dari `tengen` menuju gateway `10.96.4.1` digunakan untuk memverifikasi konektivitas dasar Network 4.

## Soal 2 — Konfigurasi DHCP Server `tengen`

> **Dikerjakan Oleh:** Zidny Ilman Nafi'an

### Soal

> Tengen, penjaga pusat distribusi di Data Center, harus mengambil alih pengalokasian alamat bagi wilayah yang membutuhkan konfigurasi dinamis. Konfigurasikan `isc-dhcp-server` pada `10.96.4.10`. Deklarasikan Network 4 (`10.96.4.0/24`) sebagai subnet lokal tanpa pool dinamis, lalu siapkan scope DHCP untuk Network 2 (`10.96.2.0/24`) dan Network 3 (`10.96.3.0/24`). Validasi konfigurasi dan pastikan service aktif tanpa error "No subnet declaration".

### Langkah Pengerjaan dan Implementasi

#### 1. Konfigurasi IP Address `tengen`

Node `tengen` berfungsi sebagai DHCP Server dan berada pada Network 4 / Data Center. Alamat IP ditetapkan secara statis menggunakan `/etc/network/interfaces`.

| Parameter | Nilai |
|---|---|
| Host | `tengen` |
| Interface | `eth0` |
| IP Address | `10.96.4.10/24` |
| Default Gateway | `10.96.4.1` |
| Image | `ardhptr21/debinet:latest` |

Konfigurasi interface:

```text
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
    address 10.96.4.10
    netmask 255.255.255.0
    gateway 10.96.4.1
```

Setelah konfigurasi diterapkan, dilakukan pemeriksaan menggunakan `ip -br addr`, `ip route`, dan `ping -c 4 10.96.4.1` untuk memastikan interface aktif dan gateway dapat dijangkau.

#### 2. Script instalasi dan konfigurasi DHCP Server
File /root/soal2-dhcp.sh pada node tengen. Script ini memasang paket DHCP Server, menentukan interface yang digunakan, menulis deklarasi subnet, lalu memvalidasi konfigurasi.

```bash
```bash
#!/bin/bash
# Praktikum Jaringan Komputer - Modul 3
# Soal 2: Konfigurasi DHCP Server Tengen
# Kelompok K-65

set -e

# Pastikan script dijalankan sebagai root
if [ "$(id -u)" -ne 0 ]; then
    echo "Jalankan script sebagai root."
    exit 1
fi

echo "=== Persiapan DNS ==="
echo "nameserver 192.168.122.1" > /etc/resolv.conf

echo "=== Instalasi isc-dhcp-server ==="
apt-get update
DEBIAN_FRONTEND=noninteractive apt-get install -y isc-dhcp-server

echo "=== Konfigurasi interface DHCP ==="
cat > /etc/default/isc-dhcp-server <<'EOF'
INTERFACESv4="eth0"
INTERFACESv6=""
EOF

echo "=== Konfigurasi DHCP ==="
mkdir -p /root/soal2

# Cadangkan konfigurasi sebelumnya jika tersedia
if [ -f /etc/dhcp/dhcpd.conf ]; then
    cp /etc/dhcp/dhcpd.conf /root/soal2/dhcpd.conf.backup
fi

cat > /etc/dhcp/dhcpd.conf <<'EOF'
# DHCP Server Tengen - Kelompok K-65
authoritative;
default-lease-time 600;
max-lease-time 7200;

# Network 4: subnet lokal DHCP Server.
# Tidak menyediakan pool dinamis.
subnet 10.96.4.0 netmask 255.255.255.0 {
}

# Network 2: Divisi A
subnet 10.96.2.0 netmask 255.255.255.0 {
    range 10.96.2.100 10.96.2.200;
    option subnet-mask 255.255.255.0;
    option routers 10.96.2.1;
    option domain-name-servers 10.96.1.2;
}

# Network 3: Divisi B
subnet 10.96.3.0 netmask 255.255.255.0 {
    range 10.96.3.100 10.96.3.200;
    option subnet-mask 255.255.255.0;
    option routers 10.96.3.1;
    option domain-name-servers 10.96.1.2;
}
EOF

echo "=== Validasi konfigurasi DHCP ==="
dhcpd -t -4 -cf /etc/dhcp/dhcpd.conf

echo "=== Menyimpan salinan konfigurasi ==="
cp /etc/dhcp/dhcpd.conf /root/soal2/dhcpd.conf
cp /etc/default/isc-dhcp-server /root/soal2/isc-dhcp-server

echo "=== Mengaktifkan DHCP Server ==="
if command -v systemctl >/dev/null 2>&1; then
    systemctl enable isc-dhcp-server
    systemctl restart isc-dhcp-server
    systemctl --no-pager --full status isc-dhcp-server
else
    service isc-dhcp-server restart
    service isc-dhcp-server status
fi

echo "=== Konfigurasi DHCP selesai ==="
```

Jalankan script:
```bash
chmod +x /root/soal2-dhcp.sh
bash /root/soal2-dhcp.sh
```

#### 2. Instalasi `isc-dhcp-server`

Paket `isc-dhcp-server` dipasang pada node `tengen`. Sebelum instalasi, nameserver awal menggunakan `192.168.122.1` sesuai ketentuan persiapan praktikum.

```bash
echo "nameserver 192.168.122.1" > /etc/resolv.conf
apt-get update
apt-get install -y isc-dhcp-server
```

#### 3. Konfigurasi Interface DHCP Server

Berkas `/etc/default/isc-dhcp-server` digunakan untuk menentukan interface yang melayani permintaan DHCP.

```text
INTERFACESv4="eth0"
INTERFACESv6=""
```

Dengan konfigurasi ini, layanan DHCP IPv4 dijalankan melalui interface `eth0` milik `tengen`.

#### 4. Deklarasi Subnet dan Scope DHCP

Konfigurasi utama disimpan dalam `/etc/dhcp/dhcpd.conf`.

```text
authoritative;
default-lease-time 600;
max-lease-time 7200;

# Network 4: subnet lokal tanpa pool dinamis
subnet 10.96.4.0 netmask 255.255.255.0 {
}

# Network 2: Divisi A
subnet 10.96.2.0 netmask 255.255.255.0 {
    range 10.96.2.100 10.96.2.200;
    option subnet-mask 255.255.255.0;
    option routers 10.96.2.1;
    option domain-name-servers 10.96.1.2;
}

# Network 3: Divisi B
subnet 10.96.3.0 netmask 255.255.255.0 {
    range 10.96.3.100 10.96.3.200;
    option subnet-mask 255.255.255.0;
    option routers 10.96.3.1;
    option domain-name-servers 10.96.1.2;
}
```

Penjelasan konfigurasi:

- Network 4 dideklarasikan tanpa `range` karena digunakan sebagai subnet lokal DHCP Server, bukan pool dinamis.
- Network 2 memiliki scope awal `10.96.2.100–10.96.2.200`.
- Network 3 memiliki scope awal `10.96.3.100–10.96.3.200`.
- `option routers` menentukan default gateway klien pada masing-masing subnet.
- `option domain-name-servers` menunjuk ke DNS Master `utahime` (`10.96.1.2`), sesuai rancangan jaringan.
- Rentang DHCP perlu disesuaikan apabila terdapat ketentuan pool khusus pada soal berikutnya.

#### 5. Validasi Konfigurasi dan Status Layanan

Validasi sintaks dilakukan menggunakan:

```bash
dhcpd -t -4 -cf /etc/dhcp/dhcpd.conf
```

Setelah konfigurasi dinyatakan valid, layanan diaktifkan dan dimulai ulang:

```bash
systemctl enable isc-dhcp-server
systemctl restart isc-dhcp-server
systemctl status isc-dhcp-server
```

Jika `systemctl` tidak tersedia, pemeriksaan dilakukan melalui layanan `service` sesuai dukungan image container.

Pemeriksaan log dilakukan untuk memastikan tidak terdapat error `No subnet declaration for eth0`.

```bash
grep -iE 'error|subnet|fail' /var/log/syslog | tail -n 30
```

Konfigurasi yang valid harus mendeklarasikan subnet `10.96.4.0/24`, karena interface `eth0` pada `tengen` menggunakan alamat `10.96.4.10/24`.

#### 6. Penyimpanan Konfigurasi pada Direktori `/root`

Salinan konfigurasi disimpan di `/root/soal2` untuk dokumentasi dan kebutuhan praktikum.

```bash
mkdir -p /root/soal2
cp /etc/dhcp/dhcpd.conf /root/soal2/dhcpd.conf
cp /etc/default/isc-dhcp-server /root/soal2/isc-dhcp-server
```

Berkas aktif tetap berada di `/etc/dhcp/dhcpd.conf` dan `/etc/default/isc-dhcp-server`.

### Bukti dan Hasil

1. **Konfigurasi DHCP Server**
   - Bukti: `Screenshot/soal-2/dhcpd-conf.png`
   - Keterangan: Menampilkan deklarasi Network 4 tanpa pool dinamis serta scope DHCP Network 2 dan Network 3.

2. **Validasi Sintaks & Status Aktif Layanan ISC DHCP Server**
   - Bukti: `Screenshot/soal-2/dhcp-validation.png`
   - Keterangan: Menampilkan hasil `dhcpd -t -4 -cf /etc/dhcp/dhcpd.conf` yang tidak menghasilkan error sintaks dan Menampilkan status layanan `isc-dhcp-server` setelah konfigurasi diterapkan.

3. **Pemeriksaan Log**
   - Bukti: `Screenshot/soal-2/dhcp-log.png`
   - Keterangan: Menampilkan log layanan untuk memastikan tidak ada error `No subnet declaration for eth0`.

## Soal 3 — Konfigurasi DHCP Relay pada Router `jichi`

> **Dikerjakan Oleh:** Zidny Ilman Nafi'an

### 1. Tujuan

Mengonfigurasi router `jichi` sebagai DHCP Relay agar permintaan alamat IP dari klien Network 2 (Divisi A) dan Network 3 (Divisi B) dapat diteruskan ke DHCP Server `tengen` dengan alamat IP `10.96.4.10`.

Pengujian dilakukan pada dua klien:
- **Divisi A:** `itadori`, IP target `10.96.2.10`
- **Divisi B:** `ino`, IP target `10.96.3.10`

### 2. Topologi dan Alur DHCP Relay

DHCP Relay diperlukan karena permintaan DHCP dari klien menggunakan broadcast yang tidak diteruskan secara langsung oleh router ke subnet lain.

Alur komunikasi:

```text
itadori (Divisi A)
10.96.2.10
     |
   Switch
     |
jichi eth2
10.96.2.1
     |
     | DHCP Relay
     |
jichi eth4 ---- Network 4
     |              |
     |          tengen
     |        10.96.4.10
     |
jichi eth3
10.96.3.1
     |
   Switch
     |
ino (Divisi B)
10.96.3.10
```

### 3. Konfigurasi DHCP Relay pada `jichi`

- Buat script konfigurasi di jichi
Buat script /root/soal3-dhcp-relay.sh agar konfigurasinya terdokumentasi dan bisa dijalankan ulang.

```bash
cat > /root/soal3-dhcp-relay.sh <<'SCRIPT'
#!/bin/bash
# Praktikum Jaringan Komputer - Modul 3
# Soal 3: Konfigurasi DHCP Relay pada jichi
# Kelompok K-65

set -Ee
trap 'rc=$?; echo "ERROR pada baris ${LINENO}, exit code ${rc}"; exit "$rc"' ERR

if [ "$(id -u)" -ne 0 ]; then
    echo "Jalankan script sebagai root."
    exit 1
fi

echo "======================================"
echo " SOAL 3 - DHCP RELAY JICHI"
echo "======================================"

echo
echo "[1/6] Memeriksa interface router..."
ip -br addr

for iface in eth2 eth3 eth4; do
    if ! ip link show "$iface" >/dev/null 2>&1; then
        echo "ERROR: Interface $iface tidak ditemukan."
        exit 1
    fi
done

echo
echo "[2/6] Memeriksa konektivitas ke Tengen..."
if ! ip -4 addr show eth4 | grep -q '10\.96\.4\.'; then
    echo "PERINGATAN: eth4 belum terlihat memiliki IP Network 4."
    echo "Pastikan eth4 terhubung ke Switch6/Network 4."
fi

echo
echo "[3/6] Mengaktifkan IPv4 forwarding..."
sysctl -w net.ipv4.ip_forward=1

if ! grep -qE '^[[:space:]]*net\.ipv4\.ip_forward[[:space:]]*=[[:space:]]*1' /etc/sysctl.conf; then
    echo 'net.ipv4.ip_forward=1' >> /etc/sysctl.conf
fi

echo
echo "[4/6] Memasang isc-dhcp-relay..."
if ! command -v dhcrelay >/dev/null 2>&1; then
    apt-get update
    DEBIAN_FRONTEND=noninteractive apt-get install -y isc-dhcp-relay
fi

echo
echo "[5/6] Mengatur DHCP Relay..."
mkdir -p /root/soal3

if [ -f /etc/default/isc-dhcp-relay ]; then
    cp /etc/default/isc-dhcp-relay \
       "/root/soal3/isc-dhcp-relay.backup-$(date +%Y%m%d-%H%M%S)"
fi

cat > /etc/default/isc-dhcp-relay <<'EOF'
SERVERS="10.96.4.10"
INTERFACES="eth2 eth3"
OPTIONS=""
EOF

cat /etc/default/isc-dhcp-relay

echo
echo "[6/6] Memulai layanan DHCP Relay..."
if command -v systemctl >/dev/null 2>&1; then
    systemctl enable isc-dhcp-relay
    systemctl restart isc-dhcp-relay
else
    service isc-dhcp-relay restart
fi

cp /etc/default/isc-dhcp-relay /root/soal3/isc-dhcp-relay

echo
echo "=== Status DHCP Relay ==="
service isc-dhcp-relay status

echo
echo "=== Proses DHCP Relay ==="
if pgrep -a dhcrelay; then
    echo "Proses dhcrelay ditemukan."
else
    echo "ERROR: Proses dhcrelay tidak ditemukan."
    exit 1
fi

echo
echo "=== Konfigurasi DHCP Relay selesai ==="
SCRIPT

chmod +x /root/soal3-dhcp-relay.sh
```

Jalankan script dan simpan log pengujiannya:
```bash
bash -x /root/soal3-dhcp-relay.sh 2>&1 | tee /root/soal3-test.log
```

#### 3.1. Instalasi DHCP Relay

DHCP Relay dijalankan pada router `jichi` menggunakan paket `isc-dhcp-relay`.

```bash
apt-get update
apt-get install -y isc-dhcp-relay
```

#### 3.2. Konfigurasi interface relay

File konfigurasi yang digunakan adalah `/etc/default/isc-dhcp-relay`.

```bash
cat > /etc/default/isc-dhcp-relay <<'EOF'
SERVERS="10.96.4.10"
INTERFACES="eth2 eth3"
OPTIONS=""
EOF
```

Keterangan:
- `SERVERS="10.96.4.10"` menentukan alamat DHCP Server `tengen`.
- `INTERFACES="eth2 eth3"` menentukan interface yang menerima permintaan DHCP dari Network 2 dan Network 3.
- `OPTIONS=""` menggunakan opsi default dari paket DHCP Relay.

Interface `eth4` menjadi jalur menuju Network 4 melalui routing yang tersedia pada router.

#### 3.3. Mengaktifkan IPv4 forwarding

```bash
sysctl -w net.ipv4.ip_forward=1
```

Agar konfigurasi forwarding tetap aktif setelah reboot, tambahkan konfigurasi berikut ke `/etc/sysctl.conf` jika belum tersedia:

```text
net.ipv4.ip_forward=1
```

#### 3.4. Menjalankan layanan DHCP Relay

```bash
service isc-dhcp-relay restart
service isc-dhcp-relay status
ps aux | grep '[d]hcrelay'
```

Konfigurasi dianggap berhasil pada tahap ini apabila proses `dhcrelay` berjalan dan layanan tidak melaporkan kegagalan.

### 4. Konfigurasi DHCP Server `tengen`

DHCP Server berada di `tengen` dengan alamat IP `10.96.4.10/24`.

Konfigurasi subnet pada `/etc/dhcp/dhcpd.conf` harus mencakup Network 4, Network 2, dan Network 3.

```conf
authoritative;
ddns-update-style none;

default-lease-time 600;
max-lease-time 7200;

option domain-name-servers 10.96.1.2;

# Network 4: subnet lokal DHCP Server, tanpa pool dinamis
subnet 10.96.4.0 netmask 255.255.255.0 {
}

# Network 2: Divisi A
subnet 10.96.2.0 netmask 255.255.255.0 {
    range 10.96.2.100 10.96.2.200;
    option subnet-mask 255.255.255.0;
    option routers 10.96.2.1;
}

# Network 3: Divisi B
subnet 10.96.3.0 netmask 255.255.255.0 {
    range 10.96.3.100 10.96.3.200;
    option subnet-mask 255.255.255.0;
    option routers 10.96.3.1;
}
```

Validasi dan pemeriksaan layanan:

```bash
dhcpd -t -4 -cf /etc/dhcp/dhcpd.conf
service isc-dhcp-server status
```

**Catatan:** Konfigurasi di atas menggunakan pool `.100–.200`. Jika `itadori` harus memperoleh `.10` dari DHCP, diperlukan konfigurasi reservasi berdasarkan MAC address atau penyesuaian pool sesuai ketentuan tugas. IP `.10` tidak boleh digunakan bersamaan oleh dua perangkat.

### 5. Pengujian Klien Divisi A — `itadori`

Alamat IP target: `10.96.2.10/24`

#### 5.1. Memeriksa interface jaringan

```bash
ip -br addr
ip route
```

Pastikan `eth0` terhubung ke Network 2.

#### 5.2. Meminta alamat melalui DHCP

Jika klien menggunakan Alpine Linux dengan `udhcpc`:

```bash
ip link set eth0 up
udhcpc -i eth0 -v
```

#### 5.3. Memeriksa hasil

```bash
ip -br addr
ip route
```

Hasil yang diharapkan:
- Alamat IPv4: `10.96.2.10/24` jika reservasi atau konfigurasi pool sudah disesuaikan.
- Gateway: `10.96.2.1`.
- DHCP Server: `10.96.4.10` sebagai server tujuan yang melayani permintaan melalui relay.

Simpan output aktual sebagai bukti pengujian.

### 6. Pengujian Klien Divisi B — `ino`

Alamat IP target: `10.96.3.10/24`

#### 6.1. Memeriksa interface jaringan

```bash
ip -br addr
ip route
```

Pastikan `eth0` terhubung ke Network 3.

#### 6.2. Meminta alamat melalui DHCP

```bash
ip link set eth0 up
udhcpc -i eth0 -v
```

#### 6.3. Memeriksa hasil

```bash
ip -br addr
ip route
```

Hasil yang diharapkan:
- Alamat IPv4: `10.96.3.10/24` jika reservasi atau konfigurasi pool sudah disesuaikan.
- Gateway: `10.96.3.1`.
- DHCP Server: `10.96.4.10` sebagai server tujuan yang melayani permintaan melalui relay.

Simpan output aktual sebagai bukti pengujian.

### 7. Verifikasi dan Analisis

Pemeriksaan dilakukan dari sisi router, server, dan klien.

Pada `jichi`:

```bash
bash -x /root/soal3-dhcp-relay.sh 2>&1 | tee /root/soal3-test.log
service isc-dhcp-relay status
ps aux | grep '[d]hcrelay'
```

Pada `tengen`:

```bash
service isc-dhcp-server status
ps aux | grep '[d]hcpd'
```

Pada klien `itadori` dan `ino`:

```bash
ip -br addr
ip route
```

Keberhasilan DHCP Relay dibuktikan apabila kedua klien memperoleh lease yang sesuai dari DHCP Server melalui router. Jika klien masih mendapatkan alamat lain, tidak memperoleh lease, atau layanan gagal aktif, konfigurasi perlu diperiksa kembali sebelum hasil dinyatakan berhasil.

### 8. Dokumentasi Screenshot

Simpan bukti pengujian dalam folder `bukti/soal-3/` dengan nama file berikut:

1. `jichi-relay-config.png` — konfigurasi `/etc/default/isc-dhcp-relay` dan status layanan DHCP Relay.
2. `itadori-dhcp-lease.png` — hasil pengujian DHCP klien Divisi A.
3. `ino-dhcp-lease.png` — hasil pengujian DHCP klien Divisi B.