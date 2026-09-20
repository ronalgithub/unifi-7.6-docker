# **Instalasi UniFi Controller 7.6 di Docker**



*User prompt: berikan saya step by step dalam menginstall unifi controller 7.6 yang berjalan di docker dengan operating sistem ubuntu 24.04. \- simpan semua authentication menggunakan .env \- gunakan command sesuai dengan official guide dari mulai install docker sampai running unifi*

Response: Proses instalasi UniFi Network Application (versi 7.6) menggunakan Docker di Ubuntu 24.04 memerlukan setup database eksternal (MongoDB), karena Ubiquiti telah menghapus database internal bawaan pada versi-versi teranyar.

Berikut adalah panduan lengkap langkah demi langkah yang disesuaikan dengan dokumentasi resmi Docker dan best practice penataan variabel lingkungan (.env).

&nbsp;

&nbsp;

**1. Update Sistem & Install Dependensi Awal**

Langkah Resmi Docker

&nbsp;

Langkah pertama adalah memastikan indeks paket lokal Anda diperbarui dan dependensi dasar untuk pengunduhan repositori via HTTPS sudah terpasang:

`sudo apt-get update`  
`sudo apt-get install -y ca-certificates curl`

&nbsp;

**2. Tambahkan GPG Key Resmi Docker**

Langkah Resmi Docker

&nbsp;

Unduh dan daftarkan kunci GPG resmi milik Docker agar sistem Ubuntu dapat memverifikasi keaslian paket yang akan diunduh:

`sudo install -m 0755 -d /etc/apt/keyrings`  
`sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc`  
`sudo chmod a+r /etc/apt/keyrings/docker.asc`

&nbsp;

**3. Daftarkan Repositori Docker ke Sistem**

Langkah Resmi Docker

&nbsp;

Gunakan perintah berikut untuk menambahkan repositori stabil Docker ke dalam daftar APT sources Ubuntu 24.04 Anda:

`echo \`  
&nbsp;&nbsp;`"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \`  
&nbsp;&nbsp;`$(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \`  
&nbsp;&nbsp;`sudo tee /etc/apt/sources.list.d/docker.list > /dev/null`

&nbsp;

**4. Install Docker Engine & Docker Compose**

Langkah Resmi Docker

&nbsp;

Lakukan pembaruan indeks paket sekali lagi, lalu install Docker Engine beserta plugin Docker Compose versi terbaru:

`sudo apt-get update`  
`sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin`

*(Opsional)* Agar Anda dapat menjalankan perintah docker tanpa perlu mengetik sudo, tambahkan user Anda ke grup docker:

`sudo usermod -aG docker $USER`

*(Catatan: Anda perlu log out dan log in kembali ke terminal agar efek perubahan grup ini aktif).*

&nbsp;

**5. Siapkan Direktori Kerja UniFi**

Struktur Folder Workspace

&nbsp;

Buat folder khusus untuk menampung seluruh berkas konfigurasi UniFi dan database agar manajemen data Anda lebih rapi:

`mkdir -p ~/unifi-infra`  
`cd ~/unifi-infra`

&nbsp;

**6. Buat File Autentikasi (.env)**

Keamanan Variabel Lingkungan

&nbsp;

Sesuai permintaan Anda, seluruh kredensial database disimpan secara terisolasi di berkas .env. Buat file baru dengan perintah nano .env lalu isi dengan konfigurasi berikut (silakan ubah password sesuai preferensi keamanan Anda):

`# Autentikasi Utama Root MongoDB (Hanya digunakan saat pertama kali init)`  
`MONGO_INITDB_ROOT_USERNAME=rootadmin`  
`MONGO_INITDB_ROOT_PASSWORD=PasswordRootRahasia123!`

`# Kredensial Akses Aplikasi UniFi ke MongoDB`  
`MONGO_USER=unifi`  
`MONGO_PASS=PasswordUniFiRahasia456!`  
`MONGO_DBNAME=unifi`  
`MONGO_HOST=unifi-db`  
`MONGO_PORT=27017`  
`MONGO_AUTHSOURCE=admin`

`# Parameter Lingkungan Kontainer`  
`TZ=Asia/Jakarta`  
`PUID=1000`  
`PGID=1000`

&nbsp;

**7. Buat Script Inisialisasi Database (init-mongo.js)**

Konfigurasi Otomatis MongoDB

&nbsp;

Karena citra (*image*) resmi MongoDB memerlukan pembuatan database user non-root secara manual untuk kebutuhan isolasi multi-database UniFi, buatlah file init-mongo.js (nano init-mongo.js) yang akan membaca variabel dari berkas .env secara otomatis:

`db.getSiblingDB(process.env.MONGO_DBNAME).createUser({`  
&nbsp;&nbsp;`user: process.env.MONGO_USER,`  
&nbsp;&nbsp;`pwd: process.env.MONGO_PASS,`  
&nbsp;&nbsp;`roles: [{ role: "readWrite", db: process.env.MONGO_DBNAME }]`  
`});`  
`db.getSiblingDB(process.env.MONGO_DBNAME + '_stat').createUser({`  
&nbsp;&nbsp;`user: process.env.MONGO_USER,`  
&nbsp;&nbsp;`pwd: process.env.MONGO_PASS,`  
&nbsp;&nbsp;`roles: [{ role: "readWrite", db: process.env.MONGO_DBNAME + '_stat' }]`  
`});`

&nbsp;

**8. Susun Berkas docker-compose.yml**

Orkestrasi Aplikasi

&nbsp;

Sekarang, buat file orkestrasinya (nano docker-compose.yml). Di sini kita menggunakan image dari *LinuxServer* dengan mengunci tag spesifik ke versi **7.6.126** untuk menjamin versi 7.6 terpasang dengan stabil:

`services:`  
&nbsp;&nbsp;`unifi-db:`  
&nbsp;&nbsp;&nbsp;&nbsp;`image: docker.io/mongo:7.0`  
&nbsp;&nbsp;&nbsp;&nbsp;`container_name: unifi-db`  
&nbsp;&nbsp;&nbsp;&nbsp;`environment:`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- MONGO_INITDB_ROOT_USERNAME=${MONGO_INITDB_ROOT_USERNAME}`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- MONGO_INITDB_ROOT_PASSWORD=${MONGO_INITDB_ROOT_PASSWORD}`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- MONGO_USER=${MONGO_USER}`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- MONGO_PASS=${MONGO_PASS}`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- MONGO_DBNAME=${MONGO_DBNAME}`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- MONGO_AUTHSOURCE=${MONGO_AUTHSOURCE}`  
&nbsp;&nbsp;&nbsp;&nbsp;`volumes:`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- ./db:/data/db`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- ./init-mongo.js:/docker-entrypoint-initdb.d/init-mongo.js:ro`  
&nbsp;&nbsp;&nbsp;&nbsp;`restart: unless-stopped`

&nbsp;&nbsp;`unifi-network-application:`  
&nbsp;&nbsp;&nbsp;&nbsp;`image: lscr.io/linuxserver/unifi-network-application:7.6.126`  
&nbsp;&nbsp;&nbsp;&nbsp;`container_name: unifi-network-application`  
&nbsp;&nbsp;&nbsp;&nbsp;`environment:`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- PUID=${PUID}`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- PGID=${PGID}`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- TZ=${TZ}`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- MONGO_USER=${MONGO_USER}`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- MONGO_PASS=${MONGO_PASS}`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- MONGO_HOST=${MONGO_HOST}`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- MONGO_PORT=${MONGO_PORT}`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- MONGO_DBNAME=${MONGO_DBNAME}`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- MONGO_AUTHSOURCE=${MONGO_AUTHSOURCE}`  
&nbsp;&nbsp;&nbsp;&nbsp;`volumes:`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- ./config:/config`  
&nbsp;&nbsp;&nbsp;&nbsp;`ports:`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- "8443:8443"      # Web GUI Admin via HTTPS`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- "30001:10001/udp" # Opsional: AP discovery alternatif jika port 10001 bentrok`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- "10001:10001/udp" # AP discovery (L2 adoption)`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- "8080:8080"      # Device communication / Inform port`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- "3478:3478/udp"  # UniFi STUN port`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- "8843:8843"      # Guest portal HTTPS`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- "8880:8880"      # Guest portal HTTP`  
&nbsp;&nbsp;&nbsp;&nbsp;`restart: unless-stopped`  
&nbsp;&nbsp;&nbsp;&nbsp;`depends_on:`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- unifi-db`

&nbsp;

**9. Jalankan Seluruh Stack Kontainer**

Deployment Akhir

&nbsp;

Eksekusi stack Docker Compose Anda secara *detached* (berjalan di latar belakang) menggunakan perintah resmi berikut:

`docker compose up -d`

Proses inisialisasi awal database biasanya memakan waktu sekitar 1 hingga 2 menit. Anda bisa memantau prosesnya secara real-time lewat log:

`docker logs -f unifi-network-application`

Setelah container berhasil berjalan tanpa kendala (*status: running*), Anda kini dapat membuka browser kesayangan Anda lalu mengakses dashboard web UniFi Controller pada tautan berikut:

**Alamat Akses:** https://\<IP\_SERVER\_UBUNTU\_ANDA\>:8443

*(Abaikan peringatan SSL/TLS "Your connection is not private" saat pertama kali masuk karena UniFi menggunakan self-signed certificate bawaan).*

⚠️ **Penting untuk Adopsi Perangkat (L3/Inform IP Override):** Karena UniFi berjalan di dalam jaringan internal Docker (bridge mode), perangkat fisik Anda seperti Access Point atau Switch tidak akan mendeteksi keberadaan controller secara otomatis.

Begitu Anda berhasil menyelesaikan wizard setup di Web UI, segera masuk ke **Settings \> System \> Advanced**, cari opsi **Inform Host**, aktifkan centang **Override**, kemudian masukkan **IP asli dari Server Ubuntu** Anda. Langkah ini wajib dilakukan agar perangkat UniFi Anda tidak berstatus *Disconnected* atau *Adoption Failed*.

**Sources**

> 1. [How to Install Docker on Ubuntu – Step-by-Step Guide | DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-22-04)  
> 2. [pjortiz/docker-compose-unifi-network-application \- GitHub](https://github.com/pjortiz/docker-compose-unifi-network-application)  
> 3. [How to Install Docker on Ubuntu 24.04: Step-by-Step Guide | Cherry Servers](https://www.cherryservers.com/blog/install-docker-ubuntu)

---

*User prompt: docker compose belum terinstall*

Response: Masalah ini sering sekali terjadi karena adanya perubahan cara Docker mendistribusikan Docker Compose pada versi-versi terbaru.

Dulu, Docker Compose adalah aplikasi mandiri yang dipanggil dengan perintah pakai strip (docker-compose). Sekarang, Docker menjadikannya sebagai **plugin resmi** yang dipanggil menggunakan **spasi** (docker compose).

Mari kita bereskan ini dengan dua langkah cepat berikut:

&nbsp;

&nbsp;

**1. Cek Penggunaan Sintaksis (Spasi vs Strip)**

Deteksi awal

&nbsp;

Coba ketik perintah ini di terminal Anda (pastikan menggunakan **spasi**, bukan tanda hubung/strip):

`docker compose version`

Jika perintah di atas memunculkan versi (misalnya *Docker Compose version v2.x.x*), artinya Docker Compose **sudah terinstall** dan Anda hanya perlu mengganti perintah docker-compose up \-d menjadi docker compose up \-d pada langkah sebelumnya.

&nbsp;

**2. Install Ulang Plugin via APT**

Jika perintah tetap tidak ditemukan

&nbsp;

Jika langkah pertama tetap menghasilkan error *command not found*, kemungkinan paket plugin terlewat saat instalasi awal. Jalankan perintah resmi ini untuk memasangnya langsung dari repositori Docker yang sudah kita daftarkan tadi:

`sudo apt-get update`  
`sudo apt-get install -y docker-compose-plugin`

Setelah proses selesai, verifikasi kembali dengan mengetik docker compose version.

💡 **Tips Tambahan:** Jika Anda terbiasa mengetik menggunakan tanda strip (docker-compose) dan ingin perintah itu tetap berfungsi, Anda bisa membuat pintasan (alias) di sistem Ubuntu dengan perintah ini:

`echo "alias docker-compose='docker compose'" >> ~/.bashrc`  
`source ~/.bashrc`

Setelah docker compose version memberikan output versi yang benar, Anda bisa kembali ke folder infra Anda dengan cd \~/unifi-infra dan langsung mengeksekusi perintah akhir:

`docker compose up -d`

---

*User prompt: docker compose plugin already installed but i cant run command docker compose version*

Response: Sangat menyebalkan ketika sistem mengatakan sebuah paket "sudah terpasang" (*already installed*) tetapi perintahnya tetap tidak mau berjalan saat dipanggil.

Pada Ubuntu 24.04, masalah ini biasanya disebabkan oleh salah satu dari tiga hal: **masalah hak akses (permission)** pada Docker socket, **sesi terminal yang belum diperbarui** setelah mendaftarkan user ke grup Docker, atau **bentrok antara Docker versi APT dan versi Snap** bawaan Ubuntu.

Mari kita telusuri dan selesaikan masalah ini secara berurutan menggunakan langkah-langkah di bawah ini:

&nbsp;

&nbsp;

**1. Coba Jalankan Menggunakan Sudo**

Uji hak akses dasar

&nbsp;

Langkah pertama adalah memastikan apakah ini murni masalah izin akses (*permission*) atau memang sistem tidak bisa menemukan binari pluginnya. Jalankan perintah ini:

`sudo docker compose version`

* **Jika berhasil memunculkan versi:** Masalahnya ada pada izin akses user Anda (lanjut ke **Langkah 2**).  
* **Jika tetap error "command not found":** Berarti binari Docker atau pluginnya tidak berada di *path* yang benar (lanjut ke **Langkah 4**).

&nbsp;

**2. Terapkan Perubahan Grup Docker Tanpa Log Out**

Perbarui sesi user

&nbsp;

Di langkah sebelumnya, kita sudah memasukkan user Anda ke grup docker menggunakan usermod. Namun, Ubuntu tidak akan mendeteksi perubahan ini sebelum Anda memperbarui sesi terminal.

Jalankan perintah berikut untuk memaksa sistem membaca grup baru Anda secara instan:

`newgrp docker`

Setelah itu, coba jalankan kembali tanpa sudo:

`docker compose version`

&nbsp;

**3. Pastikan Layanan Docker Sedang Berjalan**

Cek status engine

&nbsp;

Jika perintah menggantung (*hang*) atau memberikan error *cannot connect to the Docker daemon*, kemungkinan besar layanan Docker belum aktif di latar belakang sistem Ubuntu 24.04 Anda.

Aktifkan dan jalankan layanannya dengan perintah ini:

`sudo systemctl enable docker`  
`sudo systemctl start docker`

Lalu cek statusnya untuk memastikan keterangannya berwarna hijau (**active/running**):

`sudo systemctl status docker`

&nbsp;

**4. Periksa Konflik Docker Versi Snap**

Atasi konflik Ubuntu 24.04

&nbsp;

Ubuntu 24.04 sering kali membawa paket Docker versi **Snap** secara bawaan yang bisa bentrok dengan Docker versi **APT** yang baru saja kita pasang.

Ketik perintah berikut untuk melihat dari mana perintah docker Anda berjalan:

`which docker`

* Jika hasilnya /snap/bin/docker, berarti sistem Anda mendahulukan versi Snap (yang tidak memiliki plugin Compose kita).  
* Jika hasilnya /usr/bin/docker, maka jalurnya sudah benar.

Jika yang aktif adalah versi Snap, Anda bisa menghapusnya agar tidak terjadi tumpang tindih:

`sudo snap remove docker`

### **Bagaimana cara memverifikasi jika semuanya sudah beres?**

Jika langkah-langkah di atas sudah dilewati, silakan jalankan kembali perintah verifikasi ini:

`docker compose version`

Sistem seharusnya akan memberikan respons bersih seperti ini: Docker Compose version v2.x.x (angka versi bisa bervariasi). Setelah output ini muncul, Anda bisa langsung kembali ke folder proyek Anda (cd \~/unifi-infra) dan menjalankan UniFi Controller Anda dengan lancar menggunakan docker compose up \-d.

---

*User prompt: ok done for docker compose, give information back to unifi controller install on the docker*

Response: Mantap\! Sekarang setelah docker compose sudah terpasang dan berfungsi dengan benar, kita bisa langsung melanjutkan proses deployment UniFi Network Application 7.6 yang sempat tertunda.

Mari kita eksekusi langkah akhir untuk menyalakan dan mengonfigurasi UniFi Controller Anda.

&nbsp;

&nbsp;

**1. Masuk ke Direktori UniFi**

Kembali ke Workspace

&nbsp;

Pastikan Anda berada di dalam folder infrastruktur tempat berkas .env, init-mongo.js, dan docker-compose.yml yang kita buat sebelumnya berada:

`cd ~/unifi-infra`

*(Opsional)* Anda bisa mengetik ls \-la untuk memastikan ketiga file tersebut sudah lengkap di dalam folder.

&nbsp;

**2. Jalankan Kontainer UniFi & MongoDB**

Perintah Resmi Docker Compose

&nbsp;

Nyalakan seluruh stack kontainer di latar belakang (*detached mode*) menggunakan perintah resmi berikut:

`docker compose up -d`

Docker akan otomatis mengunduh (*pull*) image MongoDB 7.0 dan UniFi 7.6.126 (jika belum ada), lalu membuat kontainernya sesuai variabel dari berkas .env Anda.

&nbsp;

**3. Pantau Proses Inisialisasi Awal**

Verifikasi Sistem

&nbsp;

Saat pertama kali dinyalakan, MongoDB akan menjalankan script init-mongo.js untuk membuat user, dan UniFi akan melakukan setup skema database. Proses ini biasanya memakan waktu 1–2 menit.

Pantau log-nya untuk memastikan tidak ada error:

`docker logs -f unifi-network-application`

Tunggu sampai Anda melihat baris log yang menandakan aplikasi siap menerima koneksi (biasanya memunculkan informasi tentang *Web UI started* atau *Tomcat initialized*). Tekan CTRL \+ C untuk keluar dari pemantauan log.

&nbsp;

**4Akses Web GUI UniFi Controller**

Konfigurasi Lewat Browser

&nbsp;

Sekarang, buka browser Anda dan akses halaman administrasi UniFi melalui IP server Ubuntu Anda:

`https://<IP_SERVER_UBUNTU_ANDA>:8443`

*Catatan: Karena UniFi menggunakan sertifikat SSL bawaan (self-signed), browser Anda akan menampilkan peringatan keamanan. Cukup klik **Advanced** lalu pilih **Proceed to... (unsafe) / Lanjutkan**.*

### **⚠️ Langkah Wajib Pasca-Instalasi (Penting\!)**

Karena UniFi Controller ini berjalan di dalam isolasi jaringan Docker (*bridge mode*), perangkat UniFi fisik Anda (seperti Access Point atau Switch) tidak akan bisa mendeteksi keberadaan controller ini secara otomatis.

Setelah Anda menyelesaikan wizard setup awal di Web GUI, **segera lakukan hal ini:**

> 1. Masuk ke menu **Settings** (Ikon gerigi di kiri bawah).  
> 2. Pergi ke tab **System** \-\> **Advanced**.  
> 3. Cari opsi bernama **Inform Host**.  
> 4. Aktifkan centang pada **Override**.  
> 5. Masukkan **IP lokal asli dari Server Ubuntu** Anda (contoh: 192.168.1.50), bukan IP internal Docker.  
> 6. Klik **Apply Changes**.

Langkah ini menjamin bahwa setiap kali perangkat UniFi Anda mencoba melakukan *check-in* (inform), mereka akan diarahkan langsung ke IP server Ubuntu Anda melalui port 8080 yang sudah kita buka di berkas docker-compose.yml. Selamat mencoba, controller Anda kini sudah siap digunakan\!

---

*User prompt: there is problem, mongo db container still restarting, unifi container cannot reach mongo db*

Response: Masalah MongoDB yang terus-menerus melakukan *restart* (*restart loop*) adalah kendala klasik saat melakukan setup UniFi di Docker. Ketika MongoDB tidak stabil, UniFi otomatis tidak akan bisa terhubung dan ikut *crash* atau gagal mendengarkan port.

Pada Ubuntu 24.04, ada dua penyebab utama mengapa hal ini terjadi. Mari kita periksa dan selesaikan masalah ini sekarang.

### **Langkah 1: Diagnosis Masalah (Cek Log MongoDB)**

Sebelum memperbaiki, kita harus tahu persis apa penyebab MongoDB mogok kerja. Jalankan perintah ini di terminal Anda:

`docker logs unifi-db`

Perhatikan baris terakhir pada log tersebut. Masalah Anda hampir pasti disebabkan oleh salah satu dari dua hal di bawah ini:

&nbsp;

&nbsp;

**1. Masalah Instruksi AVX CPU (Error:**

Penyebab \#1 (Paling Sering di VPS/VM)

&nbsp;

Jika log Anda memunculkan pesan Illegal instruction atau kontainer langsung mati tanpa pesan jelas, ini disebabkan karena **MongoDB versi 5.0 ke atas (termasuk versi 7.0 yang kita gunakan) mewajibkan CPU mendukung instruksi AVX**.

Banyak VPS murah (seperti DigitalOcean, Hetzner, Proxmox tanpa passthrough) atau CPU lawas yang tidak mendukung AVX.

**Solusinya:** Downgrade MongoDB ke versi **4.4** (versi terakhir yang tidak mewajibkan AVX). UniFi 7.6 masih sangat kompatibel dengan MongoDB 4.4.

1. Buka berkas docker-compose.yml:

`nano docker-compose.yml`

2. Ubah baris image: docker.io/mongo:7.0 menjadi:

`image: docker.io/mongo:4.4`

3. Simpan dan keluar (CTRL+X, lalu Y, lalu Enter).

&nbsp;

**2. Masalah Izin Akses Folder (Permission Denied)**

Penyebab \#2

&nbsp;

Jika log memunculkan pesan seperti Permission denied atau input/output error saat mencoba menulis ke /data/db, ini karena folder ./db di Ubuntu Anda dibuat oleh user root, sehingga user internal MongoDB (UID 999\) tidak diizinkan untuk menulis data ke sana.

**Solusinya:** Berikan kepemilikan folder ./db kepada user MongoDB secara manual dengan perintah ini:

`sudo chown -R 999:999 ./db`

&nbsp;

**3. Reset dan Jalankan Ulang Stack**

Membersihkan sisa instalasi yang gagal

&nbsp;

Jika Anda melakukan perubahan versi MongoDB (pada Langkah 1\) atau memperbaiki izin folder (pada Langkah 2), Anda **wajib membersihkan sisa database yang korup** akibat proses inisialisasi yang gagal sebelumnya:

1. Matikan stack dan hapus volume lama:

`docker compose down`

2. Bersihkan isi folder ./db secara total (pastikan Anda berada di direktori \~/unifi-infra):

`sudo rm -rf ./db/*`

3. Jalankan kembali stack kontainer Anda:

`docker compose up -d`

### **Langkah 2: Pastikan Semuanya Berjalan Normal**

Setelah menjalankan ulang, tunggu sekitar 30 detik lalu jalankan perintah pemantauan ini untuk memastikan kedua kontainer berstatus **Up** (bukan *Restarting*):

`docker compose ps`

Anda juga bisa memastikan MongoDB berhasil melakukan inisialisasi tanpa hambatan dengan mengecek log-nya sekali lagi:

`docker logs unifi-db`

*(Jika sukses, Anda akan melihat baris log aktif seperti Waiting for connections di bagian paling bawah).*

Jika status MongoDB sudah stabil (Up), kontainer UniFi secara otomatis akan berhasil terhubung ke database dalam hitungan detik dan Anda bisa langsung mengakses Web UI UniFi di port 8443\.

---

*User prompt: container unifi and mongo db is run, but i cant access web ui unifi*

Response: It is a great milestone that both containers are now stable and running\! Since the backend is healthy, the issue with accessing the Web UI is almost certainly a networking, protocol, or firewall barrier.

Let's systematically troubleshoot and get you into that dashboard.

&nbsp;

&nbsp;

**1. Force HTTPS Protocol in Your Browser**

The HTTPS Requirement

&nbsp;

UniFi's Web UI **strictly rejects** standard HTTP connections. If you type just the IP and port, your browser might default to HTTP and show a "Connection Reset" or blank page.

Make sure you are explicitly typing the https:// prefix:

`https://YOUR_SERVER_IP:8443`

*Note: Your browser will warn you about an "Insecure Connection" or "Invalid Certificate". This is normal because UniFi uses a self-signed SSL certificate. Click **Advanced** and then **Proceed**.*

&nbsp;

**2. Verify the UniFi Application is Fully Initialized**

Check Java Boot Status

&nbsp;

Just because the Docker container says "Up" or "Running" doesn't mean the UniFi web server is ready. The UniFi Java application takes **1 to 3 minutes** to fully boot up and start listening on port 8443\.

Run this command to check the actual startup logs:

`docker logs unifi-network-application | grep -i -E "started|tomcat|ready"`

If you do not see boot-up confirmation logs, run:

`docker logs --tail 50 unifi-network-application`

Ensure there are no Java Out-Of-Memory (OOM) errors or database connection timeouts listed at the end.

&nbsp;

**3. Open the Ports in Ubuntu Firewall (UFW)**

Ubuntu 24.04 Security

&nbsp;

Ubuntu 24.04 often has the UFW firewall active by default, which blocks external traffic on port 8443 even if Docker is forwarding it.

Allow traffic on the crucial UniFi ports by running:

`sudo ufw allow 8443/tcp`  
`sudo ufw allow 8080/tcp`  
`sudo ufw allow 3478/udp`  
`sudo ufw reload`

*If you are running this on a cloud VPS (like AWS, DigitalOcean, or Oracle Cloud), you must also open port 8443 TCP in your cloud provider's Security Group/Firewall dashboard.*

&nbsp;

**4. Check if the Port is Active on the Host**

Docker Proxy Test

&nbsp;

Let's check if Ubuntu itself is actually listening on port 8443\. Run this command:

`sudo ss -tulpn | grep 8443`

You should see a line showing docker-proxy or dockerd listening on 0.0.0.0:8443 or :::8443.

* If you see it, the port is open and listening.  
* If you see nothing, Docker did not bind the port successfully (likely due to another service already using port 8443 on your machine).

💡 **Quick Test:** If you are running this on a local Ubuntu machine (not a remote headless server), you can test if the port is responding locally inside the server using curl:

`curl -k https://127.0.0.1:8443`

If it returns HTML code, the UniFi controller is working perfectly, confirming that the issue is entirely a network firewall blocking your remote browser.

---

*User prompt: error like : Caused by: com.mongodb.MongoCommandException: Command failed with error 18 (AuthenticationFailed): 'Authentication failed.' on server unifi-db:27017. The full response is {"ok": 0.0, "errmsg": "Authentication failed.", "code": 18, "codeName": "AuthenticationFailed"}         at com.mongodb.internal.connection.ProtocolHelper.getCommandFailureException(ProtocolHelper.java:205)         at com.mongodb.internal.connection.InternalStreamConnection.receiveCommandMessageResponse(InternalStreamConnection.java:515)         at com.mongodb.internal.connection.InternalStreamConnection.sendAndReceiveInternal(InternalStreamConnection.java:438)         at com.mongodb.internal.connection.InternalStreamConnection.lambda$sendAndReceive$0(InternalStreamConnection.java:366)         at com.mongodb.internal.connection.InternalStreamConnection.sendAndReceive(InternalStreamConnection.java:369)         at com.mongodb.internal.connection.CommandHelper.sendAndReceive(CommandHelper.java:92)         at com.mongodb.internal.connection.CommandHelper.executeCommand(CommandHelper.java:48)         at com.mongodb.internal.connection.SaslAuthenticator.sendSaslStart(SaslAuthenticator.java:231)         at com.mongodb.internal.connection.SaslAuthenticator.getNextSaslResponse(SaslAuthenticator.java:138)         ... 67 more 20:17:52,801 |-INFO in ch.qos.logback.classic.model.processor.LoggerModelHandler \- Setting level of logger \[org.apache.tomcat\] to ERROR*

Response: Error AuthenticationFailed (code 18\) ini terjadi karena adanya **ketidakcocokan lokasi database tempat user disimpan**.

Pada langkah sebelumnya, di dalam file init-mongo.js kita membuat user menggunakan perintah db.getSiblingDB('unifi'), yang artinya user unifi dibuat di dalam database **unifi**. Namun, di file .env kita mengatur MONGO\_AUTHSOURCE=admin, yang memerintahkan UniFi untuk mencari user tersebut di database **admin**. Karena tidak ditemukan di sana, MongoDB menolak koneksi tersebut.

Berikut adalah langkah cepat untuk menyelaraskan konfigurasi dan melakukan *reset* database agar inisialisasi berjalan 100% bersih:

&nbsp;

&nbsp;

**1. Ubah MONGO\_AUTHSOURCE di File .env**

Koreksi Konfigurasi

&nbsp;

Buka kembali file .env Anda:

`nano .env`

Cari baris MONGO\_AUTHSOURCE dan ubah nilainya dari admin menjadi unifi (menyamakan dengan database tempat user dibuat):

`MONGO_AUTHSOURCE=unifi`

Simpan perubahan dengan menekan CTRL+X, lalu Y, dan Enter.

&nbsp;

**2. Hentikan Container & Bersihkan Database Lama**

Pembersihan Total

&nbsp;

Karena kegagalan autentikasi sebelumnya mungkin membuat proses inisialisasi database menjadi setengah matang atau korup, kita wajib membersihkannya terlebih dahulu agar MongoDB mau membaca ulang file init-mongo.js dari kondisi kosong:

`# Hentikan semua container`  
`docker compose down`

`# Hapus semua sisa data database yang lama (pastikan berada di ~/unifi-infra)`  
`sudo rm -rf ./db/*`

&nbsp;

**3. Jalankan Kembali Stack Docker**

Menyalakan Ulang

&nbsp;

Sekarang, jalankan kembali stack dengan konfigurasi baru yang sudah selaras:

`docker compose up -d`

&nbsp;

**4Pantau Log Koneksi**

Pengecekan Akhir

&nbsp;

Tunggu sekitar 30 detik agar database selesai melakukan inisialisasi awal, lalu periksa log UniFi untuk memastikan koneksi ke database kini sudah sukses terjalin:

`docker logs -f unifi-network-application`

Jika sukses, Anda tidak akan melihat lagi error Authentication failed merah, melainkan log inisialisasi sistem UniFi (Java/Tomcat) yang berjalan normal hingga selesai.

Setelah log menunjukkan status berjalan dengan aman, Anda bisa langsung mengakses kembali Web UI di browser Anda melalui \`

---

*User prompt: in log unifi-db shown authenticationDatabase":"admin","remote":"172.18.0.3:39466","extraInfo":{},"error":"UserNotFound: Could not find user \\"unifi\\" for db \\"admin\\""}}*

Response: Log ini sangat memperjelas masalahnya\! Terima kasih telah melampirkan baris log tersebut.

Pesan UserNotFound: Could not find user "unifi" for db "admin" mengonfirmasi bahwa UniFi (atau mekanisme internal dari *image* Docker LinuxServer) **selalu memaksa proses autentikasi awal dilakukan terhadap database admin**, terlepas dari parameter MONGO\_AUTHSOURCE yang kita ganti di berkas .env.

Karena pada langkah sebelumnya kita membuat user unifi di dalam database unifi (bukan admin), MongoDB menolak koneksi tersebut karena tidak menemukan user tersebut di dalam daftar database admin.

Solusi paling ampuh dan standar industri untuk MongoDB adalah **membuat user unifi di dalam database admin**, tetapi memberikan hak akses penuh (*roles*) khusus untuk database unifi dan unifi\_stat. Dengan begitu, proses login sukses dan UniFi tetap hanya bisa mengakses databasenya sendiri.

Mari kita terapkan perubahan ini:

&nbsp;

&nbsp;

**1. Update Berkas init-mongo.js**

Perbaiki Lokasi User

&nbsp;

Buka file inisialisasi database Anda:

`nano init-mongo.js`

Hapus seluruh isinya dan ganti dengan script di bawah ini. Perhatikan bahwa kita menggunakan getSiblingDB('admin') untuk membuat user di database admin, tetapi memberikan peran dbOwner khusus ke database milik UniFi:

`db.getSiblingDB('admin').createUser({`  
&nbsp;&nbsp;`user: process.env.MONGO_USER,`  
&nbsp;&nbsp;`pwd: process.env.MONGO_PASS,`  
&nbsp;&nbsp;`roles: [`  
&nbsp;&nbsp;&nbsp;&nbsp;`{ role: "dbOwner", db: process.env.MONGO_DBNAME },`  
&nbsp;&nbsp;&nbsp;&nbsp;`{ role: "dbOwner", db: process.env.MONGO_DBNAME + '_stat' }`  
&nbsp;&nbsp;`]`  
`});`

Simpan dan keluar (CTRL+X, lalu Y, lalu Enter).

&nbsp;

**2. Pastikan .env Menggunakan Database admin**

Kembalikan Parameter

&nbsp;

Buka file .env Anda:

`nano .env`

Pastikan nilai MONGO\_AUTHSOURCE diatur kembali ke admin (karena sekarang user kita buat di sana):

`MONGO_AUTHSOURCE=admin`

Simpan dan keluar.

&nbsp;

**3. Hapus Data Lama & Jalankan Ulang**

Reset Database

&nbsp;

Agar MongoDB mengeksekusi ulang file init-mongo.js yang baru saja kita perbarui, kita harus mematikan container dan mengosongkan folder database lama yang setengah matang:

`# Matikan stack container`  
`docker compose down`

`# Bersihkan sisa data lama yang gagal`  
`sudo rm -rf ./db/*`

`# Nyalakan kembali stack`  
`docker compose up -d`

&nbsp;

**4. Pantau Proses Booting UniFi**

Verifikasi Log

&nbsp;

Tunggu sekitar 30 detik, lalu pantau log kontainer UniFi untuk memastikan koneksi ke database berjalan sukses tanpa error autentikasi:

`docker logs -f unifi-network-application`

Begitu Anda melihat log menunjukkan bahwa aplikasi Java UniFi mulai memuat komponen web (*Tomcat* atau *Web UI started*), silakan buka browser Anda kembali dan akses:

**URL:** \`

---

*User prompt: error Caused by: com.mongodb.MongoCommandException: Command failed with error 18 (AuthenticationFailed): 'Authentication failed.' on server unifi-db:27017. The full response is {"ok": 0.0, "errmsg": "Authentication failed.", "code": 18, "codeName": "AuthenticationFailed"}         at com.mongodb.internal.connection.ProtocolHelper.getCommandFailureException(ProtocolHelper.java:205)         at com.mongodb.internal.connection.InternalStreamConnection.receiveCommandMessageResponse(InternalStreamConnection.java:515)         at com.mongodb.internal.connection.InternalStreamConnection.sendAndReceiveInternal(InternalStreamConnection.java:438)         at com.mongodb.internal.connection.InternalStreamConnection.lambda$sendAndReceive$0(InternalStreamConnection.java:366)         at com.mongodb.internal.connection.InternalStreamConnection.sendAndReceive(InternalStreamConnection.java:369)         at com.mongodb.internal.connection.CommandHelper.sendAndReceive(CommandHelper.java:92)         at com.mongodb.internal.connection.CommandHelper.executeCommand(CommandHelper.java:48)         at com.mongodb.internal.connection.SaslAuthenticator.sendSaslStart(SaslAuthenticator.java:231)         at com.mongodb.internal.connection.SaslAuthenticator.getNextSaslResponse(SaslAuthenticator.java:138)*

Response: Error AuthenticationFailed (code 18\) ini kembali muncul karena sebuah detail teknis pada versi MongoDB yang kita gunakan:

Ketika kita menurunkan (*downgrade*) versi MongoDB ke **4.4** (untuk menghindari error CPU AVX), MongoDB menggunakan engine CLI lama bernama mongo shell (bukan mongosh versi baru). Pada mongo shell versi lama, perintah **process.env tidak didukung** untuk membaca variabel lingkungan OS dari dalam file JavaScript (init-mongo.js).

Akibatnya, variabel MONGO\_USER dan MONGO\_PASS dibaca sebagai undefined, sehingga user database gagal dibuat dan UniFi ditolak saat mencoba masuk.

### **Solusi Terbaik & Paling Aman: Gunakan Root User Secara Langsung**

Karena database MongoDB ini berjalan di dalam jaringan internal Docker yang terisolasi dan **port 27017 tidak dibuka ke publik/host** di file docker-compose.yml, maka sangat aman dan jauh lebih praktis untuk mengoneksikan UniFi menggunakan **Root User** secara langsung.

Cara ini mengeliminasi kebutuhan file init-mongo.js sepenuhnya, sehingga Anda terbebas dari bug ketidakcocokan versi MongoDB untuk selamanya.

Ikuti langkah perbaikan bersih berikut ini:

&nbsp;

&nbsp;

**1. Perbarui Berkas .env**

Koreksi Variabel

&nbsp;

Buka file .env Anda:

`nano .env`

Ubah isinya menjadi lebih sederhana seperti di bawah ini. Kita akan menggunakan kredensial root untuk semuanya:

`# Kredensial Utama MongoDB (Root)`  
`MONGO_USER=rootadmin`  
`MONGO_PASS=PasswordRootRahasia123!`  
`MONGO_DBNAME=unifi`  
`MONGO_HOST=unifi-db`  
`MONGO_PORT=27017`  
`MONGO_AUTHSOURCE=admin`

`# Parameter Lingkungan Kontainer`  
`TZ=Asia/Jakarta`  
`PUID=1000`  
`PGID=1000`

Simpan dengan menekan CTRL+X, lalu Y, dan Enter.

&nbsp;

**2. Sesuaikan docker-compose.yml & Hapus File JS**

Sederhanakan Konfigurasi

&nbsp;

Buka berkas docker-compose.yml Anda:

`nano docker-compose.yml`

Ubah konfigurasinya agar MongoDB menggunakan variabel root langsung dari .env tanpa memuat file init-mongo.js lagi:

`services:`  
&nbsp;&nbsp;`unifi-db:`  
&nbsp;&nbsp;&nbsp;&nbsp;`image: docker.io/mongo:4.4  # Tetap gunakan 4.4 jika CPU Anda tidak mendukung AVX`  
&nbsp;&nbsp;&nbsp;&nbsp;`container_name: unifi-db`  
&nbsp;&nbsp;&nbsp;&nbsp;`environment:`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- MONGO_INITDB_ROOT_USERNAME=${MONGO_USER}`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- MONGO_INITDB_ROOT_PASSWORD=${MONGO_PASS}`  
&nbsp;&nbsp;&nbsp;&nbsp;`volumes:`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- ./db:/data/db`  
&nbsp;&nbsp;&nbsp;&nbsp;`restart: unless-stopped`

&nbsp;&nbsp;`unifi-network-application:`  
&nbsp;&nbsp;&nbsp;&nbsp;`image: lscr.io/linuxserver/unifi-network-application:7.6.126`  
&nbsp;&nbsp;&nbsp;&nbsp;`container_name: unifi-network-application`  
&nbsp;&nbsp;&nbsp;&nbsp;`environment:`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- PUID=${PUID}`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- PGID=${PGID}`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- TZ=${TZ}`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- MONGO_USER=${MONGO_USER}`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- MONGO_PASS=${MONGO_PASS}`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- MONGO_HOST=${MONGO_HOST}`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- MONGO_PORT=${MONGO_PORT}`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- MONGO_DBNAME=${MONGO_DBNAME}`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- MONGO_AUTHSOURCE=${MONGO_AUTHSOURCE}`  
&nbsp;&nbsp;&nbsp;&nbsp;`volumes:`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- ./config:/config`  
&nbsp;&nbsp;&nbsp;&nbsp;`ports:`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- "8443:8443"      # Web GUI Admin via HTTPS`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- "10001:10001/udp" # AP discovery (L2 adoption)`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- "8080:8080"      # Device communication / Inform port`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- "3478:3478/udp"  # UniFi STUN port`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- "8843:8843"      # Guest portal HTTPS`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- "8880:8880"      # Guest portal HTTP`  
&nbsp;&nbsp;&nbsp;&nbsp;`restart: unless-stopped`  
&nbsp;&nbsp;&nbsp;&nbsp;`depends_on:`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`- unifi-db`

Simpan dan keluar. Setelah itu, **hapus berkas init-mongo.js** yang sudah tidak diperlukan lagi agar direktori Anda tetap bersih:

`rm init-mongo.js`

&nbsp;

**3. Reset Data Lama & Jalankan Ulang**

Pembersihan Sisa Database

&nbsp;

Untuk memastikan tidak ada sisa-sisa autentikasi lama yang korup di dalam folder database, jalankan perintah pembersihan total ini:

`# Matikan stack kontainer saat ini`  
`docker compose down`

`# Bersihkan folder data lama`  
`sudo rm -rf ./db/*`

`# Jalankan kembali stack kontainer dari awal`  
`docker compose up -d`

&nbsp;

**4. Pantau Log Aplikasi UniFi**

Uji Koneksi

&nbsp;

Tunggu sekitar 30-45 detik agar sistem melakukan inisialisasi awal database barunya, lalu cek log kontainer UniFi Anda:

`docker logs -f unifi-network-application`

**Hasil yang Diharapkan:** Pada log UniFi, Anda kini tidak akan lagi menemui error AuthenticationFailed. Proses booting Java akan terus berjalan hingga menampilkan log inisialisasi Tomcat/Web Server selesai.

Begitu log stabil, buka kembali browser Anda dan akses halaman instalasi UniFi di: \`

---

*User prompt: di versi unifi 10 dimana menemukan: Karena UniFi Controller ini berjalan di dalam isolasi jaringan Docker (bridge mode), perangkat UniFi fisik Anda (seperti Access Point atau Switch) tidak akan bisa mendeteksi keberadaan controller ini secara otomatis. Setelah Anda menyelesaikan wizard setup awal di Web GUI, segera lakukan hal ini: Masuk ke menu Settings (Ikon gerigi di kiri bawah). Pergi ke tab System \-\> Advanced. Cari opsi bernama Inform Host. Aktifkan centang pada Override. Masukkan IP lokal asli dari Server Ubuntu Anda (contoh: 192.168.1.50), bukan IP internal Docker. Klik Apply Changes.*

Response: Di UniFi Network Application versi terbaru (termasuk versi 8, 9, hingga versi 10), Ubiquiti sedikit merapikan tata letak menu *Settings* mereka agar lebih bersih, namun lokasinya tidak bergeser terlalu jauh.

Untuk menemukan opsi tersebut di versi terbaru, silakan ikuti jalur berikut:

> 1. **Buka Settings:** Klik ikon **Gerigi (Settings)** di sudut kiri bawah dasbor UniFi Anda.  
> 2. **Pilih System:** Di panel navigasi sebelah kiri, klik menu **System**.  
> 3. **Buka Menu Advanced:** Di halaman utama *System*, gulir ke bawah sampai Anda menemukan bagian **Advanced**. Jika bagian ini tertutup (collapsed), klik untuk membukanya.  
> 4. **Temukan Override Inform Host:**  
   * Cari opsi bernama **Override Inform Host** (pada beberapa sub-versi terbaru, opsi ini dikelompokkan di dalam sub-bagian **Device Adoption**).  
   * **Aktifkan/Toggle** opsi tersebut ke posisi *On* (aktif).  
> 5. **Masukkan IP Server:** Pada kolom **Host for Inform** yang muncul tepat di bawahnya, masukkan IP lokal asli dari Server Ubuntu Anda (misalnya: 192.168.1.50).  
> 6. **Simpan Perubahan:** Klik tombol **Apply Changes** di bagian bawah halaman.

💡 **Tips Cepat:** Jika Anda kesulitan menemukannya karena perbedaan bahasa atau pembaruan minor tampilan, Anda bisa langsung menggunakan **fitur pencarian (Search Bar)** di bagian paling atas halaman *Settings*, lalu ketik kata kunci **Inform** atau **Override**. Sistem akan langsung mengarahkan Anda ke opsi tersebut secara instan.

---

*User prompt: tidak ada*

Response: Wah, tidak ada ya? Ubiquiti memang terkenal suka sekali memindah-mindahkan atau menyembunyikan menu di setiap pembaruan versi besar, yang terkadang cukup membingungkan.

Jika opsi **Override Inform Host** benar-benar tidak muncul atau sulit ditemukan di Web UI versi Anda, jangan khawatir. Kita tidak wajib mengaktifkan opsi tersebut dari dalam Web UI.

Ada metode alternatif yang justru jauh lebih andal dan sering digunakan oleh para profesional untuk setup UniFi di Docker, yaitu **menggunakan jalur SSH langsung ke perangkat (Access Point atau Switch)**.

## **Cara Adopsi Perangkat via SSH (L3 Adoption)**

Metode ini memaksa perangkat UniFi Anda untuk langsung melapor ke IP Server Ubuntu Anda tanpa bergantung pada fitur penemuan otomatis (auto-discovery) dari Web UI.

&nbsp;

&nbsp;

**1. Cari IP Perangkat & Siapkan SSH**

Persiapan Koneksi

&nbsp;

Temukan IP lokal dari Access Point (AP) atau Switch UniFi Anda (Anda bisa melihatnya di daftar client DHCP pada Router Anda).

Buka terminal baru di komputer Anda, lalu masuk ke perangkat UniFi menggunakan SSH:

`ssh ubnt@<IP_PORT_ACCESS_POINT_ANDA>`

* **Username default:** ubnt  
* **Password default:** ubnt *(Jika perangkat sudah pernah diadopsi sebelumnya, gunakan username dan password admin yang pernah Anda buat di controller lama).*

&nbsp;

**2. Kirim Perintah Inform Pertama**

Koneksi Tahap 1

&nbsp;

Setelah berhasil masuk ke dalam sistem operasi perangkat UniFi via SSH, jalankan perintah berikut untuk mengarahkan perangkat ke IP Server Ubuntu Anda:

`set-inform http://<IP_SERVER_UBUNTU_ANDA>:8080/inform`

*(Ganti \<IP\_SERVER\_UBUNTU\_ANDA\> dengan IP lokal komputer Ubuntu Anda, misalnya 192.168.1.50).*

&nbsp;

**3. Klik Adopt di Web UI UniFi**

Konfirmasi di Browser

&nbsp;

Sekarang, buka Web UI UniFi Anda di browser (https://\<IP\_SERVER\_UBUNTU\_ANDA\>:8443).

Pergi ke menu **Devices**. Perangkat Anda sekarang seharusnya sudah muncul di sana dengan status **Click to Adopt** atau **Pending Adoption**. Klik tombol **Adopt** pada perangkat tersebut.

&nbsp;

**4. Kirim Perintah Inform Kedua (Sangat Penting\!)**

Kunci Koneksi

&nbsp;

Setelah Anda mengklik *Adopt* di Web UI, status perangkat akan berubah menjadi *Adopting*. Pada tahap ini, koneksi belum sepenuhnya terkunci.

Kembali ke terminal SSH Anda yang tadi masih terbuka, lalu **jalankan sekali lagi** perintah yang sama:

`set-inform http://<IP_SERVER_UBUNTU_ANDA>:8080/inform`

Langkah kedua ini wajib dilakukan agar perangkat mengunci alamat IP controller Docker Anda secara permanen.

💡 **Mengapa langkah terakhir wajib dilakukan?** Perangkat UniFi memerlukan proses jabat tangan (*two-way handshake*). Perintah pertama memberi tahu AP untuk meminta adopsi ke controller, sedangkan perintah kedua (setelah Anda mengklik Adopt di UI) mengonfirmasi bahwa AP menyetujui kunci enkripsi baru dari controller dan mengunci alamat inform tersebut secara permanen.

Setelah langkah di atas selesai, status di Web UI UniFi Anda akan berubah menjadi **Online / Active** dalam hitungan detik, dan Anda sudah bisa mengonfigurasi jaringan atau Wi-Fi Anda dengan normal.

---

*User prompt: bagaimana merubah ui unifi versi 10 ke ui model lama*

Response: Di UniFi Network Application, ada perbedaan besar mengenai ketersediaan UI lama (Legacy UI) tergantung pada versi yang sebenarnya Anda jalankan.

Karena di awal kita menginstal **versi 7.6** (tetapi mungkin Anda melihat tampilan barunya seperti versi 10), mari kita bahas dua kondisi ini agar Anda mendapatkan solusi yang tepat.

### **Kondisi 1: Jika Anda Menggunakan UniFi Versi 7.6 (Sesuai Setup Docker Kita)**

Pada versi 7.6, fitur untuk kembali ke tampilan klasik/lama **masih tersedia**. Anda bisa mengubahnya dengan langkah-langkah berikut:

> 1. Masuk ke **Settings** (ikon gerigi di kiri bawah).  
> 2. Pilih menu **System** di panel sebelah kiri.  
> 3. Klik atau gulir ke bawah ke bagian **Advanced**.  
> 4. Cari opsi bernama **Interface**.  
> 5. Di sana akan ada pilihan **New User Interface** (aktif secara default).  
> 6. **Matikan (Toggle Off)** opsi *New User Interface* tersebut.  
> 7. Akan muncul jendela konfirmasi peringatan (*"Switching to Legacy Interface..."*). Klik **Deactivate** untuk menyetujui.  
> 8. Halaman akan memuat ulang (*reload*) dan Anda akan masuk ke tampilan klasik UniFi yang lama.

💡 **Cara kembali ke UI Baru dari UI Lama:** Jika nanti Anda ingin kembali ke tampilan baru saat berada di UI lama, masuk ke **Settings \> User Interface**, lalu aktifkan kembali opsi **New User Interface** dan klik **Apply Changes** di bagian bawah.

### **Kondisi 2: Jika Anda Benar-benar Menggunakan UniFi Versi 8.0 ke Atas (Versi Terbaru)**

Jika Anda tidak sengaja menginstal versi terbaru (v8, v9, atau yang Anda sebut versi 10), Anda perlu mengetahui kebijakan baru dari Ubiquiti:

⚠️ **Catatan Penting:** Mulai dari UniFi Network Application **versi 8.0 ke atas**, Ubiquiti telah **menghapus sepenuhnya opsi Legacy UI** dari sistem mereka.

Keputusan ini diambil karena:

> * Kode pemrograman UI lama sudah usang dan memperlambat kinerja controller.  
> * Banyak fitur keamanan dan manajemen perangkat baru (seperti pengaturan VLAN modern dan manajemen Wi-Fi 7\) tidak kompatibel dengan struktur UI lama.

Jadi, jika controller Anda berada di versi 8 ke atas, satu-satunya cara jika Anda benar-benar membutuhkan UI lama adalah dengan melakukan *downgrade* ke versi 7.x (seperti versi 7.6.126 yang kita buat di awal) dengan cara menghapus kontainer saat ini dan menginstal ulang menggunakan tag versi yang sesuai di file docker-compose.yml.

---
