# Serie A 2026/27 — Dashboard Power BI (PBIP)

Klasemen dan statistik pemain Serie A musim ini, dengan **foto pemain diambil
otomatis dari Wikipedia** saat refresh.

---

## Cara membuka

1. **Power BI Desktop** → File → Open → `SerieA_2026_27.pbip`
   *(PBIP harus aktif: File → Options → Preview features → "Power BI Project (.pbip) save option")*
2. **Home → Transform data → Edit parameters**
   `DataFolderPath` → path absolut ke folder `Data` di sebelah README ini, mis.
   `C:\PBIP\SerieA_2026_27\Data`
3. **Home → Refresh**

Saat refresh pertama Power BI akan menanyakan privacy level untuk
`en.wikipedia.org`. Pilih **Public** (atau Options → Privacy → *Ignore the
Privacy Levels*). Tanpa itu, kolom foto akan kosong — bukan error, hanya blank,
dan tooltip akan menampilkan placeholder.

---

## Foto pemain: bagaimana caranya

Tidak ada URL gambar yang di-hardcode. Model punya satu fungsi M bernama
`FnWikipediaImage` yang memanggil REST API Wikipedia:

```
https://en.wikipedia.org/api/rest_v1/page/summary/{judul_artikel}
```

lalu mengambil `thumbnail.source` dari JSON-nya — yaitu gambar utama artikel itu.

Jadi yang tersimpan di CSV hanya **judul artikel Wikipedia** (`WikiPage`),
misalnya `Donyell_Malen`. Kalau Wikipedia mengganti fotonya, refresh berikutnya
ikut berubah. Kalau ada pemain baru, cukup tambah satu baris di CSV dengan
judul artikelnya — tidak perlu cari URL gambar.

**Kenapa cara ini, bukan salin URL Commons satu per satu:**
menebak nama file di `upload.wikimedia.org` hampir selalu salah, dan URL yang
salah berarti gambar rusak. Judul artikel jauh lebih mudah diverifikasi.

Tiga hal yang membuat ini tidak rapuh:

| Hal | Alasan |
|---|---|
| `RelativePath` di `Web.Contents` | Pola yang aman untuk refresh di Power BI Service — URL dinamis biasa akan gagal di sana |
| `try ... otherwise null` | Satu judul artikel yang salah tidak menggagalkan seluruh refresh |
| `COALESCE` di measure `[Player Photo]` | Pemain tanpa foto di Wikipedia menampilkan placeholder netral, bukan kotak gambar rusak |

`DimPlayer[PlayerImageURL]` diberi `dataCategory: ImageUrl`. Ini yang membuat
Power BI merender gambar, bukan teks mentah link-nya.

### Lisensi foto

Foto di Wikipedia umumnya **CC BY-SA**, yang mewajibkan atribusi saat
dipublikasikan ulang. Tooltip sudah memuat baris kredit, dan `DimPlayer` punya
kolom `WikiArticleURL` (dataCategory `WebUrl`) untuk menautkan ke artikel
sumbernya. Untuk dashboard internal ini cukup; kalau dipublikasikan ke luar,
periksa lisensi tiap foto di halaman file Commons-nya.

---

## Data

Sumber: artikel Wikipedia **2026–27 Serie A** dan halaman klub terkait.

| Tabel | Isi | Per tanggal |
|---|---|---|
| `DimClub` | 20 klub: kota, stadion, kapasitas, manajer, kapten, warna | Awal musim |
| `FactStandings` | Klasemen: Pld/W/D/L/GF/GA/Pts | **6 September 2026** |
| `DimPlayer` | 24 pemain (pencetak gol + kiper clean sheet) | — |
| `FactPlayerStat` | Gol dan clean sheet per pemain | **5 September 2026** (gol), 23 Agustus 2026 (clean sheet) |

### Dua hal penting soal data ini

**1. Musim baru berjalan ~3 pertandingan per klub.**
Klub belum memainkan jumlah laga yang sama — ada yang 2, ada yang 3. Jadi
membandingkan **total poin** antar klub itu menyesatkan. Semua measure
perbandingan di model ini memakai basis **per laga**:
`[Points per Match]`, `[Goals For per Match]`, `[Win Rate]`, dan
`[Projected Points (38 Matches)]` yang mengekstrapolasi ke musim penuh.
Measure `[Data As Of]` ditaruh di kanvas supaya tanggalnya selalu terlihat.

**2. Halaman Wikipedia-nya sendiri tidak konsisten.**
Ringkasan di atas artikel menyebut *24 pertandingan, 65 gol*, tetapi tabel
klasemennya menjumlah *27 pertandingan, 76 gol* — bagian-bagian di satu halaman
Wikipedia diperbarui pada waktu yang berbeda. Saya memakai **tabel klasemen**,
karena tabel itu konsisten secara internal: GF = GA = 76, W = L = 24, jumlah
Drawn genap, dan W+D+L = Played di setiap baris. Semua itu diperiksa oleh
validator. Angka di ringkasan artikel saya abaikan.

Kalau butuh angka resmi terbaru, `legaseriea.it` adalah sumber utamanya.

---

## Model

Star schema, 3 relationship, semua single-direction many-to-one:

```
DimClub (20) ──┬── FactStandings (20)
               └── FactPlayerStat (24) ──── DimPlayer (24)
```

24 measure, dikelompokkan:

| Kelompok | Contoh |
|---|---|
| Dasar | `Total Points`, `Matches Played`, `Goals For`, `Total Goals` |
| Per laga | `Points per Match`, `Goals For per Match`, `Win Rate` |
| Proyeksi | `Projected Points (38 Matches)` |
| Tooltip | `Player Photo`, `Player Meta`, `Club Colour`, `Headline Stat` |

`[Headline Stat]` menyesuaikan diri dengan posisi: kiper ditampilkan dengan
clean sheet, pemain lain dengan gol — jadi satu baris tooltip melayani keduanya
tanpa menampilkan "0 goals" untuk kiper.

`DimPlayer[Position]` diurutkan Goalkeeper → Defender → Midfielder → Forward
lewat `PositionSort`, yang **dihitung di Power Query, bukan DAX**. Calculated
column DAX yang membaca `[Position]` tidak boleh sekaligus menjadi
`sortByColumn`-nya — itu circular dependency.

---

## Halaman

**Overview** (1280 × 720) — KPI per laga, klasemen lengkap, grafik pencetak gol
dan clean sheet. Dua grafik terakhir memakai tooltip foto.

**Tooltip — Pemain** (320 × 300, tab tersembunyi) — foto, nama, klub/posisi/
negara, statistik utama, kredit foto. Satu halaman ini melayani kedua grafik.

---

## Struktur file

```
SerieA_2026_27.pbip                    ← buka ini
Data/
  DimClub.csv          20 klub
  DimPlayer.csv        24 pemain  (kolom WikiPage = judul artikel Wikipedia)
  FactStandings.csv    20 baris klasemen
  FactPlayerStat.csv   24 baris statistik
SerieA_2026_27.SemanticModel/definition/
  model.tmdl  relationships.tmdl  database.tmdl
  expressions.tmdl     ← parameter DataFolderPath + fungsi FnWikipediaImage
  tables/              DimClub, DimPlayer, FactStandings, FactPlayerStat
  roles/               Viewer
SerieA_2026_27.Report/
  report.json          Overview + TooltipPemain
  StaticResources/SharedResources/BaseThemes/SerieATheme.json
```
