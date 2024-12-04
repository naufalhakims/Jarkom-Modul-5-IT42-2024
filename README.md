# Jarkom-Modul-5-IT42-2024

#### **1. Mengonfigurasi Routing Agar Jaringan Eridu Terhubung ke Internet**
Untuk menghubungkan jaringan **Eridu** ke internet tanpa menggunakan MASQUERADE, konfigurasi routing dilakukan dengan menggunakan **iptables** untuk NAT (Network Address Translation) dan routing statis.

**Langkah-langkah Konfigurasi:**
1. **Menambahkan IP routing statis di router**:
   Routing statis ditambahkan pada **router** agar paket yang menuju internet dapat diteruskan ke gateway eksternal.
   ```bash
   sudo ip route add default via 192.168.1.1  # Contoh rute default
   ```
2. **Mengonfigurasi iptables** untuk NAT tanpa menggunakan MASQUERADE:
   Konfigurasi **SNAT** digunakan untuk mengganti alamat IP sumber paket yang keluar, namun tetap menghindari penggunaan MASQUERADE.
   ```bash
   sudo iptables -t nat -A POSTROUTING -o eth0 -j SNAT --to-source 192.168.1.1
   ```
   Penjelasan:
   - `-o eth0` menunjukkan bahwa paket yang keluar dari interface **eth0** akan dikenakan SNAT dengan alamat IP sumber **192.168.1.1**.

#### **2. Membatasi Akses ke Fairy Agar Tidak Ada Perangkat Lain yang Bisa Melakukan Ping**
Untuk memastikan bahwa hanya **Fairy** yang dapat diakses melalui ping, sementara perangkat lain tidak dapat melakukannya, konfigurasi **iptables** digunakan untuk memblokir **ICMP** (ping) dari perangkat selain Fairy.

**Langkah-langkah Konfigurasi:**
1. **Memblokir ping dari perangkat lain**:
   Iptables dikonfigurasi untuk memblokir **ICMP** dari perangkat selain **Fairy**.
   ```bash
   sudo iptables -A INPUT -p icmp -s ! 192.168.1.100 -j DROP  # IP Fairy
   ```

2. **Memastikan Fairy tetap dapat mengakses seluruh perangkat**:
   Aturan ini memastikan bahwa hanya perangkat selain **Fairy** yang diblokir untuk melakukan ping, sedangkan Fairy tetap dapat mengakses seluruh perangkat di jaringan.

#### **3. Membatasi Akses Hanya untuk Fairy yang Dapat Mengakses HDD**
Akses ke **HDD** hanya diizinkan untuk **Fairy**. Dengan menggunakan **nc (netcat)**, dapat dipastikan bahwa hanya Fairy yang bisa mengakses HDD.

**Langkah-langkah Konfigurasi:**
1. **Mengonfigurasi iptables untuk membatasi akses**:
   Iptables dikonfigurasi untuk menerima koneksi hanya dari **Fairy** dan menolak koneksi lainnya.
   ```bash
   sudo iptables -A INPUT -p tcp -s 192.168.1.100 --dport 80 -j ACCEPT  # IP Fairy
   sudo iptables -A INPUT -p tcp --dport 80 -j DROP  # Blokir akses lainnya
   ```

2. **Verifikasi dengan menggunakan nc**:
   **Netcat** digunakan untuk memastikan bahwa hanya **Fairy** yang bisa mengakses **HDD**.
   ```bash
   nc -v 192.168.2.100 80  # Dari Fairy
   nc -v 192.168.2.100 80  # Dari perangkat lain (seharusnya gagal)
   ```

#### **4. Mengonfigurasi Akses ke Hollow Zero Berdasarkan Hari dan Faksi**
HollowZero hanya boleh diakses pada hari Senin hingga Jumat oleh faksi tertentu, yaitu **SoC** dan **PubSec**. Untuk mengontrol akses ini, iptables dikonfigurasi untuk membatasi akses pada hari Sabtu.

**Langkah-langkah Konfigurasi:**
1. **Menambahkan kontrol akses berdasarkan waktu dan faksi**:
   Akses ke HollowZero hanya diizinkan untuk faksi **SoC** (Burnice & Caesar) dan **PubSec** (Jane & Policeboo) pada hari kerja (Senin-Jumat).
   ```bash
   sudo iptables -A INPUT -p tcp --dport 80 -m time --timestart 08:00 --timestop 21:00 --weekdays Mon,Tue,Wed,Thu,Fri -j ACCEPT
   sudo iptables -A INPUT -p tcp --dport 80 -j DROP  # Blokir akses selain hari Senin hingga Jumat
   ```

2. **Verifikasi dengan curl**:
   Akses menggunakan curl pada hari Sabtu untuk memastikan bahwa aturan waktu berfungsi dengan benar:
   ```bash
   curl http://192.168.3.100  # Akses ke HollowZero
   ```

#### **5. Membatasi Akses ke Server HIA Berdasarkan Waktu dan Faksi**
Akses ke **HIA** dibatasi berdasarkan faksi dan jam akses yang telah ditentukan.

**Langkah-langkah Konfigurasi:**
1. **Menambahkan aturan iptables untuk mengontrol akses berdasarkan waktu**:
   - Akses hanya diizinkan untuk **Ellen dan Lycaon** pada jam 08:00-21:00, dan untuk **Jane dan Policeboo** pada jam 03:00-23:00.
   ```bash
   sudo iptables -A INPUT -p tcp -s 192.168.4.10 --dport 80 -m time --timestart 08:00 --timestop 21:00 --weekdays Mon,Tue,Wed,Thu,Fri -j ACCEPT
   sudo iptables -A INPUT -p tcp -s 192.168.4.20 --dport 80 -m time --timestart 03:00 --timestop 23:00 --weekdays Mon,Tue,Wed,Thu,Fri -j ACCEPT
   sudo iptables -A INPUT -p tcp --dport 80 -j DROP  # Blokir akses di luar jam yang ditentukan
   ```

2. **Verifikasi dengan curl**:
   **Curl** digunakan untuk memastikan akses hanya diizinkan pada jam yang telah ditentukan.
   ```bash
   curl http://192.168.4.50  # Akses ke HIA
   ```

#### **6. Memblokir Aktivitas Port Scan yang Mencurigakan Menggunakan Nmap**
**PubSec** diminta untuk memperketat keamanan dengan memblokir aktivitas **port scan** yang melebihi 25 port dalam 10 detik menggunakan **iptables**.

**Langkah-langkah Konfigurasi:**
1. **Membatasi port scan dengan iptables**:
   Aturan iptables ditambahkan untuk memblokir aktivitas **port scan** yang terdeteksi melebihi 25 port dalam waktu 10 detik.
   ```bash
   sudo iptables -A INPUT -p tcp --dport 1:100 -m recent --name portscan --rcheck --seconds 10 --hitcount 25 -j REJECT --reject-with tcp-reset
   sudo iptables -A INPUT -p tcp --dport 1:100 -m recent --name portscan --set
   ```

2. **Memblokir ping dan nc**:
   Akses dari IP yang terdeteksi melakukan port scan diblokir, termasuk **ping** dan **nc**.
   ```bash
   sudo iptables -A INPUT -p icmp -s <IP_penyerang> -j DROP
   sudo iptables -A INPUT -p tcp -s <IP_penyerang> --dport 80 -j DROP
   ```

3. **Mencatat log untuk analisis**:
   Iptables dikonfigurasi untuk mencatat semua akses yang diterima dari sumber tertentu.
   ```bash
   sudo iptables -A INPUT -p tcp --dport 80 -m limit --limit 5/minute --log-prefix "HIA Access Attempt: " --log-level 4
   ```

#### **7. Membatasi Akses ke Hollow Zero Berdasarkan Dua Koneksi Aktif**
Akses ke **HollowZero** dibatasi hanya untuk dua koneksi aktif yang berasal dari dua IP berbeda pada waktu yang bersamaan.

**Langkah-langkah Konfigurasi:**
1. **Menggunakan iptables untuk membatasi koneksi**:
   Konfigurasi iptables memastikan hanya dua koneksi aktif yang dapat mengakses **HollowZero** pada waktu yang bersamaan.
   ```bash
   sudo iptables -A INPUT -p tcp --dport 80 -m state --state NEW -m recent --set --name hollow_access
   sudo iptables -A INPUT -p tcp --dport 80 -m recent --update --name hollow_access --rcheck --seconds 5 --hitcount 2 -j ACCEPT
   ```

2. **Verifikasi dengan curl**:
   Verifikasi koneksi menggunakan **curl** untuk memastikan bahwa aturan koneksi dua IP berbeda berfungsi dengan benar.
   ```bash
   curl http://192.168.30.30
   ```

#### **8. Menggunakan Netcat (nc) untuk Memastikan Alur Pengalihan Paket dari Fairy ke HollowZero**
Setiap paket yang dikirim oleh **Fairy** ke **Burnice** dialihkan ke **HollowZero**.

**Langkah-langkah Konfigurasi:**
1. **Verifikasi pengalihan dengan nc**:
  

 **Netcat** digunakan untuk memastikan bahwa paket dari **Fairy** dialihkan ke **HollowZero**.
   ```bash
   nc -v 192.168.5.100 80  # Fairy mengirim paket ke Burnice yang dialihkan ke HollowZero
   ```

