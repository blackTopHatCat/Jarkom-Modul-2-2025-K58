# Jarkom-Modul-2-2025-K58

| Nama                        | NRP        |
| --------------------------- | ---------- |
| Thio Billy Amansyah         | 5027231007 |
| Ivan Syarifuddin            | 5027241045 | 

## No.1
**SOAL:** Di tepi Beleriand yang porak-poranda, Eonwe merentangkan tiga jalur: Barat untuk Earendil dan Elwing, Timur untuk Círdan, Elrond, Maglor, serta pelabuhan DMZ bagi Sirion, Tirion, Valmar, Lindon, Vingilot. Tetapkan alamat dan default gateway tiap tokoh sesuai glosarium yang sudah diberikan.

**PENJELASAN:** Buat topologi LAN seperti gambar di bawah ini:

<img src="img/1.jpg" />

Berikan IP address pada setiap interface yang terhubung dengan node lain dengan konfigurasi yang kurang lebih seperti dibawah:

```
# Static config for eth0
auto eth0
iface eth0 inet static
	address 192.240.3.2
	netmask 255.255.255.0
	gateway 192.240.3.1
```

## No.2
**SOAL:** Angin dari luar mulai berhembus ketika Eonwe membuka jalan ke awan NAT. Pastikan jalur WAN di router aktif dan NAT meneruskan trafik keluar bagi seluruh alamat internal sehingga host di dalam dapat mencapai layanan di luar menggunakan IP address.

**PENJELASAN:** Hubungkan router dengan node NAT. Berikan IP berdasarkan DHCP server. Lalu jalankan iptables seperti berikut yang menjadikan router sebagai NAT bagi client-clientnya. 

```
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE -s 192.240.0.0/16
```

## No.3
**SOAL:** Kabar dari Barat menyapa Timur. Pastikan kelima klien dapat saling berkomunikasi lintas jalur (routing internal via Eonwe berfungsi), lalu pastikan setiap host non-router menambahkan resolver 192.168.122.1 saat interfacenya aktif agar akses paket dari internet tersedia sejak awal.

**PENJELASAN:** Tambahkan konfigurasi berikut pada konfigurasi interface milik tiap node 

```
up echo nameserver 192.168.122.1 > /etc/resolv.conf
```

## No.4
**SOAL:** Para penjaga nama naik ke menara, di Tirion (ns1/master) bangun zona <xxxx>.com sebagai authoritative dengan SOA yang menunjuk ke ns1.<xxxx>.com dan catatan NS untuk ns1.<xxxx>.com dan ns2.<xxxx>.com. Buat A record untuk ns1.<xxxx>.com dan ns2.<xxxx>.com yang mengarah ke alamat Tirion dan Valmar sesuai glosarium, serta A record apex <xxxx>.com yang mengarah ke alamat Sirion (front door), aktifkan notify dan allow-transfer ke Valmar, set forwarders ke 192.168.122.1. Di Valmar (ns2/slave) tarik zona <xxxx>.com dari Tirion dan pastikan menjawab authoritative pada seluruh host non-router ubah urutan resolver menjadi IP dari ns1.<xxxx>.com → ns2.<xxxx>.com → 192.168.122.1. Verifikasi query ke apex dan hostname layanan dalam zona dijawab melalui ns1/ns2.

**PENJELASAN:** Soal mau kita..

1. Bikin zonefile "k58.com" yang ..

- SOAnya ns1.k58.com
- NSnya ns1.k58.com and ns2.k58.com
- A recordnya ns1.k58.com and ns2.k58.com nunjuk ke IP dns server masing-masing
- A recordnya k58.com nunjuk ke IP server Sirion

```
$TTL            3h
$ORIGIN         k58.com.

@               IN      SOA     n1.k58.com.     admin.k58.net. (
                        1       ; Serial
                        3h      ; Refresh
                        1h      ; Retry
                        1w      ; Expire
                        1h      ; Negative caching TTL
                        )

@               IN      NS      ns1.k58.com. 
@               IN      NS      ns2.k58.com. 

ns1             IN      A       192.240.2.3 ; Tirion
ns2             IN      A       192.240.2.4 ; Valmar
@               IN      A       192.240.2.2 ; Sirion
```

2. Buat master/slave dns server antar Tirion(master) dan Sirion(slave) dengan..

- Menulis `/etc/bind/named.conf.local` di Tirion sbg berikut..
```
zone "k58.com" {
        type master;
        notify yes;
        allow-transfer { 192.240.2.4; }; // IP Sirion/slave
        also-notify { 192.240.2.4; };
        file "/etc/bind/ns1.k58.com";
};
```

- Menulis `/etc/bind/named.conf.options` di Tirion sbg berikut..
```
options {
        directory "/var/cache/bind";
        listen-on port 53 { localhost; 192.240.0.0/16; };
        allow-query { localhost; 192.240.0.0/16; };
        forwarders { 192.168.122.1; };
};
```

- Menulis `/etc/bind/named.conf.local` di Sirion sbg berikut..
```
zone "k58.com" {
        type slave;
        masters { 192.240.2.3; }; // IP Tirion/master
        file "/var/cache/bind/ns2.k58.com";
};
```

3. Menulis `/etc/resolv.conf` seperti berikut pada setiap node..
```
nameserver 192.240.2.3
nameserver 192.240.2.4
nameserver 192.168.122.1
```

## No.5
**SOAL:** “Nama memberi arah,” kata Eonwe. Namai semua tokoh (hostname) sesuai glosarium, eonwe, earendil, elwing, cirdan, elrond, maglor, sirion, tirion, valmar, lindon, vingilot, dan verifikasi bahwa setiap host mengenali dan menggunakan hostname tersebut secara system-wide. Buat setiap domain untuk masing masing node sesuai dengan namanya (contoh: eru.<xxxx>.com) dan assign IP masing-masing juga. Lakukan pengecualian untuk node yang bertanggung jawab atas ns1 dan ns2.

**PENJELASAN:** Assign A record pada setiap node dengan subdomain yang sesuai dengan nama tiap node (eg. lindon jadi lindon.k58.com). Tulis ini pada zonefile di node dns master.

```
...
eonwe           IN      A       192.240.1.1
eonwe           IN      A       192.240.2.1
eonwe           IN      A       192.240.3.1

cirdan          IN      A       192.240.1.2
elrond          IN      A       192.240.1.3
Maglor          IN      A       192.240.1.4

@               IN      A       192.240.2.2
ns1             IN      A       192.240.2.3
ns2             IN      A       192.240.2.4
lindon          IN      A       192.240.2.5
vingilot        IN      A       192.240.2.6

earendil        IN      A       192.240.2.2
elwing          IN      A       192.240.2.3
```


