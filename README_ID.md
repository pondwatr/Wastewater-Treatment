# Pengontrol Dewatering Separator Screw Press (Omron CP1E)

[![English](https://img.shields.io/badge/Language-English-blue)](readme.md)
*Tersedia dalam bahasa lain: [🇬🇧 English Version](readme.md)*

**Program ini dibuat untuk memenuhi mata kuliah Otomasi Industri, BS1TE-48-REG-02 dalam binaan bapak Dr. Muhammad Ary Murti S.T., Ph.D pada Universitas Telkom.** 

Program PLC untuk mesin **Screw Press Dewatering Separator** ("Auger Press / Pemadat Limbah") ini dikembangkan menggunakan **Omron CX-Programmer**.

Pengontrol ini mengotomatiskan proses pemisahan fasa padat-cair (dewatering) berdasarkan penelitian ilmiah pada pemisah kotoran ternak/slurry (*Senfter et al., Separations 2024*) serta mengatasi masalah operasional di lapangan dari analisis bongkar unit mesin riil (penyumbatan saringan/screen blinding, otomatisasi solenoid pencuci, dan kemacetan sumbat ampas).

---

## 📌 Gambaran Proyek & Prinsip Kerja Fisik

Mesin screw press memisahkan filtrat cair dari bubur kental (slurry/manure) menggunakan ulir konveyor (auger) yang berputar di dalam tabung saringan kawat baji (*wedge-wire screen*). Efisiensi pemerasan sangat bergantung pada **tekanan balik mekanis (*counter-pressure*)** yang dijaga pada pintu penahan ampas (*choke flap/cone*) di ujung keluaran:
* **Tekanan balik terlalu rendah:** Bubur basah yang belum terperas akan lolos keluar begitu saja.
* **Tekanan balik terlalu tinggi:** Saringan tersumbat, torsi poros auger melonjak tajam, dan motor listrik mengalami macet (*jamming/stall*).

Pengontrol ini mengotomatiskan seluruh proses menggunakan umpan balik analog dari sensor aliran secara *closed-loop* dan kendali arah sederhana untuk aktuator silinder elektrik.

---

## ⚙️ Arsitektur Perangkat Keras

| Komponen | Spesifikasi / Model Industri | Antarmuka PLC | Peran dalam Proses |
| :--- | :--- | :--- | :--- |
| **PLC** | **Omron CP1E** | CPU internal + Modul Analog Bawaan | Pengendali Utama (Master Controller) |
| **Flow Sensor** | **Allen-Bradley 839E** Solid-State Flow Transmitter | Loop Arus 4–20 mA ke `CIO 90` | Memantau laju aliran filtrat; mendeteksi penyumbatan saringan & kondisi kering (*dry run*) |
| **Actuator** | **Allen-Bradley MPAR** Kinetix Electric Cylinder | Output diskrit: `100.02` (Maju), `100.03` (Mundur) | Mengatur posisi dan tekanan balik pintu penahan ampas secara dinamis |
| **Motor Auger** | Motor Induksi 3-Fasa / Inverter (VFD) | Output digital `100.00` | Memutar poros ulir utama untuk memeras dan mendorong ampas |
| **Flush Solenoid** | Katup Solenoid Air 2-Arah | Output digital `100.01` | Membuka nozzle semprot pencuci saringan untuk pembersihan & pembilasan akhir |

---

## 🗺️ Tabel Alokasi I/O & Memori PLC

### Input Fisik (`CIO 0` & `CIO 90`)
| Alamat | Simbol | Tipe | Deskripsi |
| :--- | :--- | :--- | :--- |
| `0.00` | `PB_START` | Digital (NO) | Tombol Tekan Start Siklus |
| `0.01` | `PB_STOP` | Digital (NC) | Tombol Tekan Stop Siklus |
| `0.02` | `ESTOP_OK` | Digital (NC) | Kontak Sirkuit Emergency Stop Sehat/Aman |
| `0.03` | `AUGER_OL_OK` | Digital (NC) | Kontak Proteksi Termal Overload Motor / Status Siap VFD |
| `CIO 90`| `FLOW_ANALOG` | Analog In | Nilai Mentah Laju Aliran Sensor (0–6000 setara 4–20 mA) |

### Setpoint Memori & Bit Internal (`D` & `W`)
| Alamat | Simbol | Deskripsi |
| :--- | :--- | :--- |
| `D10` | `FLOW_SP_LOW` | Nilai Ambang Batas Bawah Aliran / Low Flow (misal: `&600` = 10%) |
| `W0.00` | `SYS_RUN` | Pengunci Status Sistem Berjalan (Master Latch) |
| `W0.01` | `SEQ_PRECHOKE` | Fase 1: Tahap Pembentukan Sumbat Awal (Pre-Choke) |
| `W0.02` | `SEQ_DEWATER` | Fase 2: Operasi Pemerasan Normal (Dewatering Run) |
| `W0.03` | `JAM_RELIEF` | Fase 3: Rutinitas Pelepasan Tekanan Macet Otomatis |
| `W0.04` | `SEQ_SHUTDOWN` | Fase 4: Siklus Pembilasan & Pengurasan Saat Berhenti |
| `W0.05` | `ALM_LOW_FLOW` | Bendera Alarm Aliran Rendah / Saringan Tersumbat |

### Output Fisik (`CIO 100`)
| Alamat | Simbol | Deskripsi |
| :--- | :--- | :--- |
| `100.00` | `AUGER_RUN` | Kontaktor Motor Auger / Perintah Jalan VFD |
| `100.01` | `FLUSH_SOLENOID` | Katup Solenoid Air Pencuci Saringan Wedge-Wire |
| `100.02` | `ACTUATOR_EXT` | Perintah Silinder Elektrik Maju (Menambah Tekanan Choke) |
| `100.03` | `ACTUATOR_RET` | Perintah Silinder Elektrik Mundur (Melepas Tekanan Choke) |
| `100.04` | `ALARM_BEACON` | Lampu / Sirene Indikator Alarm Gangguan |

### Timer
| Timer | Nilai Set | Durasi | Fungsi |
| :--- | :--- | :--- | :--- |
| `TIM 000` | `#50` | 5,0 detik | Jeda stabilisasi aliran saat pertama kali mulai memeras |
| `TIM 001` | `#40` | 4,0 detik | Durasi penarikan mundur aktuator saat terjadi kemacetan motor |
| `TIM 002` | `#150` | 15,0 detik | Pembilasan air dan pengurasan sisa ampas saat tombol Stop ditekan |
| `TIM 003` | `#30` | 3,0 detik | Pergerakan awal aktuator maju untuk menyetel celah awal penahan ampas |

---

## 🔄 Diagram Alur & Urutan Operasional

```
                     [ Siaga / E-Stop Aman ]
                               │ (PB_START 0.00)
                               ▼
                   [ Fase 1: Pre-Choke ]
       (Aktuator maju selama 3,0 detik untuk menutup celah awal)
                               │ (TIM 003 Selesai)
                               ▼
                [ Fase 2: Pemerasan Normal ]
      (Auger berputar; Sensor Aliran memantau laju cairan)
          │                                         │
 (Overload Motor 0.03 = 0)              (Aliran < Setpoint D10)
          │                                         │
          ▼                                         ▼
[ Fase 3: Pelepasan Macet ]             [ Semprot Solenoid Cuci ]
(Aktuator mundur 4,0 detik untuk         (Membilas saringan agar
 buang tekanan; motor pulih)              filtrat kembali lancar)
                               │ (PB_STOP 0.01)
                               ▼
                 [ Fase 4: Pengurasan Stop ]
      (Aktuator terbuka penuh; Solenoid Air & Motor menyala
       selama 15,0 detik agar ampas tidak mengeras jadi semen)
```

---

## 🛠️ Diagram Ladder (Daftar Instruksi Mnemonic CX-Programmer)

```text
// Rung 0: Pengunci Master Start/Stop dengan Interlock Keselamatan
LD      0.02
AND     0.03
AND-NOT 0.01
LD      0.00
OR      W0.00
OR      W0.04
AND-LD
OUT     W0.00

// Rung 1: Fase 1 - Pembentukan Celah Awal (Pre-Choke)
LD      W0.00
AND-NOT TIM003
AND-NOT W0.02
AND-NOT W0.04
OUT     W0.01
TIM     003     #0030

// Rung 2: Fase 2 - Operasi Pemerasan Berjalan
LD      W0.00
AND     TIM003
AND-NOT W0.03
AND-NOT W0.04
OR      W0.02
OUT     W0.02

// Rung 3: Timer Penundaan Stabilisasi Aliran
LD      W0.02
TIM     000     #0050

// Rung 4: Komparasi Analog Aliran & Pemicu Alarm
LD      W0.02
AND     TIM000
AND<    90      D10
OUT     W0.05

// Rung 5: Fase 3 - Rutinitas Pelepasan Beban Macet Otomatis
LD      W0.02
AND-NOT 0.03
LD      W0.03
AND-NOT TIM001
OR-LD
OUT     W0.03
TIM     001     #0040

// Rung 6: Fase 4 - Penghentian Terkendali & Pengurasan Air
LD      0.01
AND     W0.00
AND-NOT TIM002
LD      W0.04
AND-NOT TIM002
OR-LD
OUT     W0.04
TIM     002     #0150

// Rung 7: Output Fisik - Motor Auger
LD      W0.02
AND-NOT W0.03
OR      W0.04
OUT     100.00

// Rung 8: Output Fisik - Solenoid Air Pembersih
LD      W0.04
LD      W0.02
AND     W0.05
OR-LD
OUT     100.01

// Rung 9: Output Fisik - Perintah Aktuator Maju
LD      W0.01
OUT     100.02

// Rung 10: Output Fisik - Perintah Aktuator Mundur
LD      W0.03
OR      W0.04
OUT     100.03

// Rung 11: Output Fisik - Lampu / Sirene Alarm
LD      W0.03
OR      W0.05
OUT     100.04

END
```

---

## 🧪 Panduan Pengujian & Simulasi (Langkah Demi Langkah CX-Simulator)

Ikuti langkah-langkah di bawah ini untuk menguji seluruh logika ladder secara langsung di komputer tanpa perlu perangkat keras fisik.

### Pintasan Keyboard & Jendela Penting
* **Kompilasi Program:** `Ctrl + F7` *(Pastikan Output Window di bawah menampilkan `0 errors, 0 warnings`)*
* **Menjalankan Simulator:** `Ctrl + Shift + W` *(Memulai virtual CPU CP1E via Work Online Simulator)*
* **Pohon Proyek (Project Workspace):** `Alt + 1` *(Atau melalui menu `View` > `Windows` > `Project Workspace`)*
* **Jendela Pantau (Watch Window):** `Alt + 3` *(Atau melalui menu `View` > `Windows` > `Watch`)*
* **Mode Operasi PLC:** Pastikan status di kanan bawah berstatus **MONITOR** atau **RUN** *(Tekan `Ctrl + M` jika masih dalam mode PROGRAM)*

---

### Langkah 1: Persiapan Awal (Konfigurasi Watch Window & Kontak Pengaman)

Sebelum menekan tombol Start, atur nilai setpoint analog awal dan kondisi pengaman:

1. **Buka Watch Window (`Alt + 3`)**:
   * Klik baris kosong di bawah kolom **Name**, ketik `90` (Input Analog Sensor Aliran), lalu tekan `Enter`.
   * Klik baris berikutnya, ketik `D10` (Batas Ambang Aliran Rendah), lalu tekan `Enter`.
   * Klik baris berikutnya, ketik `W0.05` (Bendera Alarm Aliran Rendah), lalu tekan `Enter`.
2. **Isi Nilai di Watch Window**:
   * Klik dua kali kolom **Value** pada baris **`D10`**, ketik `600` (atau `&600` untuk desimal), lalu tekan `Enter`.
   * Klik dua kali kolom **Value** pada baris **`90`**, ketik `1200` (atau `&1200`), lalu tekan `Enter` (kondisi awal aliran normal > 600).
3. **Aktifkan Kontak Pengaman Normally-Closed (NC)**:
   * Pada Rung 0, klik dua kali kontak **`0.02` (`ESTOP_OK`)** $\rightarrow$ ubah nilai menjadi `1` $\rightarrow$ `Enter`.
   * Pada Rung 0, klik dua kali kontak **`0.03` (`AUGER_OL_OK`)** $\rightarrow$ ubah nilai menjadi `1` $\rightarrow$ `Enter`.

---

### Langkah 2: Uji Coba Start & Tahap Pre-Choke (Fase 1 $\rightarrow$ 2)

1. Klik dua kali kontak **`0.00` (`PB_START`)**, ubah nilainya menjadi `1`, tekan `Enter`.
2. Segera klik dua kali **`0.00`** kembali dan ubah ke `0` (simulasi menekan dan melepas tombol sesaat/momentary).
3. **Verifikasi:**
   * `W0.00` (`SYS_RUN`) menyala hijau dan terkunci aktif.
   * `W0.01` (`SEQ_PRECHOKE`) menyala.
   * Koil output **`100.02` (`ACTUATOR_EXT`)** menyala untuk mendorong pintu penahan ampas ke posisi awal.
   * Timer **`TIM 003`** mulai menghitung mundur selama 3,0 detik (`#30` $\rightarrow$ `0`).
4. **Setelah `TIM 003` mencapai 0:**
   * `100.02` (`ACTUATOR_EXT`) mati.
   * `W0.02` (`SEQ_DEWATER`) menyala dan mengunci.
   * Koil output **`100.00` (`AUGER_RUN`)** menyala (motor auger mulai memeras).
   * Timer **`TIM 000`** mulai menghitung mundur jeda stabilisasi aliran selama 5,0 detik (`#50` $\rightarrow$ `0`).

---

### Langkah 3: Uji Coba Alarm Aliran Rendah & Saringan Tersumbat (Fase 2)

1. Tunggu hingga timer **`TIM 000`** selesai (`T000` menyala hijau).
2. Pindah ke **Watch Window (`Alt + 3`)**:
   * Klik dua kali kolom **Value** pada alamat **`90`**.
   * Ubah nilainya dari `1200` turun menjadi **`200`** (atau `&200`), lalu tekan `Enter`.
3. **Verifikasi:**
   * Kondisi perbandingan `90 < D10` ($200 < 600$) terpenuhi (TRUE).
   * Pada Watch Window dan diagram ladder, **`W0.05` (`ALM_LOW_FLOW`)** menyala (indikator titik hijau pada koil).
   * Koil output **`100.01` (`FLUSH_SOLENOID`)** menyala untuk membuka nozzle pencuci saringan.
   * Koil output **`100.04` (`ALARM_BEACON`)** menyala.
4. **Kembalikan Aliran Normal:**
   * Di Watch Window, ubah nilai `90` kembali menjadi **`1200`**.
   * `W0.05`, `100.01`, dan `100.04` akan langsung padam kembali.

---

### Langkah 4: Uji Coba Beban Berlebih & Pelepasan Macet Otomatis (Fase 3)

1. Saat proses dewatering berjalan (`W0.02` aktif dan `100.00` menyala), simulasikan motor auger mengalami beban berlebih:
   * Klik dua kali kontak **`0.03` (`AUGER_OL_OK`)**, ubah nilainya menjadi `0` (simulasi trip thermal overload/jamming).
2. **Verifikasi:**
   * Output **`100.00` (`AUGER_RUN`)** langsung mati seketika guna melindungi motor.
   * Bit internal `W0.03` (`JAM_RELIEF`) menyala aktif.
   * Koil output **`100.03` (`ACTUATOR_RET`)** menyala untuk menarik mundur pintu penahan dan melepas sumbatan ampas.
   * Timer **`TIM 001`** menghitung mundur selama 4,0 detik (`#40` $\rightarrow$ `0`).
3. **Reset Overload:**
   * Kembalikan kontak **`0.03`** ke nilai `1` (simulasi proteksi thermal di-reset).
   * Setelah `TIM 001` selesai, `W0.03` dan `ACTUATOR_RET` mati, mengembalikan sistem ke proses pemerasan normal secara otomatis.

---

### Langkah 5: Uji Coba Penghentian & Pembilasan Pengurasan Akhir (Fase 4)

1. Klik dua kali kontak **`0.01` (`PB_STOP`)**, ubah ke `1`, lalu kembalikan ke `0`.
2. **Verifikasi:**
   * `W0.00` keluar dari mode jalan normal.
   * `W0.04` (`SEQ_SHUTDOWN`) menyala dan mengunci.
   * Koil output **`100.03` (`ACTUATOR_RET`)** menyala untuk membuka penuh saluran keluar ampas.
   * Koil output **`100.01` (`FLUSH_SOLENOID`)** dan **`100.00` (`AUGER_RUN`)** tetap menyala bersamaan.
   * Timer **`TIM 002`** menghitung mundur selama 15,0 detik (`#150` $\rightarrow$ `0`), membilas tuntas sisa lumpur kotoran dari dalam tabung saringan agar tidak mengering dan mengeras seperti semen.
3. **Setelah `TIM 002` selesai:**
   * Seluruh output padam dan mesin kembali aman ke mode Siaga (Standby).

---

## 📚 Referensi
* **Jurnal Penelitian:** Senfter, T., Schweiggl, I., Berger, M., Mayerl, C., Kofler, T., Kraxner, M., Steffens, A., & Pillei, M. (2024). *The Dewatering Performance of a Compact Screw Press Manure Separator for Non-Typical Substrates*. **Separations**, 11(1), 28. [https://doi.org/10.3390/separations11010028](https://doi.org/10.3390/separations11010028)
* **Video Referensi Teknis Lapangan:** AvE - *Fixing the Poop Screw* (Troubleshooting auger pemadat limbah industri & solenoid pencuci): [YouTube](https://youtu.be/6O5Lpm2iE44?t=374)

