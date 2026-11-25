# Cisco IOS-XR - Implementation and Operation Guide

## Daftar Isi

- [Modul 1: Architecture IOS-XR dan Matrix Platform](https://github.com/anggrdwjy/Library-Cisco-IOSXR-ServiceProvider#modul-1-architecture-ios-xr-dan-matrix-platform)
- [Modul 2: Workflow Configuration, Commit 2-Stage, Rollback, dan Operasional](https://github.com/anggrdwjy/Library-Cisco-IOSXR-ServiceProvider#modul-2-workflow-configuration-commit-2-stage-rollback-dan-operasional)
- [Modul 3: Fondasi Transport, IGP, MPLS LDP, dan BGP Labeled Unicast](https://github.com/anggrdwjy/Library-Cisco-IOSXR-ServiceProvider#modul-3-fondasi-transport-igp-mpls-ldp-dan-bgp-labeled-unicast)
- [Modul 4: BGP Core, Route Reflector, dan MPLS L3VPN End to End](https://github.com/anggrdwjy/Library-Cisco-IOSXR-ServiceProvider#modul-4-bgp-core-route-reflector-dan-mpls-l3vpn-end-to-end)
- [Modul 5: Segment Routing MPLS, SR Policy, dan TI-LFA](https://github.com/anggrdwjy/Library-Cisco-IOSXR-ServiceProvider#modul-5-segment-routing-mpls-sr-policy-dan-ti-lfa)
- [Modul 6: SRv6, Micro-SID, dan Migrasi dari SR-MPLS](https://github.com/anggrdwjy/Library-Cisco-IOSXR-ServiceProvider#modul-6-srv6-micro-sid-dan-migrasi-dari-sr-mpls)
- [Modul 7: EVPN VPWS, VPLS, dan EVPN E-LAN](https://github.com/anggrdwjy/Library-Cisco-IOSXR-ServiceProvider#modul-7-evpn-vpws-vpls-dan-evpn-e-lan)
- [Modul 8: EVPN L3VPN, Interworking L3VPN dan EVPN](https://github.com/anggrdwjy/Library-Cisco-IOSXR-ServiceProvider#modul-8-evpn-l3vpn-interworking-l3vpn-dan-evpn)
- [Modul 9: Scale, High Availability, QoS Core, dan Security Control Plane](https://github.com/anggrdwjy/Library-Cisco-IOSXR-ServiceProvider#modul-9-scale-high-availability-qos-core-dan-security-control-plane)

## Modul 1: Architecture IOS-XR dan Matrix Platform

### 1.1 Kenapa IOS-XR Beda dari IOS dan IOS-XE

IOS-XR bukan IOS yang diberi fitur baru. IOS-XR adalah OS yang ditulis ulang untuk carrier grade.

- **Kernel:** Berbasis QNX microkernel, bukan monolitik. Satu proses yang crash tidak menjatuhkan seluruh OS.
- **Model configuration:** Two-stage commit. Configuration diedit di target config, lalu committed secara atomik.
- **Control plane terdistribusi:** Proses routing berjalan terpisah dan dapat restarted tanpa reload chassis.
- **Data plane terpisah:** Forwarding di line card tidak bergantung pada RP yang sedang restart, selama NSF dan NSR dirancang benar.
- **Modularitas:** Setiap fitur adalah package. Install, upgrade, dan rollback package dapat dilakukan tanpa reload penuh pada banyak platform.

Anti-pattern yang paling sering ditemui dari engineer IOS: mengetik `write memory`, `copy running-config startup-config`, atau `ip address`. Di IOS-XR, tidak ada konsep startup-config seperti itu. Yang ada adalah committed config dan target config. Perintah yang benar adalah `commit`.

### 1.2 Architecture Internal yang Wajib Dipahami Arsitek

| Komponen | Fungsi | Dampak Design |
| --- | --- | --- |
| SysDB | Database configuration dan state terpusat | Semua proses membaca dari satu source of truth |
| RIB dan FIB | RIB di control plane, FIB diprogram ke hardware | RIB yang besar belum tentu berarti FIB bermasalah, periksa keduanya |
| Process Manager | Mengelola lifecycle proses IOS-XR | Proses BGP atau OSPF dapat restarted mandiri |
| XR Control Plane | Berjalan di RP atau RSP | Disruption RP tidak langsung mematikan forwarding jika NSF aktif |
| Sysadmin Plane | Management chassis, power, fabric, pada NCS 5500 dan ASR 9000 generasi baru | Upgrade dan diagnosa hardware dilakukan dari plane ini |

Prinsip arsitek: jangan pernah mendesain core yang bergantung pada satu RP, satu link, atau satu RR. IOS-XR memberikan alat untuk redundancy, tetapi arsitek yang harus mengaktifkannya.

### 1.3 Matrix Platform

| Platform | Chipset dan Peran | Cocok Untuk | Catatan Arsitek |
| --- | --- | --- | --- |
| ASR 9000 | Typhoon, Tomahawk, Lightspeed. Core dan edge besar | P, PE core, BNG, peering besar | Platform paling matang untuk L3VPN dan SR. Perhatikan generasi line card, karena kapabilitas SRv6 dan scale berbeda |
| NCS 5500 | Jericho, Jericho2. Core dan aggregation padat | P, PE, DCI, peering | Padat, hemat daya, sangat kuat untuk SR-MPLS. Periksa versi XR untuk fitur SRv6 |
| NCS 540 | Jericho2 compact. Access dan aggregation | PE aggregation, pre-aggregation, 5G xHaul | Ideal sebagai PE di edge, bukan sebagai P core besar |
| NCS 5700 | Jericho2C+, 800G. Core generasi baru | P, PE core 800G, DCI | Platform reference untuk SRv6 dan 800G. Gunakan XR 24.x |
| XRv 9000 | Virtual, control plane penuh | Lab, validation, training, CI/CD | Data plane terbatas. Sangat akurat untuk validation configuration, BGP, dan SR control plane. Jangan dipakai untuk benchmark forwarding |

Keputusan platform bukan soal harga per port saja. Pertanyaan arsitek: berapa scale RIB, FIB, label, VRF, dan BGP session yang dibutuhkan dalam 3 tahun. NCS 540 yang dipaksakan menjadi P core akan kehabisan scale lebih cepat daripada ASR 9000 atau NCS 5700.

### 1.4 Versi IOS-XR dan Licensing

- **ASR 9000:** Umumnya di train 7.x, contoh 7.8.2, 7.11.2. Stabil untuk L3VPN dan SR-MPLS.
- **NCS 5500 dan NCS 540:** Train 7.x dan 24.x tergantung hardware.
- **NCS 5700:** Train 24.x, contoh 24.1.1, 24.2.1, 24.4.1. Wajib untuk fitur SRv6 terbaru dan 800G.
- **XRv 9000:** Ikuti train yang sama dengan target produksi agar validation akurat.

Licensing di IOS-XR menggunakan model Essentials dan Advantage, plus add-on untuk fitur tertentu. Jangan asumsi semua fitur aktif. Sebelum design SRv6 atau EVPN, verification license di lab:

```
show license all
show license usage
show install active
```

### 1.5 Prinsip Design Core ISP

- **Loopback adalah identitas router:** Semua peering iBGP, LDP, SR, dan management menggunakan Loopback0. Jangan pernah menggunakan IP interface fisik untuk iBGP atau LDP.
- **IGP hanya untuk infrastructure:** IGP hanya mengiklankan Loopback dan link point-to-point. Prefix customer tidak pernah masuk IGP.
- **BGP adalah satu-satunya pembawa service:** Semua service customer dibawa BGP, baik VPNv4, VPNv6, maupun EVPN.
- **Hierarki RR:** PE hanya peer ke RR, bukan full mesh. RR berpasangan dan ditempatkan di lokasi yang berbeda.
- **Simetri dan determinisme:** Topology, metric, dan policy harus simetris agar troubleshooting dapat diprediksi.
- **Satu perubahan, satu commit:** Setiap commit harus memiliki label dan comment yang jelas.


## Modul 2: Workflow Configuration, Commit 2-Stage, Rollback, dan Operasional

### 2.1 Dua Mode yang Harus Dikuasai

Di IOS-XR ada dua dunia yang terpisah:

- **EXEC mode:** Untuk `show`, `ping`, `traceroute`, `clear`, dan monitoring. Prompt berakhir dengan `#`.
- **Configuration mode:** Untuk mengedit target config. Masuk dengan `configure`. Prompt berubah menjadi `(config)#`.

Contoh masuk dan keluar:

```
RP/0/RP0/CPU0:PE1# configure
RP/0/RP0/CPU0:PE1(config)# interface Loopback0
RP/0/RP0/CPU0:PE1(config-if)# ipv4 point-to-point
RP/0/RP0/CPU0:PE1(config-if)# ipv4 address 10.0.0.1/32
RP/0/RP0/CPU0:PE1(config-if)# commit
RP/0/RP0/CPU0:PE1(config-if)# end
```

### 2.2 Siklus Commit 2-Stage

1. **Edit:** Configuration diketik dan baru masuk target config, belum aktif.
2. **Validation:** Saat `commit`, IOS-XR memvalidasi sintaks, dependensi, dan semantic. Jika gagal, commit ditolak dan tidak ada perubahan parsial.
3. **Commit:** Jika valid, target config menjadi running config secara atomik.
4. **Audit:** Setiap commit memiliki ID, label, comment, dan user.

| Perintah | Fungsi |
| --- | --- |
| show configuration | Melihat perubahan yang belum committed |
| show configuration commit changes last 1 | Melihat diff commit terakhir |
| show configuration commit list | Melihat daftar commit, ID, user, dan waktu |
| show configuration history | Melihat histori commit lengkap |
| show configuration failed | Melihat kenapa commit terakhir gagal |
| commit | Commit perubahan |
| commit label <NAMA> | Commit dengan label untuk mudah dicari |
| commit comment <TEKS> | Commit dengan komentar audit |
| commit confirmed <MENIT> | Commit sementara, auto rollback jika tidak dikonfirmasi |
| commit replace | Ganti running config dengan target config secara penuh |
| rollback configuration last 1 | Kembali ke 1 commit sebelumnya, lalu `commit` |
| rollback configuration to <COMMIT-ID> | Kembali ke commit ID tertentu |
| abort | Hapus semua perubahan yang belum committed |

### 2.3 Contoh Workflow yang Benar

**Contoh 1: Edit Loopback dan commit dengan label**

```
configure
interface Loopback0
 ipv4 point-to-point
 ipv4 address 10.0.0.1/32
!
commit label LOOPBACK-PE1 comment Tambah Loopback PE1 untuk BGP dan LDP
end
```

**Contoh 2: Periksa diff sebelum commit**

```
configure
router ospf 100
 router-id 10.0.0.1
!
show configuration
commit
end
```

**Contoh 3: Commit confirmed untuk perubahan berisiko**

```
configure
interface HundredGigE0/0/0/0
 shutdown
!
commit confirmed 5
```

Jika masih dapat access setelah 5 menit, konfirmasi dengan `commit`. Jika kehilangan access, router akan rollback otomatis setelah 5 menit.

**Contoh 4: Rollback**

```
show configuration commit list
rollback configuration last 1
show configuration
commit label ROLLBACK-BGP comment Rollback perubahan BGP yang salah
```

**Contoh 5: Commit replace**

```
configure
load disk0:/golden-pe1.cfg
show configuration
commit replace label RESTORE-GOLDEN
```

### 2.4 Praktik Terbaik Operasi Carrier Grade

- **Satu commit untuk satu tujuan:** Jangan campur perubahan OSPF, BGP, dan VRF dalam satu commit.
- **Selalu gunakan label dan comment:** Format yang disarankan: `commit label CHG-1234-OSPF comment CHG 1234 tambah interface P1 ke OSPF area 0`.
- **Periksa diff sebelum commit:** `show configuration` adalah kebiasaan non-negosiasi.
- **Jangan gunakan `commit` di jam sibuk tanpa `commit confirmed`:** Untuk perubahan core, selalu ada jendela rollback.
- **Gunakan `configure exclusive` untuk perubahan kritis:** Ini mengunci configuration agar tidak ada engineer lain yang mengedit bersamaan.
- **Backup otomatis:** Setiap commit sebaiknya backed up ke server.

### 2.5 File, Log, dan Diagnosa Dasar

```
show running-config
show running-config interface HundredGigE0/0/0/0
show running-config router bgp 65000
show logging last 100
show processes cpu
show memory summary
show install active
show platform
show inventory all
show processes bgp
show processes ospf
```

### 2.6 Lab XRv 9000

Semua configuration di guide ini dapat divalidasi di XRv 9000. Topology lab minimal: CE1 - PE1 - P1 - P2 - PE2 - CE2, dengan RR1 dan RR2. Gunakan RAM minimal 8 GB per node untuk BGP dan SR. Jangan validation scale 10.000 VRF di XRv 9000, validation logika dan sintaksnya saja.


## Modul 3: Fondasi Transport, IGP, MPLS LDP, dan BGP Labeled Unicast

### 3.1 Design Addressing

Gunakan /31 untuk link point-to-point agar hemat IP dan konsisten. Untuk link yang mungkin berubah menjadi multi-access, gunakan /30.

### 3.2 OSPF versus IS-IS, Keputusan Arsitek

| Kriteria | OSPF | IS-IS |
| --- | --- | --- |
| Popularitas di ISP | Sangat umum, mudah direkrut engineer | Sangat umum di Tier-1, lebih scalable |
| Scale dan stabilitas | Baik sampai ribuan node dengan design area benar | Lebih baik untuk scale sangat besar, flooding lebih efisien |
| Support SR | Penuh, SR-MPLS dan SRv6 | Penuh, SR-MPLS dan SRv6, sering jadi pilihan utama untuk SR |
| Kemudahan troubleshooting | Mudah, banyak engineer paham | Membutuhkan pemahaman TLV dan level |
| Rekomendasi guide ini | Gunakan OSPF jika tim sudah kuat OSPF dan scale menengah | Gunakan IS-IS jika target adalah Tier-1, scale besar, dan SRv6 jangka panjang |

Guide ini menggunakan OSPF untuk Modul 3 agar mudah diikuti, lalu IS-IS diberikan sebagai alternatif yang setara. Prinsipnya sama: IGP hanya untuk Loopback dan link core.

### 3.3 OSPF di IOS-XR, Configuration Lengkap PE1

```
configure
interface Loopback0
 ipv4 point-to-point
 ipv4 address 10.0.0.1/32
!
interface HundredGigE0/0/0/0
 description TO-P1-Hu0/0/0/0
 ipv4 point-to-point
 ipv4 address 10.255.0.0/31
 no shutdown
!
router ospf 100
 router-id 10.0.0.1
 nsr
 nsf cisco
 timers throttle spf 50 100 5000
 timers throttle lsa all 50 100 5000
 area 0
 interface Loopback0
 passive enable
 !
 interface HundredGigE0/0/0/0
 network point-to-point
 hello-interval 3
 dead-interval 9
 bfd fast-detect
 !
 !
!
commit label OSPF-PE1 comment Aktifkan OSPF area 0 PE1
end
```

Penjelasan keputusan:

- **router-id:** Selalu Loopback0. Jangan biarkan OSPF memilih otomatis dari interface fisik.
- **nsr dan nsf cisco:** Untuk non-stop routing saat RP switchover. Wajib di core.
- **passive enable di Loopback0:** Loopback diiklankan, tetapi tidak mengirim hello.
- **network point-to-point:** Menghindari DR dan BDR di link point-to-point. Konvergensi lebih cepat.
- **hello 3, dead 9:** Lebih cepat dari default 10 dan 40, tetapi masih aman. Gabungkan dengan BFD untuk detection sub-detik.
- **bfd fast-detect:** Detection failure link dalam ratusan milidetik.

Configuration Bundle-Ether untuk core yang menggunakan LAG:

```
configure
interface Bundle-Ether1
 description TO-P1-BUNDLE
 ipv4 point-to-point
 ipv4 address 10.255.0.0/31
 bundle load-balancing hash src-dst-ip
!
interface HundredGigE0/0/0/0
 bundle id 1 mode active
!
interface HundredGigE0/0/0/1
 bundle id 1 mode active
!
router ospf 100
 area 0
 interface Bundle-Ether1
 network point-to-point
 hello-interval 3
 dead-interval 9
 bfd fast-detect
 !
 !
!
commit
end
```

### 3.4 IS-IS di IOS-XR, Alternatif Setara

```
configure
interface Loopback0
 ipv4 point-to-point
 ipv4 address 10.0.0.1/32
 ipv6 point-to-point
 ipv6 address 2001:db8::1/128
!
interface HundredGigE0/0/0/0
 ipv4 point-to-point
 ipv4 address 10.255.0.0/31
 ipv6 point-to-point
 ipv6 address 2001:db8:255::0/127
 no shutdown
!
router isis CORE
 is-type level-2-only
 net 49.0001.0000.0000.0001.00
 log adjacency changes
 nsf cisco
 nsr
 address-family ipv4 unicast
 metric-style wide
 router-id 10.0.0.1
 !
 address-family ipv6 unicast
 metric-style wide
 router-id 10.0.0.1
 !
 interface Loopback0
 passive
 address-family ipv4 unicast
 !
 address-family ipv6 unicast
 !
 !
 interface HundredGigE0/0/0/0
 point-to-point
 hello-interval 3
 hello-multiplier 3
 bfd fast-detect ipv4
 address-family ipv4 unicast
 metric 10
 !
 address-family ipv6 unicast
 metric 10
 !
 !
!
commit label ISIS-PE1 comment Aktifkan IS-IS Level 2 PE1
end
```

Keputusan arsitek untuk IS-IS: gunakan `level-2-only` untuk core flat. Jangan campur Level-1 dan Level-2 kecuali hierarki benar-benar dibutuhkan.

### 3.5 MPLS LDP, Transport MPLS Klasik

Configuration LDP di PE1:

```
configure
mpls ldp
 router-id 10.0.0.1
 nsr
 graceful-restart
 session protection
 interface HundredGigE0/0/0/0
 !
!
commit label LDP-PE1 comment Aktifkan MPLS LDP PE1
end
```

Configuration LDP di P1:

```
configure
mpls ldp
 router-id 10.0.0.11
 nsr
 graceful-restart
 session protection
 interface HundredGigE0/0/0/0
 !
 interface HundredGigE0/0/0/1
 !
!
commit
end
```

- **router-id:** Harus sama dengan Loopback0 dan router-id OSPF atau IS-IS.
- **session protection:** Menjaga session LDP tetap hidup meskipun interface utama down, selama masih ada path alternatif via IGP.
- **graceful-restart dan nsr:** Menjaga label forwarding tetap berjalan saat control plane restart.

### 3.6 BGP Labeled Unicast, Jembatan ke Segment Routing

BGP Labeled Unicast menggunakan BGP untuk distribution label Loopback. Ini jembatan konseptual ke SR-MPLS.

```
configure
route-policy PASS-ALL
 pass
end-policy
!
router bgp 65000
 bgp router-id 10.0.0.1
 address-family ipv4 labeled-unicast
 allocate-label all
 !
 neighbor 10.0.0.101
 remote-as 65000
 description RR1
 update-source Loopback0
 address-family ipv4 labeled-unicast
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 !
!
commit
end
```

Catatan penting IOS-XR: setiap neighbor BGP wajib memiliki `route-policy` in dan out. Tanpa route-policy, tidak ada route yang dipertukarkan. Di IOS-XR, default adalah deny.

### 3.7 BFD dan Fast Convergence

```
configure
bfd
 multiplier 3
 minimum-interval 300
!
router ospf 100
 area 0
 interface HundredGigE0/0/0/0
 bfd fast-detect
 !
 !
!
commit
end
```

Target design: detection failure link di bawah 1 detik, IGP reconverge di bawah 2 detik, dan FRR atau TI-LFA di bawah 50 milidetik untuk traffic yang diproteksi.

### 3.8 Verification Transport

```
show ospf neighbor
show ospf interface HundredGigE0/0/0/0
show route ipv4 10.0.0.2/32
show mpls ldp neighbor
show mpls ldp bindings 10.0.0.2/32
show mpls forwarding
show cef ipv4 10.0.0.2/32 detail
show bfd session
ping 10.0.0.2 source Loopback0
traceroute 10.0.0.2 source Loopback0
```

Jika `show mpls forwarding` tidak menampilkan label untuk Loopback PE tujuan, jangan lanjut ke Modul 4. Perbaiki IGP dan LDP dulu.


## Modul 4: BGP Core, Route Reflector, dan MPLS L3VPN End to End

### 4.1 Architecture BGP Core

Di ISP core, BGP memiliki dua peran yang harus dipisahkan:

- **BGP transport:** Mengiklankan Loopback jika menggunakan BGP-LU, atau tidak dipakai jika menggunakan LDP atau SR.
- **BGP service:** Mengiklankan VPNv4, VPNv6, dan EVPN.

PE tidak pernah peer langsung ke PE lain untuk service. PE hanya peer ke RR. RR hanya me-reflect route, tidak mem-forward traffic customer. RR harus redundant dan terpisah lokasi.

Topology BGP:

```
PE1 -- RR1 -- PE2
 | | |
PE1 -- RR2 -- PE2
```

Setiap PE memiliki 2 session iBGP ke RR1 dan RR2. RR1 dan RR2 peer satu sama lain. Tidak ada full mesh PE ke PE.

### 4.2 Design Route Reflector

| Keputusan | Rekomendasi | Alasan |
| --- | --- | --- |
| Jumlah RR | 2 per cluster, di POP berbeda | Satu RR down tidak mematikan service |
| Cluster ID | Berbeda untuk RR1 dan RR2 jika mereka bukan satu cluster | Mencegah loop dan mempermudah troubleshooting |
| Lokasi RR | Di POP core, dekat P, bukan di PE edge | RR tidak membutuhkan data plane besar, tetapi membutuhkan konektivitas IGP yang stabil |
| Jumlah PE per RR | 100 sampai 300 PE per pasang RR untuk design awal | Di atas itu, buat cluster baru atau hierarki RR |
| RR sebagai P atau standalone | Standalone lebih bersih untuk scale besar | RR standalone tidak terbebani forwarding traffic customer |

### 4.3 Configuration BGP Core di PE1

```
configure
route-policy PASS-ALL
 pass
end-policy
!
router bgp 65000
 bgp router-id 10.0.0.1
 bgp graceful-restart
 bgp bestpath as-path multipath-relax
 address-family ipv4 unicast
 redistribute connected route-policy CONNECTED-TO-BGP
 !
 neighbor 10.0.0.101
 remote-as 65000
 description RR1-CORE
 update-source Loopback0
 address-family ipv4 unicast
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 address-family vpnv4 unicast
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 !
 neighbor 10.0.0.102
 remote-as 65000
 description RR2-CORE
 update-source Loopback0
 address-family ipv4 unicast
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 address-family vpnv4 unicast
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 !
!
commit label BGP-PE1 comment Aktifkan BGP PE1 ke RR1 dan RR2
end
```

Configuration RR1:

```
configure
route-policy PASS-ALL
 pass
end-policy
!
router bgp 65000
 bgp router-id 10.0.0.101
 bgp graceful-restart
 bgp cluster-id 10.0.0.101
 address-family ipv4 unicast
 !
 address-family vpnv4 unicast
 !
 neighbor 10.0.0.1
 remote-as 65000
 description PE1
 update-source Loopback0
 address-family ipv4 unicast
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 address-family vpnv4 unicast
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 !
 neighbor 10.0.0.2
 remote-as 65000
 description PE2
 update-source Loopback0
 address-family ipv4 unicast
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 address-family vpnv4 unicast
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 !
 neighbor 10.0.0.102
 remote-as 65000
 description RR2
 update-source Loopback0
 address-family ipv4 unicast
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 address-family vpnv4 unicast
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 !
!
commit label BGP-RR1 comment Aktifkan RR1
end
```

Di IOS-XR, RR aktif secara otomatis ketika neighbor adalah iBGP dan RR mengiklankan route yang dipelajari dari satu client ke client lain. Perintah `route-reflector-client` seperti di IOS tidak diperlukan. Yang membedakan RR dari PE biasa adalah `bgp cluster-id` dan desainnya sebagai RR.

### 4.4 Konsep RD, RT, dan VRF

- **VRF:** Tabel routing virtual per customer di PE. VRF memisahkan routing customer A dari customer B meskipun prefix-nya sama.
- **RD atau Route Distinguisher:** Membuat prefix IPv4 yang sama menjadi unik di BGP dengan menambahkan prefix 64-bit. Contoh: 192.168.1.0/24 milik customer A menjadi 65000:100:192.168.1.0/24. RD tidak menentukan siapa yang boleh menerima route, hanya membuat route unik.
- **RT atau Route Target:** Menentukan siapa yang boleh mengimpor route. RT adalah mekanisme policy.

| Customer | RD | RT Import dan Export |
| --- | --- | --- |
| CUSTOMER-A | 65000:100 | 65000:100 |
| CUSTOMER-B | 65000:200 | 65000:200 |
| CUSTOMER Hub and Spoke | Hub RD 65000:300, Spoke RD 65000:301 | Hub import 65000:301, export 65000:300. Spoke import 65000:300, export 65000:301 |

Gunakan RD yang unik per VRF per PE untuk troubleshooting yang mudah. Gunakan format AS:NN untuk RD dan RT.

### 4.5 L3VPN End to End, Configuration PE1

Langkah 1: Buat VRF di PE1.

```
configure
vrf CUSTOMER-A
 rd 65000:100
 address-family ipv4 unicast
 import route-target 65000:100
 export route-target 65000:100
 !
!
commit label VRF-CUSTOMER-A comment Buat VRF CUSTOMER-A
end
```

Langkah 2: Assign interface PE-CE ke VRF.

```
configure
interface GigabitEthernet0/0/0/1
 description TO-CE1-CUSTOMER-A
 vrf CUSTOMER-A
 ipv4 point-to-point
 ipv4 address 192.168.1.1/30
 no shutdown
!
commit
end
```

Langkah 3: Configuration BGP VRF untuk PE-CE dengan eBGP.

```
configure
router bgp 65000
 vrf CUSTOMER-A
 rd 65000:100
 address-family ipv4 unicast
 label mode per-vrf
 redistribute connected
 redistribute static
 !
 neighbor 192.168.1.2
 remote-as 65001
 description CE1-CUSTOMER-A
 address-family ipv4 unicast
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 !
 !
!
commit label BGP-VRF-CUSTOMER-A comment Aktifkan BGP VRF CUSTOMER-A PE1
end
```

- **label mode per-vrf:** Satu label untuk seluruh VRF, bukan per prefix. Ini menghemat label dan menyederhanakan forwarding.
- **rd di BGP VRF:** Harus sama dengan RD di definisi VRF global.

Langkah 4: Ulangi di PE2.

```
configure
vrf CUSTOMER-A
 rd 65000:100
 address-family ipv4 unicast
 import route-target 65000:100
 export route-target 65000:100
 !
!
interface GigabitEthernet0/0/0/1
 description TO-CE2-CUSTOMER-A
 vrf CUSTOMER-A
 ipv4 point-to-point
 ipv4 address 192.168.2.1/30
 no shutdown
!
router bgp 65000
 vrf CUSTOMER-A
 rd 65000:100
 address-family ipv4 unicast
 label mode per-vrf
 redistribute connected
 redistribute static
 !
 neighbor 192.168.2.2
 remote-as 65002
 description CE2-CUSTOMER-A
 address-family ipv4 unicast
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 !
 !
!
commit label L3VPN-PE2 comment Aktifkan L3VPN CUSTOMER-A di PE2
end
```

### 4.6 PE-CE Routing, Pilihan Lain Selain eBGP

**Opsi 1: Static routing**

```
configure
router static
 vrf CUSTOMER-A
 address-family ipv4 unicast
 192.168.10.0/24 192.168.1.2
 !
 !
!
router bgp 65000
 vrf CUSTOMER-A
 address-family ipv4 unicast
 redistribute static
 !
 !
!
commit
end
```

**Opsi 2: OSPF PE-CE**

```
configure
router ospf 200
 vrf CUSTOMER-A
 router-id 192.168.1.1
 area 0
 interface GigabitEthernet0/0/0/1
 network point-to-point
 !
 !
 !
!
router bgp 65000
 vrf CUSTOMER-A
 address-family ipv4 unicast
 redistribute ospf 200
 !
 !
!
commit
end
```

Perhatian: OSPF PE-CE harus menggunakan process ID yang berbeda dari OSPF core. Jangan pernah mencampur OSPF customer dengan OSPF core di process yang sama.

**Opsi 3: eBGP seperti contoh utama**

Ini yang paling direkomendasikan untuk ISP karena isolasi, policy yang jelas, dan troubleshooting yang mudah.

### 4.7 Verification L3VPN

Di PE1:

```
show vrf CUSTOMER-A detail
show route vrf CUSTOMER-A ipv4
show bgp vrf CUSTOMER-A ipv4 unicast summary
show bgp vrf CUSTOMER-A ipv4 unicast neighbors 192.168.1.2
show bgp vpnv4 unicast rd 65000:100
show bgp vpnv4 unicast vrf CUSTOMER-A
show mpls forwarding vrf CUSTOMER-A
show cef vrf CUSTOMER-A ipv4 192.168.2.0/24 detail
ping vrf CUSTOMER-A 192.168.2.2 source 192.168.1.1
traceroute vrf CUSTOMER-A 192.168.2.2 source 192.168.1.1
```

Di RR1:

```
show bgp vpnv4 unicast summary
show bgp vpnv4 unicast rd 65000:100
show bgp vpnv4 unicast neighbors 10.0.0.1
```

Di P1:

```
show mpls forwarding
show mpls ldp neighbor
```

Di CE1:

```
ping 192.168.2.2 source 192.168.1.2
traceroute 192.168.2.2 source 192.168.1.2
```

Urutan troubleshooting jika ping gagal: 1) Periksa PE-CE, 2) Periksa VPNv4 di PE1, 3) Periksa RR, 4) Periksa PE2 impor RT, 5) Periksa LSP, 6) Periksa CE2 return route.

### 4.8 Anti-Pattern L3VPN

- **RT salah ketik:** Satu digit RT yang salah membuat route tidak pernah diimpor. Tidak ada error, hanya route yang hilang.
- **RD sama untuk semua VRF:** Membuat troubleshooting sulit.
- **VRF tidak assigned ke interface:** Periksa `show running-config interface GigabitEthernet0/0/0/1`.
- **Lupa route-policy di BGP:** Di IOS-XR, tanpa route-policy, BGP tidak akan bertukar route.
- **Label mode tidak konsisten:** Standarkan per-vrf.
- **Redistribute tanpa filter:** Selalu gunakan route-policy dengan prefix-set yang jelas.


## Modul 5: Segment Routing MPLS, SR Policy, dan TI-LFA

### 5.1 Kenapa Tinggalkan LDP

- **State per LSP:** Setiap router harus menjaga session LDP dan label binding untuk setiap FEC.
- **Tidak ada Traffic Engineering native:** LDP selalu mengikuti IGP shortest path. Untuk TE, RSVP-TE yang stateful dan rumit perlu ditambahkan.
- **Troubleshooting tidak deterministik:** Label LDP dialokasikan dinamis.

Segment Routing membalik logika ini. Label bukan lagi dialokasikan per hop secara dinamis, tetapi dihitung dari SID yang deterministik.

### 5.2 Konsep SR-MPLS

- **SRGB atau Segment Routing Global Block:** Range label global untuk Prefix SID. Contoh: 16000 sampai 23999. SRGB harus sama di semua router dalam satu SR domain.
- **Prefix SID:** SID untuk Loopback sebuah node. Dihitung dari SRGB ditambah index. Jika SRGB dimulai di 16000 dan index PE1 adalah 1, maka SID PE1 adalah 16001.
- **Adjacency SID atau Adj-SID:** SID untuk link spesifik. Dipakai untuk steering eksplisit.
- **Anycast SID:** SID yang sama di beberapa node. Dipakai untuk anycast service.
- **Node SID:** Istilah lain untuk Prefix SID dari Loopback.

| Node | Loopback0 | Prefix SID Index | SID atau Label |
| --- | --- | --- | --- |
| PE1 | 10.0.0.1/32 | 1 | 16001 |
| PE2 | 10.0.0.2/32 | 2 | 16002 |
| P1 | 10.0.0.11/32 | 11 | 16011 |
| P2 | 10.0.0.12/32 | 12 | 16012 |
| RR1 | 10.0.0.101/32 | 101 | 16101 |
| RR2 | 10.0.0.102/32 | 102 | 16102 |

### 5.3 Design SRGB dan Prefix SID

- **SRGB harus direncanakan sekali dan tidak diubah.**
- **Sisakan ruang untuk LDP selama migrasi.**
- **Index harus unik dan terstruktur:** PE 1 sampai 99, P 11 sampai 99, RR 101 sampai 199.
- **Jangan gunakan index 0.**

```
configure
segment-routing
 global-block 16000 23999
!
mpls label range table 0 24000 1048575
!
commit label SRGB-PE1 comment Set SRGB 16000 sampai 23999 PE1
end
```

### 5.4 Configuration SR-MPLS dengan OSPF

Configuration PE1 dengan OSPF dan SR-MPLS:

```
configure
segment-routing
 global-block 16000 23999
!
router ospf 100
 router-id 10.0.0.1
 segment-routing mpls
 segment-routing forwarding mpls
 area 0
 interface Loopback0
 passive enable
 prefix-sid index 1
 !
 interface HundredGigE0/0/0/0
 network point-to-point
 hello-interval 3
 dead-interval 9
 bfd fast-detect
 fast-reroute per-prefix
 fast-reroute per-prefix ti-lfa
 !
 !
!
commit label SR-MPLS-OSPF-PE1 comment Aktifkan SR-MPLS OSPF PE1
end
```

Configuration P1:

```
configure
segment-routing
 global-block 16000 23999
!
router ospf 100
 router-id 10.0.0.11
 segment-routing mpls
 segment-routing forwarding mpls
 area 0
 interface Loopback0
 passive enable
 prefix-sid index 11
 !
 interface HundredGigE0/0/0/0
 network point-to-point
 fast-reroute per-prefix
 fast-reroute per-prefix ti-lfa
 !
 interface HundredGigE0/0/0/1
 network point-to-point
 fast-reroute per-prefix
 fast-reroute per-prefix ti-lfa
 !
 !
!
commit label SR-MPLS-OSPF-P1 comment Aktifkan SR-MPLS OSPF P1
end
```

Jika masih menjalankan LDP bersamaan, tambahkan preferensi SR:

```
configure
router ospf 100
 segment-routing mpls sr-prefer
!
commit
end
```

### 5.5 Configuration SR-MPLS dengan IS-IS

```
configure
segment-routing
 global-block 16000 23999
!
router isis CORE
 is-type level-2-only
 net 49.0001.0000.0000.0001.00
 segment-routing mpls
 address-family ipv4 unicast
 metric-style wide
 segment-routing mpls
 router-id 10.0.0.1
 !
 interface Loopback0
 passive
 address-family ipv4 unicast
 prefix-sid index 1
 !
 !
 interface HundredGigE0/0/0/0
 point-to-point
 address-family ipv4 unicast
 metric 10
 fast-reroute per-prefix
 fast-reroute per-prefix ti-lfa
 !
 !
!
commit label SR-MPLS-ISIS-PE1 comment Aktifkan SR-MPLS IS-IS PE1
end
```

### 5.6 Adj-SID dan Anycast SID

Adj-SID dipakai untuk SR Policy yang membutuhkan steering per link. Di IOS-XR, Adj-SID dialokasikan otomatis dan dapat dilihat, tidak perlu dikonfigurasi manual untuk kasus dasar.

Anycast SID dipakai untuk dua PE yang menjadi satu service anycast:

```
configure
router ospf 100
 area 0
 interface Loopback1
 passive enable
 prefix-sid index 500
 !
 !
!
interface Loopback1
 ipv4 point-to-point
 ipv4 address 10.0.0.200/32
!
commit
end
```

### 5.7 SR Policy, Traffic Engineering Tanpa RSVP-TE

Konsep SR Policy:

- **Color:** Identifier policy, misalnya 100 untuk low latency, 200 untuk disjoint, 300 untuk bandwidth.
- **Endpoint:** Loopback tujuan, misalnya PE2 10.0.0.2.
- **Candidate Path:** Daftar path yang mungkin, dengan preference.
- **Segment List:** Daftar SID yang harus dilewati.

Contoh 1: SR Policy eksplisit dari PE1 ke PE2 lewat P1.

```
configure
segment-routing
 traffic-eng
 segment-list SL-PE1-PE2-VIA-P1
 index 10 mpls label 16011
 index 20 mpls label 16002
 !
 policy POL-PE1-TO-PE2-100
 color 100 end-point ipv4 10.0.0.2
 candidate-paths
 preference 100
 explicit segment-list SL-PE1-PE2-VIA-P1
 !
 !
 !
 !
!
commit label SR-POLICY-PE1 comment Buat SR Policy PE1 ke PE2 color 100
end
```

Contoh 2: SR Policy dinamis dengan constraint IGP.

```
configure
segment-routing
 traffic-eng
 policy POL-PE1-TO-PE2-200
 color 200 end-point ipv4 10.0.0.2
 candidate-paths
 preference 100
 dynamic
 metric
 type igp
 !
 !
 !
 !
 !
 !
!
commit
end
```

Contoh 3: SR Policy untuk L3VPN, steering VRF tertentu ke policy tertentu.

```
configure
segment-routing
 traffic-eng
 policy POL-CUSTOMER-A-LOWLAT
 color 100 end-point ipv4 10.0.0.2
 candidate-paths
 preference 100
 explicit segment-list SL-PE1-PE2-VIA-P1
 !
 !
 !
 !
!
router bgp 65000
 vrf CUSTOMER-A
 address-family ipv4 unicast
 segment-routing policy color 100
 !
 !
!
commit
end
```

Contoh 4: SR Policy dengan disjoint dan protection.

```
configure
segment-routing
 traffic-eng
 policy POL-PE1-TO-PE2-DISJOINT
 color 300 end-point ipv4 10.0.0.2
 candidate-paths
 preference 200
 explicit segment-list SL-PE1-PE2-VIA-P1
 !
 preference 100
 explicit segment-list SL-PE1-PE2-VIA-P2
 !
 !
 !
 !
!
commit
end
```

### 5.8 TI-LFA, Fast Reroute di Bawah 50 Milidetik

TI-LFA menghitung backup path yang loop-free untuk setiap prefix, bahkan ketika LFA klasik tidak menemukan backup yang valid. Saat link utama gagal, router langsung mendorong label repair ke traffic tanpa menunggu IGP reconverge.

```
show mpls forwarding
show ospf fast-reroute
show isis fast-reroute
show segment-routing traffic-eng policy
show cef ipv4 10.0.0.2/32 detail
```

Design TI-LFA yang benar: topology harus memiliki redundansi fisik, metric harus simetris, uji failover di lab dan ukur loss. Target: loss di bawah 50 milidetik untuk traffic yang diproteksi TI-LFA.

### 5.9 Migrasi LDP ke SR-MPLS Tanpa Downtime

**Tahap 1:** Aktifkan SR berdampingan dengan LDP.
**Tahap 2:** Aktifkan `sr-prefer`. Periksa `show mpls forwarding` dan pastikan label yang dipakai sekarang adalah label SR.
**Tahap 3:** Validation L3VPN di atas SR.
**Tahap 4:** Nonaktifkan LDP per node, mulai dari P.

```
configure
no mpls ldp
!
commit label REMOVE-LDP-P1 comment Nonaktifkan LDP P1 setelah SR stabil
end
```

**Tahap 5:** Bersihkan configuration LDP dari template golden config setelah stabil satu minggu.

### 5.10 Verification SR-MPLS

```
show segment-routing mpls mapping-server
show ospf segment-routing
show isis segment-routing label table
show mpls forwarding
show cef ipv4 10.0.0.2/32 detail
show segment-routing traffic-eng policy all
show segment-routing traffic-eng policy color 100 end-point ipv4 10.0.0.2 detail
show bgp vpnv4 unicast summary
ping vrf CUSTOMER-A 192.168.2.2 source 192.168.1.1
```


## Modul 6: SRv6, Micro-SID, dan Migrasi dari SR-MPLS

### 6.1 Kenapa SRv6

- **Tidak ada MPLS di core:** Forwarding menggunakan IPv6 native.
- **Programmability:** SID SRv6 dapat membawa fungsi, bukan hanya lokasi. Contoh: End.DT4 untuk decapsulasi ke VRF IPv4, End.DT6 untuk VRF IPv6, End.DX4 untuk cross-connect ke CE.
- **Scale dan otomasi:** SRv6 terintegrasi dengan IPv6 dan dapat dipakai untuk network slicing dan 5G.
- **Masa depan untuk NCS 5700 dan ASR 9000 generasi baru.**

Keputusan arsitek: jangan migrasi ke SRv6 hanya karena tren. Migrasi ke SRv6 jika core sudah dual-stack IPv6, hardware sudah mendukung SRv6 di line rate, dan ada use case yang membutuhkan programmability seperti network slicing atau 5G.

### 6.2 Konsep SRv6, uSID, dan Micro-SID

- **SID SRv6:** Alamat IPv6 yang mewakili instruksi.
- **Locator:** Prefix IPv6 yang menampung semua SID di sebuah domain. Contoh: 2001:db8:100::/48.
- **uSID atau Micro-SID:** Kompresi SID. Satu alamat IPv6 128-bit dapat membawa beberapa Micro-SID 16-bit atau 32-bit.
- **Behavior:** Fungsi yang dijalankan saat SID diterima. Contoh: uN atau End, uDT4 atau End.DT4, uDT6 atau End.DT6, uDX4.

| Bagian | Panjang | Contoh |
| --- | --- | --- |
| Locator Block | 32-bit | 2001:db8:100:: |
| Node ID | 16-bit | 0001 untuk PE1, 0002 untuk PE2, 000b untuk P1 |
| Function | 16-bit | 0000 untuk End, e000 untuk DT4 |
| Sisa | 64-bit | Nol atau argumen |

| Node | uSID | Arti |
| --- | --- | --- |
| PE1 | 2001:db8:100:1:: | Menuju PE1 |
| PE2 | 2001:db8:100:2:: | Menuju PE2 |
| P1 | 2001:db8:100:b:: | Menuju P1 |
| P2 | 2001:db8:100:c:: | Menuju P2 |

### 6.3 Design Locator

- **Satu locator untuk satu domain:** Jangan buat locator per POP. Satu locator untuk seluruh SRv6 domain, misalnya 2001:db8:100::/48.
- **Locator harus routable via IGP.**
- **Pisahkan locator untuk fungsi yang berbeda jika perlu.**
- **Alokasikan IPv6 core dengan benar:** Link core menggunakan /127, Loopback menggunakan /128, Locator menggunakan /48.

| Node | Loopback IPv6 | Locator SID |
| --- | --- | --- |
| PE1 | 2001:db8::1/128 | 2001:db8:100:1:: |
| PE2 | 2001:db8::2/128 | 2001:db8:100:2:: |
| P1 | 2001:db8::11/128 | 2001:db8:100:b:: |
| P2 | 2001:db8::12/128 | 2001:db8:100:c:: |
| RR1 | 2001:db8::101/128 | 2001:db8:100:65:: |
| RR2 | 2001:db8::102/128 | 2001:db8:100:66:: |

| Link | IPv6 |
| --- | --- |
| PE1 ke P1 | 2001:db8:255::0/127 |
| P1 ke P2 | 2001:db8:255::2/127 |
| P2 ke PE2 | 2001:db8:255::4/127 |

### 6.4 Configuration SRv6 dengan IS-IS

Langkah 1: Configuration locator SRv6 di PE1.

```
configure
segment-routing
 srv6
 locators
 locator MAIN
 micro-segment behavior unode psp-usd
 prefix 2001:db8:100::/48
 !
 !
 !
!
commit label SRV6-LOCATOR-PE1 comment Buat locator SRv6 MAIN PE1
end
```

`micro-segment behavior unode psp-usd` berarti locator ini menggunakan uSID dengan mode PSP dan USD.

Langkah 2: Aktifkan IPv6 di interface dan IS-IS SRv6.

```
configure
interface Loopback0
 ipv4 point-to-point
 ipv4 address 10.0.0.1/32
 ipv6 point-to-point
 ipv6 address 2001:db8::1/128
!
interface HundredGigE0/0/0/0
 ipv4 point-to-point
 ipv4 address 10.255.0.0/31
 ipv6 point-to-point
 ipv6 address 2001:db8:255::0/127
 no shutdown
!
router isis CORE
 is-type level-2-only
 net 49.0001.0000.0000.0001.00
 address-family ipv6 unicast
 metric-style wide
 segment-routing srv6
 locator MAIN
 !
 !
 address-family ipv4 unicast
 metric-style wide
 router-id 10.0.0.1
 !
 interface Loopback0
 passive
 address-family ipv6 unicast
 !
 address-family ipv4 unicast
 !
 !
 interface HundredGigE0/0/0/0
 point-to-point
 address-family ipv6 unicast
 metric 10
 !
 address-family ipv4 unicast
 metric 10
 !
 !
!
commit label SRV6-ISIS-PE1 comment Aktifkan SRv6 IS-IS PE1
end
```

### 6.5 SRv6 L3VPN, End.DT4 dan End.DT6

```
configure
segment-routing
 srv6
 locators
 locator MAIN
 micro-segment behavior unode psp-usd
 prefix 2001:db8:100::/48
 !
 !
 !
!
router bgp 65000
 bgp router-id 10.0.0.1
 segment-routing srv6
 locator MAIN
 allocation mode per-vrf
 !
 neighbor 10.0.0.101
 remote-as 65000
 description RR1
 update-source Loopback0
 address-family vpnv4 unicast
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 address-family vpnv6 unicast
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 !
 vrf CUSTOMER-A
 rd 65000:100
 address-family ipv4 unicast
 segment-routing srv6
 locator MAIN
 alloc mode per-vrf
 !
 redistribute connected
 !
 address-family ipv6 unicast
 segment-routing srv6
 locator MAIN
 alloc mode per-vrf
 !
 redistribute connected
 !
 !
!
commit label SRV6-L3VPN-PE1 comment Aktifkan SRv6 L3VPN CUSTOMER-A PE1
end
```

- **allocation mode per-vrf:** Satu SID SRv6 untuk seluruh VRF. SID ini akan menjadi End.DT4 untuk IPv4 dan End.DT6 untuk IPv6.
- **segment-routing srv6 locator MAIN di BGP:** Memberikan tahu BGP untuk mengalokasikan Service SID dari locator MAIN.

Configuration VRF di PE1 tetap sama seperti Modul 4, hanya ditambah SRv6:

```
configure
vrf CUSTOMER-A
 rd 65000:100
 address-family ipv4 unicast
 import route-target 65000:100
 export route-target 65000:100
 !
 address-family ipv6 unicast
 import route-target 65000:100
 export route-target 65000:100
 !
!
commit
end
```

### 6.6 SRv6 EVPN dan SR Policy SRv6

```
configure
router bgp 65000
 segment-routing srv6
 locator MAIN
 !
 neighbor 10.0.0.101
 address-family l2vpn evpn
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 !
 evpn
 evi 100
 bgp
 route-target import 65000:100
 route-target export 65000:100
 !
 !
 !
!
commit
end
```

SR Policy SRv6:

```
configure
segment-routing
 traffic-eng
 srv6
 segment-list SL-SRV6-PE1-PE2
 index 10 ipv6 address 2001:db8:100:b::
 index 20 ipv6 address 2001:db8:100:2::
 !
 policy POL-SRV6-PE1-PE2
 color 100 end-point ipv6 2001:db8::2
 candidate-paths
 preference 100
 explicit segment-list SL-SRV6-PE1-PE2
 !
 !
 !
 !
 !
!
commit
end
```

### 6.7 Migrasi SR-MPLS ke SRv6

**Tahap 1:** Dual-stack IGP. Aktifkan IPv6 di semua link core dan di IS-IS.
**Tahap 2:** Aktifkan SRv6 berdampingan dengan SR-MPLS.
**Tahap 3:** Migrasi VRF per customer. Pindahkan satu VRF percobaan ke SRv6.
**Tahap 4:** Migrasi SR Policy dari MPLS ke SRv6 per color.
**Tahap 5:** Nonaktifkan SR-MPLS setelah semua VRF dan policy sudah di SRv6 dan stabil.

### 6.8 Verification SRv6

```
show segment-routing srv6 locator MAIN detail
show segment-routing srv6 sid
show isis segment-routing srv6
show bgp vpnv4 unicast summary
show bgp vpnv4 unicast rd 65000:100
show bgp vpnv4 unicast vrf CUSTOMER-A
show cef ipv6 2001:db8:100:2:: detail
show segment-routing srv6 manager
ping vrf CUSTOMER-A 192.168.2.2 source 192.168.1.1
ping vrf CUSTOMER-A 2001:db8:192:2::2 source 2001:db8:192:1::1
```

## Modul 7: EVPN VPWS, VPLS, dan EVPN E-LAN

### 7.1 EVPN versus VPLS dan VPWS Klasik

EVPN menyelesaikan tiga masalah VPLS klasik dengan BGP sebagai control plane:

- **MAC learning via BGP:** MAC address diiklankan via BGP EVPN Route Type 2.
- **Multihoming native:** ESI memungkinkan dua PE melayani satu CE dengan All-Active atau Single-Active tanpa loop.
- **Satu control plane untuk L2 dan L3:** EVPN yang sama dapat membawa L2 dan L3.

| Aspek | VPLS LDP | VPWS LDP | EVPN |
| --- | --- | --- | --- |
| Control plane | LDP | LDP | BGP EVPN |
| MAC learning | Data plane flood | Tidak ada MAC, point-to-point | BGP Route Type 2 |
| Multihoming | Sulit, membutuhkan MC-LAG | Sulit | Native via ESI |
| Scale | Terbatas | Terbatas | Sangat scalable |
| Interworking L3 | Sulit | Tidak ada | Native via RT-5 |

### 7.2 Konsep EVPN

- **EVI atau EVPN Instance:** Identifier untuk service EVPN. Satu EVI untuk satu service, misalnya EVI 100 untuk CUSTOMER-A.
- **ESI atau Ethernet Segment Identifier:** Identifier untuk link multihoming.
- **Route Target:** Sama seperti L3VPN, RT menentukan siapa yang boleh mengimpor route EVPN.

| Route Type | Nama | Fungsi |
| --- | --- | --- |
| 1 | Ethernet Auto-Discovery | Mengiklankan ESI dan multihoming |
| 2 | MAC dan IP Advertisement | Mengiklankan MAC dan IP host |
| 3 | Inclusive Multicast | Untuk BUM traffic, broadcast, unknown unicast, multicast |
| 4 | Ethernet Segment | Untuk Designated Forwarder election |
| 5 | IP Prefix | Untuk EVPN L3VPN, dibahas di Modul 8 |

### 7.3 Fondasi BGP EVPN

Configuration di PE1:

```
configure
route-policy PASS-ALL
 pass
end-policy
!
router bgp 65000
 bgp router-id 10.0.0.1
 neighbor 10.0.0.101
 remote-as 65000
 description RR1
 update-source Loopback0
 address-family l2vpn evpn
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 !
 neighbor 10.0.0.102
 remote-as 65000
 description RR2
 update-source Loopback0
 address-family l2vpn evpn
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 !
!
evpn
 evi 100
 bgp
 route-target import 65000:100
 route-target export 65000:100
 !
 !
!
commit label EVPN-BGP-PE1 comment Aktifkan BGP EVPN PE1
end
```

### 7.4 EVPN VPWS, Point-to-Point L2

Configuration di PE1:

```
configure
interface GigabitEthernet0/0/0/2
 description TO-CE1-EVPN-VPWS
 l2transport
 encapsulation dot1q 100
 rewrite ingress tag pop 1 symmetric
!
l2vpn
 xconnect group VPWS-CUSTOMER-A
 p2p VPWS-100
 interface GigabitEthernet0/0/0/2
 neighbor evpn evi 100 target 1 source 1
 !
 !
!
evpn
 evi 100
 bgp
 route-target import 65000:100
 route-target export 65000:100
 !
 !
!
commit label EVPN-VPWS-PE1 comment Aktifkan EVPN VPWS PE1
end
```

Configuration di PE2:

```
configure
interface GigabitEthernet0/0/0/2
 description TO-CE2-EVPN-VPWS
 l2transport
 encapsulation dot1q 100
 rewrite ingress tag pop 1 symmetric
!
l2vpn
 xconnect group VPWS-CUSTOMER-A
 p2p VPWS-100
 interface GigabitEthernet0/0/0/2
 neighbor evpn evi 100 target 2 source 2
 !
 !
!
evpn
 evi 100
 bgp
 route-target import 65000:100
 route-target export 65000:100
 !
 !
!
commit label EVPN-VPWS-PE2 comment Aktifkan EVPN VPWS PE2
end
```

Verification VPWS:

```
show l2vpn xconnect group VPWS-CUSTOMER-A
show evpn evi 100 detail
show bgp l2vpn evpn evi 100
show l2vpn xconnect detail
```

### 7.5 EVPN E-LAN, VPLS Modern

Configuration di PE1:

```
configure
interface GigabitEthernet0/0/0/3
 description TO-CE1-EVPN-ELAN
 l2transport
 encapsulation dot1q 200
 rewrite ingress tag pop 1 symmetric
!
l2vpn
 bridge group BG-CUSTOMER-A
 bridge-domain BD-200
 interface GigabitEthernet0/0/0/3
 !
 evi 200
 !
 !
!
evpn
 evi 200
 bgp
 route-target import 65000:200
 route-target export 65000:200
 !
 !
!
commit label EVPN-ELAN-PE1 comment Aktifkan EVPN E-LAN PE1
end
```

Configuration di PE2 sama, hanya interface dan CE yang berbeda, tetapi EVI dan bridge-domain sama.

Perbedaan VPWS dan E-LAN: VPWS point-to-point, satu CE ke satu CE. E-LAN multipoint, banyak CE dalam satu bridge domain, MAC learning via BGP Route Type 2.

### 7.6 Multihoming ESI, Single-Active dan All-Active

Configuration ESI di PE1:

```
configure
interface Bundle-Ether10
 description TO-CE1-MULTIHOMING
 l2transport
 encapsulation dot1q 200
 rewrite ingress tag pop 1 symmetric
!
evpn
 interface Bundle-Ether10
 ethernet-segment
 identifier type 0 00.00.00.00.00.01.00.00.00.01
 bgp route-target 65000:200
 !
 !
!
l2vpn
 bridge group BG-CUSTOMER-A
 bridge-domain BD-200
 interface Bundle-Ether10
 !
 evi 200
 !
 !
!
commit label EVPN-ESI-PE1 comment Aktifkan ESI multihoming PE1
end
```

Configuration di PE2 harus menggunakan ESI yang sama persis. Mode multihoming:

- **Single-Active:** Hanya satu PE yang aktif forwarding untuk VLAN tertentu.
- **All-Active:** Kedua PE aktif forwarding. Traffic load-balanced.

```
evpn
 interface Bundle-Ether10
 ethernet-segment
 identifier type 0 00.00.00.00.00.01.00.00.00.01
 bgp route-target 65000:200
 load-balancing-mode all-active
 !
 !
!
```

Verification ESI:

```
show evpn ethernet-segment
show evpn evi 200 detail
show bgp l2vpn evpn route-type 1
show bgp l2vpn evpn route-type 4
```

### 7.7 EVPN dengan IRB dan Anycast Gateway

```
configure
interface BVI200
 description GATEWAY-CUSTOMER-A-VLAN200
 vrf CUSTOMER-A
 ipv4 point-to-point
 ipv4 address 192.168.200.1/24
 ipv6 point-to-point
 ipv6 address 2001:db8:200::1/64
!
l2vpn
 bridge group BG-CUSTOMER-A
 bridge-domain BD-200
 interface GigabitEthernet0/0/0/3
 !
 routed interface BVI200
 evi 200
 !
 !
!
router bgp 65000
 vrf CUSTOMER-A
 address-family ipv4 unicast
 redistribute connected
 !
 !
!
commit label EVPN-IRB-PE1 comment Aktifkan IRB BVI200 PE1
end
```

Anycast Gateway berarti PE1 dan PE2 memiliki IP gateway yang sama untuk VLAN yang sama. CE tidak perlu tahu PE mana yang menjadi gateway.

### 7.8 Verification EVPN

```
show evpn evi
show evpn evi 100 detail
show evpn ethernet-segment
show bgp l2vpn evpn summary
show bgp l2vpn evpn evi 100
show bgp l2vpn evpn route-type 2
show bgp l2vpn evpn route-type 3
show l2vpn bridge-domain group BG-CUSTOMER-A
show l2vpn xconnect group VPWS-CUSTOMER-A
show l2vpn forwarding bridge-domain BG-CUSTOMER-A:BD-200 mac-address location 0/0/CPU0
show arp vrf CUSTOMER-A
```

## Modul 8: EVPN L3VPN, Interworking L3VPN dan EVPN

### 8.1 EVPN L3VPN versus MPLS L3VPN

| Aspek | MPLS L3VPN VPNv4 | EVPN L3VPN RT-5 |
| --- | --- | --- |
| Address family | vpnv4 unicast, vpnv6 unicast | l2vpn evpn dengan Route Type 5 |
| Prefix yang dibawa | Prefix IP | Prefix IP plus MAC dan gateway info |
| Interworking dengan L2 | Membutuhkan IRB terpisah | Native, satu EVPN untuk L2 dan L3 |
| Scale | Sangat baik | Sangat baik, plus fleksibilitas DCI |
| Use case utama | L3VPN klasik, ISP core | DCI, data center, dan core yang membutuhkan L2 dan L3 dalam satu control plane |

### 8.2 Architecture EVPN RT-5

EVPN Route Type 5 membawa prefix IP seperti 192.168.10.0/24. Prefix ini diiklankan dari PE ke RR via BGP EVPN, sama seperti VPNv4, tetapi dengan informasi gateway dan MAC yang lebih kaya.

### 8.3 Configuration EVPN L3VPN

Configuration EVPN L3VPN di PE1 untuk CUSTOMER-A:

```
configure
vrf CUSTOMER-A
 rd 65000:100
 address-family ipv4 unicast
 import route-target 65000:100
 export route-target 65000:100
 import route-target 65000:100 evpn
 export route-target 65000:100 evpn
 !
 address-family ipv6 unicast
 import route-target 65000:100
 export route-target 65000:100
 import route-target 65000:100 evpn
 export route-target 65000:100 evpn
 !
!
router bgp 65000
 bgp router-id 10.0.0.1
 neighbor 10.0.0.101
 address-family l2vpn evpn
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 !
 vrf CUSTOMER-A
 rd 65000:100
 address-family ipv4 unicast
 label mode per-vrf
 redistribute connected
 redistribute static
 !
 address-family ipv6 unicast
 label mode per-vrf
 redistribute connected
 !
 !
!
evpn
 evi 100
 bgp
 route-target import 65000:100
 route-target export 65000:100
 !
 !
!
commit label EVPN-L3VPN-PE1 comment Aktifkan EVPN L3VPN CUSTOMER-A PE1
end
```

Penjelasan configuration RT ganda:

- **import route-target 65000:100:** Untuk VPNv4, MPLS L3VPN klasik.
- **import route-target 65000:100 evpn:** Untuk EVPN, Route Type 5.

Dengan dua RT ini, PE1 dapat menerima prefix dari PE yang menggunakan VPNv4 maupun dari PE yang menggunakan EVPN.

Configuration di PE2 yang masih menggunakan MPLS L3VPN klasik tetap seperti Modul 4, tetapi tambahkan juga RT evpn:

```
configure
vrf CUSTOMER-A
 rd 65000:100
 address-family ipv4 unicast
 import route-target 65000:100
 export route-target 65000:100
 import route-target 65000:100 evpn
 export route-target 65000:100 evpn
 !
!
router bgp 65000
 vrf CUSTOMER-A
 rd 65000:100
 address-family ipv4 unicast
 label mode per-vrf
 redistribute connected
 !
 !
!
commit label INTERWORK-PE2 comment Tambah RT evpn di PE2 untuk interworking
end
```

### 8.4 Interworking L3VPN dan EVPN, RT Stitching

Ada tiga pola interworking:

**Pola 1: PE Dual-Stack, satu PE bicara dua bahasa**

Satu PE mengiklankan prefix yang sama via VPNv4 dan via EVPN. PE lain dapat memilih salah satu.

**Pola 2: RT Stitching di RR atau Border Leaf**

RR atau border leaf melakukan stitching: menerima VPNv4 dari satu sisi dan mengiklankan kembali sebagai EVPN ke sisi lain. Dipakai untuk inter-AS atau DCI antar domain yang berbeda.

**Pola 3: VRF Leaking antar VRF**

VRF CUSTOMER-A di PE yang sama dapat leak ke VRF CUSTOMER-B via RT yang berbeda.

Contoh VRF leaking untuk shared service:

```
configure
vrf SHARED-SERVICE
 rd 65000:999
 address-family ipv4 unicast
 import route-target 65000:100
 import route-target 65000:200
 export route-target 65000:999
 !
!
vrf CUSTOMER-A
 rd 65000:100
 address-family ipv4 unicast
 import route-target 65000:100
 import route-target 65000:999
 export route-target 65000:100
 !
!
commit
end
```

### 8.5 DCI dan Inter-AS EVPN

Ada tiga opsi inter-AS EVPN:

- **Option A:** VRF-to-VRF di border. Sederhana, tetapi tidak scalable.
- **Option B:** eBGP VPNv4 dan EVPN dengan next-hop-self. Scalable, titik awal yang paling seimbang.
- **Option C:** eBGP dengan label dan next-hop unchanged. Paling scalable, tetapi paling kompleks.

Contoh eBGP EVPN antar AS:

```
configure
router bgp 65000
 neighbor 10.0.1.1
 remote-as 65001
 description PEER-AS-65001-DCI
 update-source Loopback0
 address-family l2vpn evpn
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 address-family vpnv4 unicast
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 !
!
commit
end
```

### 8.6 Design RT untuk Interworking yang Bersih

- **Satu RT per service per customer:** Jangan gunakan RT yang sama untuk CUSTOMER-A dan CUSTOMER-B.
- **Pisahkan RT untuk L2 dan L3 jika perlu.**
- **Dokumentasikan RT di IPAM:** RT harus dicatat di IPAM atau NMS, bukan hanya di configuration router.
- **Gunakan RT yang mudah dibaca:** Format AS:NN, misalnya 65000:100 untuk CUSTOMER-A, 65000:200 untuk CUSTOMER-B, 65000:999 untuk shared service.

| Service | RT VPNv4 | RT EVPN | Keterangan |
| --- | --- | --- | --- |
| CUSTOMER-A L3VPN | 65000:100 | 65000:100 evpn | Dapat diakses via VPNv4 dan EVPN |
| CUSTOMER-A VPWS | Tidak ada | 65000:100 | Hanya EVPN |
| CUSTOMER-A E-LAN | Tidak ada | 65000:200 | Hanya EVPN |
| SHARED-SERVICE | 65000:999 | 65000:999 evpn | Dapat diakses dari semua VRF |

### 8.7 Verification Interworking

Di PE EVPN:

```
show bgp l2vpn evpn summary
show bgp l2vpn evpn route-type 5
show bgp l2vpn evpn evi 100
show route vrf CUSTOMER-A ipv4
show bgp vrf CUSTOMER-A ipv4 unicast summary
```

Di PE MPLS:

```
show bgp vpnv4 unicast summary
show bgp vpnv4 unicast rd 65000:100
show route vrf CUSTOMER-A ipv4
```

Di RR:

```
show bgp l2vpn evpn summary
show bgp vpnv4 unicast summary
show bgp l2vpn evpn route-type 5
show bgp vpnv4 unicast rd 65000:100
```

Uji end-to-end:

```
ping vrf CUSTOMER-A 192.168.2.2 source 192.168.1.1
traceroute vrf CUSTOMER-A 192.168.2.2 source 192.168.1.1
ping vrf CUSTOMER-A 2001:db8:192:2::2 source 2001:db8:192:1::1
```

### 8.8 Anti-Pattern Interworking

- **RT evpn lupa dikonfigurasi:** PE MPLS tidak akan menerima Route Type 5 jika tidak ada `import route-target... evpn`.
- **RD berbeda tetapi RT sama:** RD boleh berbeda per PE, tetapi RT harus sama untuk service yang sama.
- **BGP EVPN tidak aktif di RR:** RR yang hanya mengaktifkan VPNv4 tidak akan me-reflect EVPN.
- **VRF leaking tanpa filter:** Leaking semua RT ke semua VRF membuat isolasi customer rusak.
- **Mencampur EVI dan VRF secara acak:** Satu EVI harus memiliki satu RT yang jelas dan satu VRF yang jelas.


## Modul 9: Scale, High Availability, QoS Core, dan Security Control Plane

### 9.1 Scale dan Capacity Core

Scale bukan soal berapa banyak route yang dapat ditampung hari ini. Scale adalah soal berapa banyak route, VRF, label, SID, dan session BGP yang masih aman 3 tahun ke depan tanpa ganti hardware.

**Tiga scale yang harus dihitung terpisah:**

- **RIB scale:** Berapa banyak route di control plane, termasuk VPNv4, VPNv6, EVPN Route Type 2 dan Route Type 5.
- **FIB scale:** Berapa banyak prefix yang diprogram ke hardware. FIB selalu lebih kecil dari RIB, dan FIB yang penuh lebih berbahaya dari RIB yang penuh.
- **Label dan SID scale:** Berapa banyak label MPLS, Prefix SID, dan SID SRv6 yang dipakai. Label per-VRF jauh lebih hemat dari label per-prefix.

| Platform | Peran | FIB IPv4 dan IPv6 | VRF | Session BGP | Catatan Arsitek |
| --- | --- | --- | --- | --- | --- |
| ASR 9000 Tomahawk dan Lightspeed | P dan PE besar | 2 juta sampai 4 juta, tergantung line card | Ribuan | Ribuan | Paling aman untuk core dengan RIB besar dan banyak VRF |
| NCS 5500 Jericho2 | P dan PE padat | 1 juta sampai 2 juta | Ribuan | Ribuan | Padat dan hemat daya, periksa scale EVPN per line card |
| NCS 5700 Jericho2C+ | P dan PE 800G | 2 juta ke atas | Ribuan | Ribuan | Reference untuk SRv6 dan 800G, scale paling longgar |
| NCS 540 Jericho2 compact | PE aggregation | Ratusan ribu | Ratusan | Ratusan | Jangan dipaksakan jadi P core besar, FIB akan penuh duluan |
| XRv 9000 | Lab dan validation | Terbatas, untuk logika | Puluhan | Puluhan | Untuk validation sintaks dan BGP, bukan untuk benchmark scale |

Prinsip design scale:

- **Satu label per VRF, bukan per prefix:** Di Modul 4 dan Modul 6 telah digunakan `label mode per-vrf` dan `alloc mode per-vrf`.
- **IGP hanya untuk infrastructure:** Jika prefix customer masuk IGP, FIB IGP akan membengkak dan konvergensi melambat.
- **Summarisasi di PE-CE:** Jangan terima prefix /32 yang tidak perlu dari CE. Gunakan prefix-set dan route-policy untuk filter.
- **Batasi jumlah VRF per PE:** VRF yang tidak dipakai tetap memakan memori dan proses.
- **Pisahkan RR dari P:** RR yang juga menjadi P akan kehabisan CPU saat BGP churn dan forwarding bersamaan.

Contoh filter prefix di PE agar scale tetap sehat:

```
configure
prefix-set PE-CE-ALLOWED
 192.168.0.0/16 le 24,
 2001:db8:192::/48 le 64
end-set
!
route-policy PE-CE-IN
 if destination in PE-CE-ALLOWED then
 pass
 else
 drop
 endif
end-policy
!
router bgp 65000
 vrf CUSTOMER-A
 neighbor 192.168.1.2
 remote-as 65001
 address-family ipv4 unicast
 route-policy PE-CE-IN in
 route-policy PASS-ALL out
 !
 !
 !
!
commit label SCALE-PE-CE comment Filter prefix PE-CE CUSTOMER-A
end
```

Verification scale secara rutin:

```
show cef ipv4 summary
show cef ipv6 summary
show route ipv4 summary
show bgp vpnv4 unicast summary
show bgp l2vpn evpn summary
show mpls forwarding summary
show segment-routing srv6 sid summary
show processes memory
show platform
```

Jika `show cef summary` menampilkan FIB di atas 80 persen, itu sudah alarm. Jangan tunggu 95 persen.

### 9.2 High Availability, NSF, NSR, dan ISSU

High availability di IOS-XR bukan satu fitur, melainkan lapisan.

**Lapisan 1: Redundansi hardware**

Dual RP atau RSP, dual fabric, dual power, dual fan. Di ASR 9000 dan NCS 5700, ini wajib untuk P dan PE core.

```
show redundancy
show platform
show environment power
show environment fan
show controllers fabric plane all
```

**Lapisan 2: Non-Stop Forwarding dan Non-Stop Routing**

| Protokol | NSF | NSR | Configuration di XR |
| --- | --- | --- | --- |
| OSPF | Ya, dengan `nsf cisco` | Ya, dengan `nsr` | `router ospf 100`, `nsf cisco`, `nsr` |
| IS-IS | Ya, dengan `nsf cisco` | Ya, dengan `nsr` | `router isis CORE`, `nsf cisco`, `nsr` |
| BGP | Ya, dengan `bgp graceful-restart` | Ya, dengan `nsr` di BGP | `router bgp 65000`, `bgp graceful-restart`, `nsr` |
| LDP | Ya, dengan `graceful-restart` | Ya, dengan `nsr` | `mpls ldp`, `graceful-restart`, `nsr` |
| SR-MPLS | Mengikuti IGP | Mengikuti IGP | Tidak ada NSF terpisah, ikut IGP |
| SRv6 | Mengikuti IS-IS | Mengikuti IS-IS | Ikut IS-IS SRv6 |

Configuration HA lengkap di PE1:

```
configure
router ospf 100
 router-id 10.0.0.1
 nsr
 nsf cisco
!
router isis CORE
 nsf cisco
 nsr
!
mpls ldp
 nsr
 graceful-restart
!
router bgp 65000
 bgp router-id 10.0.0.1
 bgp graceful-restart
 nsr
!
commit label HA-PE1 comment Aktifkan NSF dan NSR PE1
end
```

Perbedaan NSF dan NSR:

- **NSF:** Saat RP switchover, neighbor membantu rebuild state. Ada jeda singkat, tetapi forwarding tidak berhenti karena FIB tetap di line card.
- **NSR:** State sudah disinkronkan ke standby RP sebelum switchover. Neighbor tidak tahu ada switchover.

Untuk core, aktifkan keduanya. NSR untuk mulus, NSF sebagai fallback jika NSR gagal.

**Lapisan 3: BFD, TI-LFA, dan Fast Reroute**

```
configure
bfd
 multiplier 3
 minimum-interval 300
!
router ospf 100
 area 0
 interface HundredGigE0/0/0/0
 bfd fast-detect
 fast-reroute per-prefix
 fast-reroute per-prefix ti-lfa
 !
 !
!
commit label HA-BFD-TILFA comment Aktifkan BFD dan TI-LFA PE1
end
```

**Lapisan 4: ISSU dan SMU**

ISSU atau In-Service Software Upgrade memungkinkan upgrade XR tanpa reload chassis. SMU atau Software Maintenance Upgrade memungkinkan patch bug tanpa upgrade penuh.

```
show install active
install add source disk0:/xr-7.11.2-smu-1.rpm
install activate
install commit
show install active
```

Aturan operasi: selalu `install commit` setelah `install activate`. Uji SMU di XRv 9000 dulu, lalu di satu PE non-kritis, baru ke seluruh core. Jangan upgrade semua P dan PE dalam satu jendela.

**Lapisan 5: Redundansi topology**

Setiap PE memiliki minimal 2 uplink ke P yang berbeda. Setiap P memiliki minimal 2 path ke core. Setiap PE peer ke 2 RR di POP berbeda. Setiap link core memiliki backup path untuk TI-LFA.

Verification HA:

```
show redundancy
show ospf nsr status
show isis nsr status
show bgp nsr
show mpls ldp nsr
show bfd session
show segment-routing traffic-eng policy
show logging last 100
```

### 9.3 QoS Core, DiffServ, MPLS EXP, dan H-QoS

QoS core bukan soal membuat traffic cepat. QoS core adalah soal memastikan traffic yang penting tetap berjalan saat core congested, dan traffic yang tidak penting yang dibuang duluan.

| Traffic Class | DSCP | MPLS EXP dan Traffic Class | Contoh | Perlakuan |
| --- | --- | --- | --- | --- |
| Voice | EF 46 | 5 | VoIP, voice trunk | Priority, latency paling rendah, police ketat |
| Video dan Real-time | AF41, CS4 | 4 | Video conference, IPTV | Bandwidth dijamin, delay rendah |
| Business Critical | AF31, AF21 | 3 dan 2 | ERP, banking, L3VPN enterprise | Bandwidth dijamin, WRED untuk TCP |
| Best Effort | BE 0 | 0 | Internet browsing | Sisa bandwidth, dibuang duluan saat congested |
| Network Control | CS6, CS7 | 6 dan 7 | OSPF, IS-IS, BGP, LDP, BFD | Priority paling tinggi, jangan pernah police terlalu rendah |

Prinsip arsitek QoS:

- **Trust di edge, bukan di core:** PE mengklasifikasikan dan menandai traffic dari CE. P hanya membaca EXP atau DSCP yang sudah ditandai.
- **Jangan remark DSCP customer secara sembarangan.**
- **MPLS EXP berasal dari Traffic Class:** Di IOS-XR, `traffic-class` dan `qos-group` diatur, lalu IOS-XR memetakan ke EXP saat penambahan label MPLS.
- **Egress queuing adalah tempat QoS benar-benar terjadi.**

**Contoh 1: Class-map untuk classification**

```
configure
class-map type qos match-any VOICE
 match dscp ef
 match dscp cs5
!
class-map type qos match-any VIDEO
 match dscp af41
 match dscp cs4
!
class-map type qos match-any BUSINESS
 match dscp af31
 match dscp af21
 match dscp af11
!
class-map type qos match-any CONTROL
 match dscp cs6
 match dscp cs7
!
commit label QOS-CLASSMAP comment Buat class-map QoS core
end
```

**Contoh 2: Policy ingress di PE, dari CE ke core**

```
configure
policy-map type qos INGRESS-PE-CE
 class CONTROL
 set traffic-class 6
 set qos-group 6
 !
 class VOICE
 set traffic-class 5
 set qos-group 5
 !
 class VIDEO
 set traffic-class 4
 set qos-group 4
 !
 class BUSINESS
 set traffic-class 3
 set qos-group 3
 !
 class class-default
 set traffic-class 0
 set qos-group 0
 !
!
commit label QOS-INGRESS comment Buat policy ingress PE-CE
end
```

**Contoh 3: Policy egress di P dan PE core**

```
configure
policy-map type qos EGRESS-CORE
 class CONTROL
 priority level 1
 police rate 500 mbps
 !
 class VOICE
 priority level 2
 police rate 5 percent
 !
 class VIDEO
 bandwidth remaining percent 30
 random-detect dscp-based
 !
 class BUSINESS
 bandwidth remaining percent 40
 random-detect dscp-based
 !
 class class-default
 bandwidth remaining percent 20
 random-detect
 shape average 80 gbps
 !
!
commit label QOS-EGRESS comment Buat policy egress core
end
```

Penjelasan:

- **priority level 1 untuk CONTROL:** Traffic routing dan BFD tidak boleh queue. Jika CONTROL policed terlalu rendah, OSPF dan BGP akan flapping saat congested.
- **priority level 2 untuk VOICE:** Voice membutuhkan latency rendah, tetapi jangan beri priority tanpa police.
- **bandwidth remaining percent:** Membagi sisa bandwidth setelah priority class terpenuhi.
- **random-detect:** WRED untuk TCP.

**Contoh 4: Aplikasi policy ke interface**

```
configure
interface GigabitEthernet0/0/0/1
 description TO-CE1-CUSTOMER-A
 vrf CUSTOMER-A
 service-policy input INGRESS-PE-CE
!
interface HundredGigE0/0/0/0
 description TO-P1-CORE
 service-policy output EGRESS-CORE
!
commit label QOS-APPLY comment Aplikasikan QoS ke interface PE1
end
```

**H-QoS untuk PE-CE dengan banyak customer di satu port**

```
configure
policy-map type qos CHILD-CUSTOMER-A
 class BUSINESS
 shape average 1 gbps
 bandwidth remaining percent 60
 !
 class class-default
 shape average 1 gbps
 !
!
policy-map type qos PARENT-PORT
 class class-default
 shape average 10 gbps
 service-policy CHILD-CUSTOMER-A
 !
!
commit label HQOS comment Buat H-QoS PE-CE
end
```

Verification QoS:

```
show policy-map interface HundredGigE0/0/0/0 output
show policy-map interface GigabitEthernet0/0/0/1 input
show qos interface HundredGigE0/0/0/0 output
show controllers HundredGigE0/0/0/0
show policy-map type qos EGRESS-CORE
```

Jika `show policy-map interface` menampilkan drop di class CONTROL, itu adalah alarm kritis.

### 9.4 Security Control Plane, LPTS, BGP Security, dan RTBH

Security core memiliki dua musuh: attack dari luar yang membanjiri control plane, dan kesalahan configuration dari dalam yang menyebarkan route yang salah.

**LPTS, pengganti CoPP di IOS-XR**

Di IOS-XR, tidak ada CoPP seperti di IOS. Yang ada adalah LPTS atau Local Packet Transport Services. LPTS mengatur semua traffic yang ditujukan ke RP, termasuk BGP, OSPF, SSH, dan SNMP. Setiap jenis traffic memiliki policer sendiri, sehingga banjir BGP tidak dapat mematikan SSH.

Verification LPTS:

```
show lpts pifib hardware policer location 0/0/CPU0
show lpts pifib policer
show lpts pifib flows
show lpts pifib statistics
```

Contoh tuning LPTS di PE1:

```
configure
lpts pifib policer bgp 5000
lpts pifib policer ospf-mc-known 2000
lpts pifib policer isis-known 2000
lpts pifib policer ldp 2000
lpts pifib policer ssh 500
lpts pifib policer snmp 500
lpts pifib policer icmp 1000
!
commit label LPTS-PE1 comment Tuning LPTS PE1
end
```

Aturan LPTS: jangan set policer CONTROL terlalu rendah. Monitor LPTS drop secara rutin. LPTS bukan pengganti ACL.

**Infrastructure ACL dan uRPF**

```
configure
ipv4 access-list INFRA-PROTECT
 10 permit ipv4 10.0.0.0/24 any
 20 permit ipv4 10.255.0.0/16 any
 30 permit tcp any 10.0.0.0/24 eq bgp
 40 permit tcp any 10.0.0.0/24 eq 179
 50 deny ipv4 any 10.0.0.0/24
 60 permit ipv4 any any
!
interface HundredGigE0/0/0/0
 ipv4 access-group INFRA-PROTECT ingress
 ipv4 verify unicast source reachable-via rx
!
commit label INFRA-ACL comment Aktifkan infrastructure ACL dan uRPF PE1
end
```

`ipv4 verify unicast source reachable-via rx` adalah uRPF strict di IOS-XR. Ini menolak packet dengan source IP yang tidak dapat kembali lewat interface yang sama.

**BGP Security: Password, TTL, dan Max-Prefix**

```
configure
router bgp 65000
 neighbor 10.0.0.101
 remote-as 65000
 description RR1
 password clear BGP-SECRET-RR1
 ttl-security hops 1
 update-source Loopback0
 address-family ipv4 unicast
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 maximum-prefix 100000 80
 !
 address-family vpnv4 unicast
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 maximum-prefix 500000 80
 !
 !
 neighbor 192.168.1.2
 remote-as 65001
 description CE1-CUSTOMER-A
 password clear BGP-SECRET-CE1
 ttl-security hops 1
 address-family ipv4 unicast
 route-policy PE-CE-IN in
 route-policy PASS-ALL out
 maximum-prefix 1000 90
 !
 !
!
commit label BGP-SEC-PE1 comment Aktifkan BGP security PE1
end
```

- **password:** MD5 untuk BGP.
- **ttl-security hops 1:** GTSM. Packet BGP dari neighbor langsung harus memiliki TTL 255.
- **maximum-prefix:** Membatasi jumlah prefix yang diterima.

**RPKI dan Validation Origin**

```
configure
router bgp 65000
 rpki server 192.0.2.1
 transport tcp port 3323
 username rpki-user
 password clear RPKI-SECRET
 !
 bgp origin-as validation signal ibgp
!
route-policy RPKI-CHECK
 if validation-state is invalid then
 drop
 elseif validation-state is valid then
 pass
 else
 pass
 endif
end-policy
!
router bgp 65000
 neighbor 192.168.1.2
 address-family ipv4 unicast
 route-policy RPKI-CHECK in
 !
 !
!
commit label RPKI-PE1 comment Aktifkan RPKI PE1
end
```

Aturan RPKI di produksi: jangan langsung drop `unknown` atau `not-found`. Drop hanya `invalid`.

**RTBH dan Flowspec untuk Mitigasi DDoS**

```
configure
route-policy RTBH-POLICY
 if community matches-any (65000:666) then
 set next-hop 192.0.2.1
 set local-preference 1000
 else
 pass
 endif
end-policy
!
router static
 address-family ipv4 unicast
 192.0.2.1/32 Null0
 !
!
router bgp 65000
 neighbor 10.0.0.101
 address-family ipv4 unicast
 route-policy RTBH-POLICY in
 !
 !
!
interface Null0
!
commit label RTBH-PE1 comment Siapkan RTBH PE1
end
```

Flowspec:

```
configure
router bgp 65000
 address-family ipv4 flowspec
 neighbor 10.0.0.101
 address-family ipv4 flowspec
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 !
 !
!
commit label FLOWSPEC-PE1 comment Aktifkan Flowspec PE1
end
```

Gunakan RTBH untuk mitigasi cepat dan kasar, Flowspec untuk mitigasi presisi.

**Management Plane Security**

```
configure
ssh server v2
ssh server vrf default
aaa authentication login default group tacacs+ local
aaa authorization exec default group tacacs+ local
!
ipv4 access-list MGMT-ALLOW
 10 permit ipv4 192.0.2.0/24 any
 20 deny ipv4 any any
!
line vty 0 99
 access-class MGMT-ALLOW ingress
 transport input ssh
!
commit label MGMT-SEC comment Amankan management plane PE1
end
```

### 9.5 Design Operasional untuk Core yang Tahan Gagal

- **Satu perubahan, satu commit, satu label.**
- **Jendela maintenance untuk QoS dan LPTS:** Jangan ubah QoS egress dan LPTS di jam sibuk. Gunakan `commit confirmed`.
- **Dokumentasi RT, RD, dan SID:** Harus ada di IPAM dan git, bukan hanya di router.
- **Monitoring yang benar:** Monitor FIB usage, LPTS drop, QoS drop di class CONTROL, BGP prefix count, dan TI-LFA coverage.
- **Latihan failover:** Setiap kuartal, matikan satu link, satu RP, dan satu RR di lab dan ukur loss.


## Appendix A: Golden Config PE, P, dan RR

Appendix ini adalah template siap gunakan. Ganti hostname, Loopback, dan interface sesuai POP. Semua template telah digabungkan dari Modul 1 sampai Modul 9.

### A.1 Golden Config PE SR-MPLS

```
hostname PE1
!
logging console disable
logging monitor disable
!
vrf CUSTOMER-A
 rd 65000:100
 address-family ipv4 unicast
 import route-target 65000:100
 export route-target 65000:100
 import route-target 65000:100 evpn
 export route-target 65000:100 evpn
 !
!
interface Loopback0
 ipv4 point-to-point
 ipv4 address 10.0.0.1/32
!
interface HundredGigE0/0/0/0
 description TO-P1-CORE
 ipv4 point-to-point
 ipv4 address 10.255.0.0/31
 no shutdown
!
interface GigabitEthernet0/0/0/1
 description TO-CE1-CUSTOMER-A
 vrf CUSTOMER-A
 ipv4 point-to-point
 ipv4 address 192.168.1.1/30
 no shutdown
!
segment-routing
 global-block 16000 23999
!
mpls label range table 0 24000 1048575
!
router ospf 100
 router-id 10.0.0.1
 nsr
 nsf cisco
 segment-routing mpls
 segment-routing forwarding mpls
 area 0
 interface Loopback0
 passive enable
 prefix-sid index 1
 !
 interface HundredGigE0/0/0/0
 network point-to-point
 hello-interval 3
 dead-interval 9
 bfd fast-detect
 fast-reroute per-prefix
 fast-reroute per-prefix ti-lfa
 !
 !
!
mpls ldp
 router-id 10.0.0.1
 nsr
 graceful-restart
 session protection
 interface HundredGigE0/0/0/0
 !
!
route-policy PASS-ALL
 pass
end-policy
!
router bgp 65000
 bgp router-id 10.0.0.1
 bgp graceful-restart
 nsr
 neighbor 10.0.0.101
 remote-as 65000
 description RR1
 password clear BGP-SECRET-RR1
 ttl-security hops 1
 update-source Loopback0
 address-family ipv4 unicast
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 maximum-prefix 100000 80
 !
 address-family vpnv4 unicast
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 maximum-prefix 500000 80
 !
 address-family l2vpn evpn
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 !
 neighbor 10.0.0.102
 remote-as 65000
 description RR2
 password clear BGP-SECRET-RR2
 ttl-security hops 1
 update-source Loopback0
 address-family ipv4 unicast
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 address-family vpnv4 unicast
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 address-family l2vpn evpn
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 !
 vrf CUSTOMER-A
 rd 65000:100
 address-family ipv4 unicast
 label mode per-vrf
 redistribute connected
 !
 neighbor 192.168.1.2
 remote-as 65001
 description CE1-CUSTOMER-A
 password clear BGP-SECRET-CE1
 ttl-security hops 1
 address-family ipv4 unicast
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 maximum-prefix 1000 90
 !
 !
 !
!
evpn
 evi 100
 bgp
 route-target import 65000:100
 route-target export 65000:100
 !
 !
!
lpts pifib policer bgp 5000
lpts pifib policer ospf-mc-known 2000
lpts pifib policer ssh 500
!
commit label GOLDEN-PE1-SR-MPLS comment Golden config PE1 SR-MPLS
end
```

### A.2 Golden Config P

P tidak memiliki VRF dan tidak memiliki BGP service. P hanya IGP, LDP atau SR, dan QoS core.

```
hostname P1
!
interface Loopback0
 ipv4 point-to-point
 ipv4 address 10.0.0.11/32
!
interface HundredGigE0/0/0/0
 description TO-PE1
 ipv4 point-to-point
 ipv4 address 10.255.0.1/31
 no shutdown
!
interface HundredGigE0/0/0/1
 description TO-P2
 ipv4 point-to-point
 ipv4 address 10.255.0.2/31
 no shutdown
!
segment-routing
 global-block 16000 23999
!
router ospf 100
 router-id 10.0.0.11
 nsr
 nsf cisco
 segment-routing mpls
 segment-routing forwarding mpls
 area 0
 interface Loopback0
 passive enable
 prefix-sid index 11
 !
 interface HundredGigE0/0/0/0
 network point-to-point
 bfd fast-detect
 fast-reroute per-prefix
 fast-reroute per-prefix ti-lfa
 !
 interface HundredGigE0/0/0/1
 network point-to-point
 bfd fast-detect
 fast-reroute per-prefix
 fast-reroute per-prefix ti-lfa
 !
 !
!
mpls ldp
 router-id 10.0.0.11
 nsr
 graceful-restart
 interface HundredGigE0/0/0/0
 !
 interface HundredGigE0/0/0/1
 !
!
policy-map type qos EGRESS-CORE
 class CONTROL
 priority level 1
 police rate 500 mbps
 !
 class VOICE
 priority level 2
 police rate 5 percent
 !
 class class-default
 bandwidth remaining percent 80
 !
!
interface HundredGigE0/0/0/0
 service-policy output EGRESS-CORE
!
interface HundredGigE0/0/0/1
 service-policy output EGRESS-CORE
!
commit label GOLDEN-P1 comment Golden config P1
end
```

### A.3 Golden Config RR

RR tidak mem-forward traffic customer. RR hanya IGP dan BGP.

```
hostname RR1
!
interface Loopback0
 ipv4 point-to-point
 ipv4 address 10.0.0.101/32
!
interface HundredGigE0/0/0/0
 description TO-P1
 ipv4 point-to-point
 ipv4 address 10.255.0.10/31
 no shutdown
!
router ospf 100
 router-id 10.0.0.101
 nsr
 nsf cisco
 area 0
 interface Loopback0
 passive enable
 prefix-sid index 101
 !
 interface HundredGigE0/0/0/0
 network point-to-point
 bfd fast-detect
 !
 !
!
route-policy PASS-ALL
 pass
end-policy
!
router bgp 65000
 bgp router-id 10.0.0.101
 bgp graceful-restart
 nsr
 bgp cluster-id 10.0.0.101
 neighbor 10.0.0.1
 remote-as 65000
 description PE1
 update-source Loopback0
 address-family ipv4 unicast
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 address-family vpnv4 unicast
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 address-family l2vpn evpn
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 !
 neighbor 10.0.0.2
 remote-as 65000
 description PE2
 update-source Loopback0
 address-family ipv4 unicast
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 address-family vpnv4 unicast
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 address-family l2vpn evpn
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 !
 neighbor 10.0.0.102
 remote-as 65000
 description RR2
 update-source Loopback0
 address-family ipv4 unicast
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 address-family vpnv4 unicast
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 address-family l2vpn evpn
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 !
!
commit label GOLDEN-RR1 comment Golden config RR1
end
```

### A.4 Varian SRv6 untuk PE

Untuk PE SRv6, ganti blok SR-MPLS dengan blok SRv6. Sisanya sama: VRF, BGP, EVPN, QoS, LPTS tetap dipakai.

```
segment-routing
 srv6
 locators
 locator MAIN
 micro-segment behavior unode psp-usd
 prefix 2001:db8:100::/48
 !
 !
 !
!
interface Loopback0
 ipv6 point-to-point
 ipv6 address 2001:db8::1/128
!
interface HundredGigE0/0/0/0
 ipv6 point-to-point
 ipv6 address 2001:db8:255::0/127
!
router isis CORE
 address-family ipv6 unicast
 segment-routing srv6
 locator MAIN
 !
 !
!
router bgp 65000
 segment-routing srv6
 locator MAIN
 allocation mode per-vrf
 !
 vrf CUSTOMER-A
 address-family ipv4 unicast
 segment-routing srv6
 locator MAIN
 alloc mode per-vrf
 !
 !
 !
!
commit label GOLDEN-PE1-SRV6 comment Varian SRv6 PE1
end
```

## Appendix B: Troubleshooting Playbook

### B.1 Prinsip Troubleshooting Principal

- **Jangan tebak, buktikan dengan satu perintah.** Setiap hipotesis harus dibuktikan dengan `show`.
- **Kerjakan dari edge ke core, lalu balik.** Mulai dari PE-CE, lalu PE, RR, P, lalu PE tujuan, lalu CE tujuan.
- **Satu perubahan dalam satu waktu.** Jangan ubah OSPF dan BGP bersamaan saat troubleshooting.
- **Catat sebelum dan sesudah.** Simpan output `show` sebelum perubahan untuk perbandingan.

### B.2 L3VPN Down, CE Tidak Dapat Ping CE

| Langkah | Perintah | Hasil yang Diharapkan | Jika Gagal, Artinya |
| --- | --- | --- | --- |
| 1. PE-CE | `show bgp vrf CUSTOMER-A ipv4 unicast summary` | Neighbor Established, prefix diterima | Periksa interface, VRF, password BGP, route-policy |
| 2. VPNv4 di PE source | `show bgp vpnv4 unicast rd 65000:100` | Ada route dari PE tujuan | Periksa RT import dan export |
| 3. RR | `show bgp vpnv4 unicast rd 65000:100` di RR | RR menerima dari PE source dan PE tujuan | Periksa BGP PE ke RR, periksa address-family vpnv4 aktif |
| 4. PE tujuan import | `show route vrf CUSTOMER-A ipv4` di PE tujuan | Ada route customer | Periksa RT evpn jika interworking EVPN, periksa RD |
| 5. LSP | `show mpls forwarding` dan `show cef vrf CUSTOMER-A` | Label dan next-hop benar | Periksa IGP dan LDP atau SR |
| 6. Return path | Ulangi langkah 1 sampai 5 dari arah berlawanan | Harus simetris | Jika satu arah berjalan, periksa RT di sisi yang gagal |

Perintah cepat L3VPN:

```
show vrf CUSTOMER-A detail
show bgp vrf CUSTOMER-A ipv4 unicast neighbors 192.168.1.2
show bgp vpnv4 unicast vrf CUSTOMER-A
show cef vrf CUSTOMER-A ipv4 192.168.2.0/24 detail
ping vrf CUSTOMER-A 192.168.2.2 source 192.168.1.1
traceroute vrf CUSTOMER-A 192.168.2.2 source 192.168.1.1
```

### B.3 EVPN VPWS dan E-LAN Down

| Gejala | Periksa Pertama | Penyebab Paling Sering |
| --- | --- | --- |
| VPWS Down | `show l2vpn xconnect group VPWS-CUSTOMER-A` | EVI tidak cocok, RT tidak cocok, BGP EVPN belum Established |
| E-LAN MAC tidak belajar | `show bgp l2vpn evpn route-type 2` | BGP EVPN belum Established, EVI belum aktif di bridge-domain |
| ESI multihoming tidak sinkron | `show evpn ethernet-segment` | ESI tidak sama persis di dua PE, termasuk format identifier |
| BUM flood berlebihan | `show bgp l2vpn evpn route-type 3` | Route Type 3 tidak terdistribusi, periksa RT dan RR |
| IRB tidak dapat ping gateway | `show interface BVI200` dan `show arp vrf CUSTOMER-A` | BVI tidak assigned ke bridge-domain, atau VRF salah |

```
show evpn evi 100 detail
show evpn evi 200 detail
show l2vpn bridge-domain group BG-CUSTOMER-A
show l2vpn xconnect detail
show bgp l2vpn evpn summary
show bgp l2vpn evpn evi 100
```

### B.4 SR-MPLS dan SR Policy Tidak Up

| Gejala | Periksa | Penyebab |
| --- | --- | --- |
| Prefix SID tidak muncul | `show ospf segment-routing` atau `show isis segment-routing label table` | SRGB tidak sama, `segment-routing mpls` belum aktif, `prefix-sid index` salah tempat |
| Label SR tidak dipakai, masih LDP | `show mpls forwarding` | `sr-prefer` belum aktif, SRGB tabrakan dengan LDP |
| SR Policy Down | `show segment-routing traffic-eng policy` | Segment list salah, endpoint tidak reachable, color tidak cocok dengan BGP VRF |
| TI-LFA tidak ada backup | `show cef ipv4 10.0.0.2/32 detail` | Topology tidak memiliki path alternatif, metric asimetris |

```
show segment-routing mpls mapping-server
show segment-routing traffic-eng policy all
show segment-routing traffic-eng policy color 100 end-point ipv4 10.0.0.2 detail
show mpls forwarding
```

### B.5 SRv6 SID Tidak Teralokasi

| Gejala | Periksa | Penyebab |
| --- | --- | --- |
| Locator tidak muncul | `show segment-routing srv6 locator MAIN detail` | Prefix locator salah, `micro-segment behavior` tidak cocok |
| SID End.DT4 tidak ada | `show segment-routing srv6 sid` | `segment-routing srv6` belum aktif di BGP VRF, allocation mode belum per-vrf |
| Ping VRF gagal via SRv6 | `show bgp vpnv4 unicast vrf CUSTOMER-A` dan `show cef ipv6` | IS-IS SRv6 belum aktif, locator tidak diiklankan via IGP |

### B.6 BGP Flapping dan IGP Flapping

- **BGP flapping:** Periksa `show logging`, periksa LPTS drop untuk BGP, periksa MTU, periksa `maximum-prefix` apakah session ter-shutdown karena prefix berlebih.
- **IGP flapping:** Periksa BFD, periksa LPTS drop untuk OSPF atau IS-IS, periksa QoS class CONTROL apakah policed terlalu rendah, periksa interface error dan CRC.
- **Semua flapping bersamaan:** Periksa power, fabric, dan RP. Jangan troubleshooting BGP dulu jika hardware bermasalah.

```
show logging last 200
show lpts pifib policer
show bfd session
show interfaces HundredGigE0/0/0/0
show policy-map interface HundredGigE0/0/0/0 output
show redundancy
```

### B.7 Template Laporan Insiden

Setiap insiden harus dicatat dengan format ini:

- Waktu mulai dan selesai, dalam WIB
- Dampak: customer, VRF, dan prefix yang terdampak
- Gejala: apa yang dilihat pertama kali
- Perintah yang dipakai dan output kunci
- Root cause: satu kalimat yang jelas
- Perbaikan: commit ID dan label yang dipakai
- Prevention: apa yang diubah di golden config dan monitoring

## Appendix C: Inter-AS dan DCI

### C.1 Kapan Membutuhkan Inter-AS

- ISP memiliki 2 AS yang berbeda karena akuisisi
- DCI antar data center dengan AS berbeda
- Peering L3VPN antar ISP untuk customer multi-homing antar ISP
- DCI antar POP yang menggunakan AS berbeda untuk isolasi

### C.2 Inter-AS Option B untuk L3VPN

Option B adalah yang paling seimbang untuk ISP. Border router saling peer eBGP VPNv4, next-hop diubah ke border router sendiri.

Topology:

```
PE1 (AS 65000) -- ASBR1 (AS 65000) -- eBGP VPNv4 -- ASBR2 (AS 65001) -- PE2 (AS 65001)
```

Configuration ASBR1 di AS 65000:

```
configure
route-policy PASS-ALL
 pass
end-policy
!
router bgp 65000
 bgp router-id 10.0.0.201
 neighbor 10.0.1.1
 remote-as 65001
 description ASBR2-AS-65001
 update-source Loopback0
 address-family vpnv4 unicast
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 address-family l2vpn evpn
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 !
 neighbor 10.0.0.101
 remote-as 65000
 description RR1-AS-65000
 update-source Loopback0
 address-family vpnv4 unicast
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 address-family l2vpn evpn
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 !
!
mpls bgp forwarding
!
commit label INTERAS-B-ASBR1 comment Aktifkan Inter-AS Option B ASBR1
end
```

Configuration ASBR2 di AS 65001 adalah cerminan, hanya AS dan Loopback yang berbeda. Pastikan RT sama di kedua AS untuk service yang sama, atau lakukan RT translation jika RT berbeda antar AS.

### C.3 Inter-AS Option C, Ringkasan Arsitek

Option C menggunakan eBGP labeled unicast antar ASBR untuk Loopback, dan eBGP VPNv4 antar RR dengan next-hop unchanged. Ini paling scalable, tetapi paling kompleks. Gunakan Option C jika jumlah PE sudah ratusan dan Option B mulai menjadi bottleneck di ASBR. Jangan mulai dengan Option C jika tim baru pertama kali implementation Inter-AS.

### C.4 DCI dengan EVPN

DCI menggunakan EVPN untuk membawa L2 dan L3 antar data center. Prinsipnya sama dengan Inter-AS Option B, tetapi address-family yang dipakai adalah `l2vpn evpn`.

```
configure
router bgp 65000
 neighbor 10.0.1.1
 remote-as 65001
 description DCI-PEER-DC2
 update-source Loopback0
 address-family l2vpn evpn
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 !
!
evpn
 evi 100
 bgp
 route-target import 65000:100
 route-target export 65000:100
 !
 !
!
commit label DCI-EVPN comment Aktifkan DCI EVPN
end
```

Untuk DCI, perhatikan:

- **RT harus sama di kedua DC** untuk service yang sama, kecuali RT translation dilakukan secara sengaja di border.
- **Batasi BUM traffic antar DC.** BUM yang diflood antar DC dapat membanjiri DCI. Gunakan EVPN dengan IGMP snooping dan selective BUM jika memungkinkan.
- **Latency DCI harus diukur.** Jangan jadikan DCI sebagai path utama jika latency antar DC di atas SLA customer.

### C.5 Verification Inter-AS dan DCI

```
show bgp vpnv4 unicast summary
show bgp vpnv4 unicast neighbors 10.0.1.1
show bgp l2vpn evpn summary
show bgp l2vpn evpn neighbors 10.0.1.1
show bgp vpnv4 unicast rd 65000:100
show route vrf CUSTOMER-A ipv4
ping vrf CUSTOMER-A 192.168.2.2 source 192.168.1.1
```


## Appendix D: Flex-Algo untuk SR-MPLS dan SRv6

### D.1 Konsep Flex-Algo

Flex-Algo atau Flexible Algorithm memungkinkan satu IGP membawa beberapa topology logis. Setiap algoritma memiliki definisi: metric type, constraint, dan tujuan optimasi.

Contoh yang paling sering dipakai:

| Algoritma | Nama | Tujuan | Use Case |
| --- | --- | --- | --- |
| 0 | Default | IGP shortest path, metric IGP | Traffic biasa |
| 128 | Low Latency | Metric delay, hindari link high delay | VoIP, trading, 5G fronthaul |
| 129 | Disjoint dan Low Delay | Metric delay, hindari link yang sudah dipakai Algo 128 | Backup disjoint untuk low latency |
| 130 | Bandwidth | Metric TE, hindari link low bandwidth | Video, backup besar |

Tanpa Flex-Algo, SR Policy manual per flow harus dibuat. Dengan Flex-Algo, BGP atau SR Policy cukup dikonfigurasi untuk menggunakan Algo tertentu, dan IGP yang menghitung path-nya.

### D.2 Configuration Flex-Algo di IS-IS

Flex-Algo paling matang di IS-IS. OSPF juga mendukung, tetapi IS-IS adalah reference di ISP Tier-1.

Langkah 1: Definisikan Flex-Algo di IS-IS.

```
configure
router isis CORE
 flex-algo 128
 metric-type delay
 advertise-definition
 !
 flex-algo 129
 metric-type delay
 advertise-definition
 !
 address-family ipv4 unicast
 metric-style wide
 !
 interface HundredGigE0/0/0/0
 address-family ipv4 unicast
 metric 10
 !
 !
!
commit label FLEXALGO-DEF comment Definisikan Flex-Algo 128 dan 129
end
```

Langkah 2: Configuration delay measurement per link jika menggunakan metric delay. Tanpa delay measurement, metric delay tidak akan akurat.

```
configure
performance-measurement
 interface HundredGigE0/0/0/0
 delay-measurement
 !
 delay-profile interfaces
 advertisement
 accelerated
 periodic
 !
 !
!
commit
end
```

Langkah 3: Aktifkan Prefix SID untuk Flex-Algo di Loopback.

```
configure
router isis CORE
 interface Loopback0
 address-family ipv4 unicast
 prefix-sid index 1
 prefix-sid algorithm 128 index 128
 prefix-sid algorithm 129 index 129
 !
 !
!
commit label FLEXALGO-SID comment Aktifkan Prefix SID Flex-Algo Loopback
end
```

Penjelasan: Loopback sekarang memiliki 3 SID, satu untuk Algo 0 dengan index 1, satu untuk Algo 128 dengan index 128, dan satu untuk Algo 129 dengan index 129. Jika SRGB 16000, maka SID-nya adalah 16001, 16128, dan 16129.

### D.3 Flex-Algo untuk SR-MPLS

Setelah Flex-Algo aktif di IGP, SR Policy dapat menggunakan Flex-Algo sebagai constraint dinamis.

```
configure
segment-routing
 traffic-eng
 policy POL-LOWLAT-PE1-PE2
 color 128 end-point ipv4 10.0.0.2
 candidate-paths
 preference 100
 dynamic
 metric
 type delay
 !
 flex-algo 128
 !
 !
 !
 !
 !
!
commit label FLEXALGO-SR-MPLS comment SR Policy dengan Flex-Algo 128
end
```

### D.4 Flex-Algo untuk SRv6

Untuk SRv6, Flex-Algo dipakai dengan locator yang berbeda per algoritma, atau dengan SID yang berbeda per algoritma.

```
configure
segment-routing
 srv6
 locators
 locator MAIN
 micro-segment behavior unode psp-usd
 prefix 2001:db8:100::/48
 !
 locator LOWLAT
 micro-segment behavior unode psp-usd
 prefix 2001:db8:200::/48
 algorithm 128
 !
 !
 !
!
router isis CORE
 address-family ipv6 unicast
 segment-routing srv6
 locator MAIN
 locator LOWLAT
 !
 !
!
commit label FLEXALGO-SRV6 comment Locator SRv6 untuk Flex-Algo 128
end
```

### D.5 Flex-Algo untuk L3VPN dan EVPN

BGP dapat menandai VRF atau EVI untuk menggunakan Flex-Algo tertentu. Ini cara menjual SLA yang berbeda tanpa membuat SR Policy manual per customer.

```
configure
router bgp 65000
 vrf CUSTOMER-LOWLAT
 address-family ipv4 unicast
 segment-routing policy color 128
 !
 !
!
commit
end
```

### D.6 Verification Flex-Algo

```
show isis flex-algo 128
show isis flex-algo 129
show isis segment-routing label table
show segment-routing traffic-eng policy
show cef ipv4 10.0.0.2/32 detail
```

Jika `show isis flex-algo 128` menampilkan prefix dengan SID yang berbeda dari Algo 0, dan SR Policy color 128 Up, maka Flex-Algo sudah bekerja.

## Appendix E: MVPN dan NG-MVPN

### E.1 Kenapa MVPN

L3VPN unicast tidak membawa multicast. Jika customer membutuhkan IPTV, video conference multicast, atau routing multicast enterprise, MVPN dibutuhkan. Tanpa MVPN, multicast customer akan diflood atau tidak berjalan sama sekali lewat core MPLS.

### E.2 Konsep MVPN

- **Default MDT:** Multicast Distribution Tree default untuk setiap MVPN. Semua PE dalam satu MVPN bergabung ke MDT yang sama untuk BUM dan multicast.
- **Data MDT:** MDT dinamis untuk source multicast yang traffic-nya besar. Data MDT dibuat hanya saat ada traffic, untuk menghemat bandwidth core.
- **NG-MVPN:** Generasi baru MVPN yang menggunakan BGP, bukan PIM di core. NG-MVPN lebih scalable dan terintegrasi dengan EVPN dan SR.

| Jenis | Control Plane | Kapan Dipakai |
| --- | --- | --- |
| MDT GRE | PIM di core | Legacy, tidak disarankan untuk core baru |
| NG-MVPN | BGP, dengan PMSI dan I-PMSI | Reference untuk core baru, integrasi dengan SR dan EVPN |

### E.3 Configuration NG-MVPN di PE1

Langkah 1: Aktifkan multicast routing di VRF.

```
configure
multicast-routing
 vrf CUSTOMER-A
 address-family ipv4
 interface GigabitEthernet0/0/0/1
 enable
 !
 mdt default ipv4 239.0.0.1
 mdt data 239.0.1.0/24 threshold 10
 bgp
 auto-discovery
 inter-as
 pmsi
 !
 !
 !
 !
!
commit label MVPN-PE1 comment Aktifkan NG-MVPN CUSTOMER-A PE1
end
```

Langkah 2: Aktifkan BGP MVPN ke RR.

```
configure
router bgp 65000
 neighbor 10.0.0.101
 address-family ipv4 mvpn
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 !
 vrf CUSTOMER-A
 address-family ipv4 unicast
 redistribute connected
 !
 address-family ipv4 mvpn
 auto-discovery
 !
 !
!
commit label MVPN-BGP-PE1 comment Aktifkan BGP MVPN PE1
end
```

Langkah 3: Configuration CE untuk multicast. CE harus menjalankan PIM atau IGMP ke PE. PE akan meneruskan join multicast via BGP MVPN ke PE lain.

### E.4 MVPN dengan SR-MPLS dan SRv6

NG-MVPN di atas SR menggunakan PMSI yang sama, tetapi transportnya SR, bukan RSVP-TE atau LDP. Untuk SR-MPLS, pastikan Prefix SID sudah aktif. Untuk SRv6, pastikan locator sudah aktif. Tidak ada configuration MVPN tambahan yang spesifik untuk SR, karena MVPN menggunakan transport yang sudah ada.

### E.5 Verification MVPN

```
show multicast-routing vrf CUSTOMER-A ipv4
show pim vrf CUSTOMER-A ipv4 neighbor
show bgp ipv4 mvpn summary
show bgp ipv4 mvpn vrf CUSTOMER-A
show mpls forwarding
show mfib vrf CUSTOMER-A ipv4
```

Jika `show pim vrf CUSTOMER-A` menampilkan neighbor CE, dan `show bgp ipv4 mvpn` menampilkan route MVPN dari PE lain, maka MVPN sudah bekerja.

---

## Appendix F: Lab XRv 9000 dan Template Change

### F.1 Lab XRv 9000 yang Direkomendasikan

Topology lab minimal untuk validation semua modul:

```
CE1 -- PE1 -- P1 -- P2 -- PE2 -- CE2
 | |
 RR1 RR2
```

Spesifikasi lab:

- **Per node:** 4 vCPU, 8 GB RAM, 1 disk 20 GB untuk XRv 9000 dengan BGP dan SR
- **Topology:** 7 node, termasuk 2 CE yang dapat menggunakan XRv 9000 atau Linux dengan FRR
- **Validation per modul:** Modul 3 periksa IGP dan LDP, Modul 4 periksa L3VPN, Modul 5 periksa SR Policy, Modul 6 periksa SRv6, Modul 7 dan 8 periksa EVPN, Modul 9 periksa QoS dan LPTS

Skenario gagal yang wajib diuji di lab:

- Matikan satu link core, ukur loss dan TI-LFA
- Matikan satu RR, periksa BGP tetap Established ke RR kedua
- Matikan satu RP atau reload satu PE, periksa NSR dan NSF
- Banjiri BGP dengan prefix berlebih, periksa maximum-prefix
- Ubah QoS class CONTROL ke police terlalu rendah, lihat dampak ke OSPF

### F.3 Template Change Management

Setiap perubahan di core harus menggunakan template ini:

```
CHG ID: CHG-2026-XXXX
Tanggal dan Jendela: Sabtu 23:00 sampai 01:00 WIB
Pelaksana: Nama Engineer dan backup
Tujuan: Satu kalimat, contoh: Aktifkan SR-MPLS di PE1 dan P1
Dampak: Customer, VRF, dan link yang terdampak
Perubahan:
 - Commit 1: Aktifkan SRGB dan Prefix SID di PE1
 - Commit 2: Aktifkan SR-MPLS di OSPF PE1
 - Commit 3: Aktifkan sr-prefer
Rollback:
 - rollback configuration last 1 untuk tiap commit
 - Commit ID sebelumnya: 1000001234
Validation Sebelum:
 - show bgp vpnv4 unicast summary
 - show mpls forwarding
 - ping vrf CUSTOMER-A
Validation Sesudah:
 - show ospf segment-routing
 - show mpls forwarding
 - ping vrf CUSTOMER-A
 - show segment-routing traffic-eng policy
Persetujuan: Nama approver
```

## Penutup Appendix

Appendix A sampai F menutup celah antara guide teori dan operasi nyata. Modul 1 sampai Modul 9 memberikan fondasi dan service. Appendix memberikan template, troubleshooting, dan design lanjutan yang membuat guide ini dapat langsung digunakan di produksi, bukan hanya dibaca.
---

## Appendix G: Studi Kasus OSPF sebagai IGP Utama, Segment Routing sebagai Opsi 2

### G.1 Posisi Baru di Guide Ini

Mulai Appendix ini, posisi guide diperjelas:

- **Opsi 1, Utama:** OSPF sebagai IGP + LDP sebagai transport MPLS + BGP sebagai pembawa service
- **Opsi 2, Alternatif:** OSPF sebagai IGP + Segment Routing MPLS atau SRv6 sebagai transport + BGP sebagai pembawa service

Segment Routing tidak lagi dianggap sebagai pengganti wajib LDP. Segment Routing adalah opsi kedua yang dipakai hanya jika ada kebutuhan spesifik yang tidak dapat dipenuhi LDP. Ini keputusan arsitek yang lebih aman untuk ISP yang tim-nya sudah kuat OSPF dan LDP, dan yang membutuhkan stabilitas lebih dulu daripada fitur baru.

Modul 3, Modul 4, dan Modul 5 tetap valid. Yang berubah adalah prioritasnya. Modul 3 dan Modul 4 adalah fondasi utama. Modul 5 dan Modul 6 adalah opsi lanjutan.

### G.2 Kenapa OSPF Dipilih sebagai IGP Utama

| Alasan | Penjelasan |
| --- | --- |
| Tim sudah kuat OSPF | Mayoritas engineer ISP di Indonesia lebih cepat troubleshooting OSPF daripada IS-IS. Di insiden jam 2 pagi, kecepatan troubleshooting lebih penting dari keunggulan teori protokol. |
| Rekrutmen lebih mudah | Engineer OSPF lebih banyak di pasar. Engineer IS-IS yang benar-benar paham TLV dan level lebih sedikit. |
| Integrasi dengan enterprise | CE enterprise sering menggunakan OSPF PE-CE. Jika core juga OSPF, tim hanya perlu satu pola pikir OSPF. |
| Dokumentasi dan training lebih tersedia | Materi OSPF untuk IOS-XR lebih banyak dan lebih mudah diakses tim operasional. |
| Scale ISP menengah sampai besar masih aman | OSPF area 0 flat dengan 50 sampai 100 node masih sangat stabil jika desainnya benar. IS-IS baru unggul jelas di atas 200 node atau untuk SRv6 scale Tier-1. |

Kapan IS-IS tetap lebih baik: jika ISP sudah di atas 200 node core, jika target adalah SRv6 scale Tier-1 dengan Flex-Algo yang kompleks, atau jika tim sudah memiliki tim IS-IS yang kuat. Di luar kondisi itu, OSPF adalah pilihan yang lebih operasional.

### G.3 Studi Kasus: ISP Regional 3 POP, 10 PE, 4 P, 2 RR

**Profil ISP contoh:**

- Nama: ISP Nusantara Net, contoh fiktif
- POP: Jakarta, Surabaya, Bandung
- PE: 10 PE, 4 di Jakarta, 3 di Surabaya, 3 di Bandung
- P: 4 P, 2 di Jakarta, 1 di Surabaya, 1 di Bandung
- RR: 2 RR, 1 di Jakarta, 1 di Surabaya
- Customer: 80 VRF, 60 persen L3VPN, 30 persen EVPN E-LAN dan VPWS, 10 persen internet
- Traffic puncak: 120 Gbps, pertumbuhan 30 persen per tahun
- Tim: 6 network engineer, 4 di antaranya kuat OSPF, 1 kuat IS-IS, 1 junior

**Keputusan arsitek untuk ISP ini:**

- IGP: OSPF area 0 flat, satu area saja, tidak ada area 1 atau area 2
- Transport Opsi 1: LDP, karena tim sudah paham dan tidak ada kebutuhan TE yang kompleks
- Transport Opsi 2: SR-MPLS, hanya diaktifkan nanti jika ada kebutuhan SR Policy atau TI-LFA yang tidak dapat dipenuhi LDP
- BGP: iBGP ke RR, dengan VPNv4 dan EVPN
- QoS dan LPTS: sama seperti Modul 9, tidak berubah

Hasilnya: ISP ini berjalan stabil selama 2 tahun dengan Opsi 1, tanpa SR. Baru di tahun ke-3, saat ada customer enterprise yang minta SLA low latency dengan path disjoint, tim mengaktifkan SR sebagai Opsi 2 di 2 PE dan 2 P saja, bukan di seluruh core.

Ini adalah pola yang benar: Opsi 1 dulu yang stabil, Opsi 2 hanya untuk kebutuhan spesifik.

### G.4 Design OSPF untuk Studi Kasus Ini

**Aturan design OSPF yang dipakai:**

- **Satu area, Area 0:** Jangan buat area 1, 2, atau NSSA untuk core. Area tambahan hanya menambah kompleksitas dan membuat troubleshooting lebih sulit. Area tambahan hanya untuk PE-CE, bukan untuk core.
- **Router ID selalu Loopback0:** Jangan biarkan OSPF memilih otomatis.
- **Loopback passive:** Loopback diiklankan, tetapi tidak mengirim hello.
- **Link point-to-point:** Semua link core menggunakan `network point-to-point`, tanpa DR dan BDR.
- **Metric simetris:** Metric pergi dan pulang harus sama. Jika Jakarta ke Surabaya metric 20, maka Surabaya ke Jakarta juga metric 20. Metric asimetris membuat TI-LFA dan troubleshooting menjadi sulit.
- **Summarisasi tidak perlu di core:** Core hanya Loopback dan link, jadi tidak ada yang perlu disummarisasi. Summarisasi hanya di PE-CE.

**IP Plan OSPF untuk studi kasus:**

| Node | Loopback0 | POP | Prefix SID jika SR dipakai nanti |
| --- | --- | --- | --- |
| PE-JKT-01 | 10.0.0.1/32 | Jakarta | 1 |
| PE-JKT-02 | 10.0.0.2/32 | Jakarta | 2 |
| PE-SBY-01 | 10.0.0.3/32 | Surabaya | 3 |
| PE-BDG-01 | 10.0.0.4/32 | Bandung | 4 |
| P-JKT-01 | 10.0.0.11/32 | Jakarta | 11 |
| P-JKT-02 | 10.0.0.12/32 | Jakarta | 12 |
| P-SBY-01 | 10.0.0.13/32 | Surabaya | 13 |
| P-BDG-01 | 10.0.0.14/32 | Bandung | 14 |
| RR-JKT-01 | 10.0.0.101/32 | Jakarta | 101 |
| RR-SBY-01 | 10.0.0.102/32 | Surabaya | 102 |

| Link | IP | Metric OSPF | Keterangan |
| --- | --- | --- | --- |
| PE-JKT-01 ke P-JKT-01 | 10.255.0.0/31 | 10 | Link lokal POP |
| P-JKT-01 ke P-JKT-02 | 10.255.0.2/31 | 10 | Link lokal POP |
| P-JKT-01 ke P-SBY-01 | 10.255.0.4/31 | 50 | Link inter-POP Jakarta Surabaya |
| P-JKT-01 ke P-BDG-01 | 10.255.0.6/31 | 30 | Link inter-POP Jakarta Bandung |
| P-SBY-01 ke P-BDG-01 | 10.255.0.8/31 | 40 | Link inter-POP Surabaya Bandung |

Metric inter-POP lebih besar dari metric lokal. Ini membuat traffic lokal POP tidak lewat POP lain, dan traffic inter-POP memilih path yang paling dekat.

### G.5 Opsi 1 Utama: OSPF + LDP + BGP, Configuration Lengkap Studi Kasus

Ini adalah configuration yang dipakai di produksi untuk ISP studi kasus. SR belum aktif sama sekali.

**PE-JKT-01, Opsi 1:**

```
hostname PE-JKT-01
!
vrf CUSTOMER-A
 rd 65000:100
 address-family ipv4 unicast
 import route-target 65000:100
 export route-target 65000:100
 import route-target 65000:100 evpn
 export route-target 65000:100 evpn
 !
!
interface Loopback0
 ipv4 point-to-point
 ipv4 address 10.0.0.1/32
!
interface HundredGigE0/0/0/0
 description TO-P-JKT-01-Hu0/0/0/0
 ipv4 point-to-point
 ipv4 address 10.255.0.0/31
 no shutdown
!
interface GigabitEthernet0/0/0/1
 description TO-CE1-CUSTOMER-A
 vrf CUSTOMER-A
 ipv4 point-to-point
 ipv4 address 192.168.1.1/30
 no shutdown
!
router ospf 100
 router-id 10.0.0.1
 nsr
 nsf cisco
 timers throttle spf 50 100 5000
 timers throttle lsa all 50 100 5000
 area 0
 interface Loopback0
 passive enable
 !
 interface HundredGigE0/0/0/0
 network point-to-point
 cost 10
 hello-interval 3
 dead-interval 9
 bfd fast-detect
 !
 !
!
mpls ldp
 router-id 10.0.0.1
 nsr
 graceful-restart
 session protection
 interface HundredGigE0/0/0/0
 !
!
route-policy PASS-ALL
 pass
end-policy
!
router bgp 65000
 bgp router-id 10.0.0.1
 bgp graceful-restart
 nsr
 neighbor 10.0.0.101
 remote-as 65000
 description RR-JKT-01
 password clear BGP-SECRET-RR1
 ttl-security hops 1
 update-source Loopback0
 address-family ipv4 unicast
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 address-family vpnv4 unicast
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 address-family l2vpn evpn
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 !
 neighbor 10.0.0.102
 remote-as 65000
 description RR-SBY-01
 password clear BGP-SECRET-RR2
 ttl-security hops 1
 update-source Loopback0
 address-family ipv4 unicast
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 address-family vpnv4 unicast
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 address-family l2vpn evpn
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 !
 vrf CUSTOMER-A
 rd 65000:100
 address-family ipv4 unicast
 label mode per-vrf
 redistribute connected
 !
 neighbor 192.168.1.2
 remote-as 65001
 description CE1-CUSTOMER-A
 address-family ipv4 unicast
 route-policy PASS-ALL in
 route-policy PASS-ALL out
 !
 !
 !
!
evpn
 evi 100
 bgp
 route-target import 65000:100
 route-target export 65000:100
 !
 !
!
commit label STUDI-KASUS-OSPF-LDP-PE-JKT-01 comment Opsi 1 OSPF LDP PE-JKT-01
end
```

**P-JKT-01, Opsi 1:**

```
hostname P-JKT-01
!
interface Loopback0
 ipv4 point-to-point
 ipv4 address 10.0.0.11/32
!
interface HundredGigE0/0/0/0
 description TO-PE-JKT-01
 ipv4 point-to-point
 ipv4 address 10.255.0.1/31
 no shutdown
!
interface HundredGigE0/0/0/1
 description TO-P-JKT-02
 ipv4 point-to-point
 ipv4 address 10.255.0.2/31
 no shutdown
!
interface HundredGigE0/0/0/2
 description TO-P-SBY-01
 ipv4 point-to-point
 ipv4 address 10.255.0.4/31
 no shutdown
!
router ospf 100
 router-id 10.0.0.11
 nsr
 nsf cisco
 area 0
 interface Loopback0
 passive enable
 !
 interface HundredGigE0/0/0/0
 network point-to-point
 cost 10
 hello-interval 3
 dead-interval 9
 bfd fast-detect
 !
 interface HundredGigE0/0/0/1
 network point-to-point
 cost 10
 hello-interval 3
 dead-interval 9
 bfd fast-detect
 !
 interface HundredGigE0/0/0/2
 network point-to-point
 cost 50
 hello-interval 3
 dead-interval 9
 bfd fast-detect
 !
 !
!
mpls ldp
 router-id 10.0.0.11
 nsr
 graceful-restart
 session protection
 interface HundredGigE0/0/0/0
 !
 interface HundredGigE0/0/0/1
 !
 interface HundredGigE0/0/0/2
 !
!
commit label STUDI-KASUS-OSPF-LDP-P-JKT-01 comment Opsi 1 OSPF LDP P-JKT-01
end
```

**Verification Opsi 1:**

```
show ospf neighbor
show ospf interface HundredGigE0/0/0/0
show route ipv4 10.0.0.2/32
show mpls ldp neighbor
show mpls ldp bindings 10.0.0.2/32
show mpls forwarding
show bgp vpnv4 unicast summary
show bgp l2vpn evpn summary
ping vrf CUSTOMER-A 192.168.2.2 source 192.168.1.1
traceroute vrf CUSTOMER-A 192.168.2.2 source 192.168.1.1
```

Jika semua `show` di atas sehat dan ping VRF berhasil, Opsi 1 sudah cukup untuk produksi. Jangan aktifkan SR hanya karena SR sedang tren.

### G.6 Opsi 2: Segment Routing sebagai Alternatif, Kapan Dipakai dan Kapan Tidak

**Gunakan SR sebagai Opsi 2 jika:**

- Ada customer yang minta SR Policy dengan path eksplisit yang tidak dapat dipenuhi LDP, misalnya low latency via path tertentu
- Ada kebutuhan TI-LFA yang tidak dapat dipenuhi LFA klasik di topology tertentu
- Ada kebutuhan Flex-Algo untuk low latency atau disjoint
- Ada kebutuhan migrasi ke SRv6 untuk 5G atau DCI di masa depan, dan tim sudah siap
- Core sudah di atas 100 node dan LDP mulai terasa stateful dan sulit di-troubleshoot

**Jangan gunakan SR jika:**

- ISP masih di bawah 30 node dan LDP sudah stabil
- Tim belum pernah lab SR di XRv 9000, dan belum pernah uji failover SR
- Tidak ada customer yang minta TE atau SLA khusus yang membutuhkan SR Policy
- Topology masih single link atau ring sederhana, TI-LFA tidak akan memiliki backup path
- Budget dan waktu maintenance terbatas, migrasi LDP ke SR membutuhkan jendela maintenance yang tidak kecil

**Perbandingan Opsi 1 dan Opsi 2:**

| Aspek | Opsi 1: OSPF + LDP | Opsi 2: OSPF + SR-MPLS atau SRv6 |
| --- | --- | --- |
| Kompleksitas operasi | Rendah, tim sudah paham | Lebih tinggi, membutuhkan pemahaman SID, SRGB, SR Policy |
| Traffic Engineering | Tidak ada, hanya IGP shortest path | Native, SR Policy, Flex-Algo |
| Fast Reroute | LFA, terbatas di topology tertentu | TI-LFA, lebih kuat dan lebih deterministik |
| Scale label | Label dinamis LDP, per FEC | Label deterministik dari SID, lebih mudah dibaca |
| Troubleshooting | Label berubah per hop, sulit trace | Label deterministik, mudah trace dari SID |
| Kebutuhan training | Minimal | Membutuhkan training SR dan lab XRv 9000 |
| Risiko migrasi | Tidak ada, sudah berjalan | Ada risiko saat migrasi, membutuhkan sr-prefer dan rollback plan |
| Rekomendasi guide ini | Utama, untuk semua ISP | Alternatif, hanya jika ada kebutuhan TE atau SRv6 |

### G.7 Cara Mengaktifkan Opsi 2 di Atas Opsi 1 Tanpa Downtime

Ini adalah cara yang dipakai di studi kasus saat ISP akhirnya membutuhkan SR Policy untuk satu customer low latency. SR diaktifkan hanya di 2 PE dan 2 P yang dilewati customer tersebut, bukan di seluruh core sekaligus.

**Langkah 1: Aktifkan SR berdampingan dengan LDP di PE-JKT-01**

```
configure
segment-routing
 global-block 16000 23999
!
mpls label range table 0 24000 1048575
!
router ospf 100
 segment-routing mpls
 segment-routing forwarding mpls
 segment-routing mpls sr-prefer
 area 0
 interface Loopback0
 prefix-sid index 1
 !
 !
!
commit label SR-OPSI2-PE-JKT-01 comment Aktifkan SR sebagai Opsi 2 di PE-JKT-01
end
```

**Langkah 2: Aktifkan SR di P-JKT-01**

```
configure
segment-routing
 global-block 16000 23999
!
router ospf 100
 segment-routing mpls
 segment-routing forwarding mpls
 area 0
 interface Loopback0
 prefix-sid index 11
 !
 !
!
commit label SR-OPSI2-P-JKT-01 comment Aktifkan SR sebagai Opsi 2 di P-JKT-01
end
```

**Langkah 3: Validation SR sudah aktif, tetapi LDP masih berjalan**

```
show ospf segment-routing
show mpls forwarding
show cef ipv4 10.0.0.2/32 detail
show bgp vpnv4 unicast summary
ping vrf CUSTOMER-A 192.168.2.2 source 192.168.1.1
```

Jika `show mpls forwarding` menampilkan label SR 16002 untuk PE tujuan, dan ping VRF masih berhasil, maka SR sudah aktif sebagai Opsi 2. LDP masih aktif sebagai fallback.

**Langkah 4: Buat SR Policy hanya untuk customer yang membutuhkan**

```
configure
segment-routing
 traffic-eng
 segment-list SL-LOWLAT-JKT-SBY
 index 10 mpls label 16011
 index 20 mpls label 16013
 index 30 mpls label 16003
 !
 policy POL-LOWLAT-CUSTOMER-B
 color 128 end-point ipv4 10.0.0.3
 candidate-paths
 preference 100
 explicit segment-list SL-LOWLAT-JKT-SBY
 !
 !
 !
 !
!
router bgp 65000
 vrf CUSTOMER-B-LOWLAT
 address-family ipv4 unicast
 segment-routing policy color 128
 !
 !
!
commit label SR-POLICY-OPSI2 comment SR Policy hanya untuk CUSTOMER-B low latency
end
```

Hanya VRF CUSTOMER-B-LOWLAT yang menggunakan SR Policy. VRF lain tetap menggunakan LDP dan IGP shortest path. Ini adalah kunci Opsi 2: SR tidak memaksa semua customer pindah, hanya customer yang membutuhkan.

**Langkah 5: Jika SR sudah stabil, baru pertimbangkan nonaktifkan LDP**

Jangan nonaktifkan LDP di seluruh core. Nonaktifkan LDP per node, mulai dari P, dengan `no mpls ldp` dan validation satu minggu per node. Jika ada masalah, rollback satu node saja.

### G.8 Studi Kasus Troubleshooting OSPF

**Kasus 1: OSPF neighbor tidak terbentuk**

Gejala: `show ospf neighbor` kosong. Periksa: MTU interface sama atau tidak, `network point-to-point` sudah sama di kedua sisi, hello dan dead interval sama, BFD tidak memblokir, ACL tidak memblokir multicast OSPF 224.0.0.5 dan 224.0.0.6.

```
show ospf interface HundredGigE0/0/0/0
show interfaces HundredGigE0/0/0/0
show logging last 50
show ospf neighbor detail
```

**Kasus 2: OSPF terbentuk, tetapi route Loopback tidak ada**

Gejala: Neighbor Full, tetapi `show route ipv4 10.0.0.2/32` tidak ada. Periksa: Loopback di `passive enable` atau tidak, Loopback sudah di area 0 atau belum, filter OSPF tidak memblokir.

**Kasus 3: OSPF flapping**

Gejala: Neighbor naik turun. Periksa: BFD, LPTS drop untuk OSPF, QoS class CONTROL policed terlalu rendah, MTU, dan interface error.

```
show lpts pifib policer
show policy-map interface HundredGigE0/0/0/0 output
show bfd session
show interfaces HundredGigE0/0/0/0
```

**Kasus 4: Path OSPF tidak sesuai design**

Gejala: Traffic Jakarta ke Surabaya lewat Bandung, padahal seharusnya langsung. Periksa: `show route ipv4 10.0.0.3/32` dan `show ospf database`. Periksa metric: metric pergi dan pulang harus simetris. Jika metric Jakarta ke Surabaya 50, tetapi Surabaya ke Jakarta 10, maka path akan asimetris.

```
show route ipv4 10.0.0.3/32 detail
show ospf database router
show ospf interface
```
