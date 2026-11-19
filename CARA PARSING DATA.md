# Contekan Latihan Responsi PAM (Flutter)

Studi kasus: aplikasi Spaceflight News (login, 3 menu, list, filter, favorit, detail).
Dokumen ini berisi **apa yang harus diubah kalau API atau studi kasusnya berganti**, dan
**cara kerja parsing dari halaman list ke halaman detail**.

---

## 1. Peta file: satu file = satu tugas

```
lib/
├── main.dart                 // cek sesi login -> LoginPage / HomePage
├── models/content_item.dart  // bentuk data + fromJson/toJson   <- SERING DIUBAH
├── services/
│   ├── auth_service.dart     // register, login, logout (SharedPreferences)
│   ├── api_service.dart      // getList(menu), getDetail(menu, id) <- SERING DIUBAH
│   └── favorite_service.dart // favorit per akun (SharedPreferences + ValueNotifier)
├── pages/
│   ├── login.dart / register.dart
│   ├── home.dart             // daftar menu (_menus)            <- SERING DIUBAH
│   ├── list_page.dart        // fetch list, chip filter, thumbnail
│   ├── detail_page.dart      // fetch by id, tombol favorit, buka web
│   └── favorite_page.dart
└── widgets/content_tile.dart // (opsional) 1 item list
```

Kalau API baru: **yang hampir pasti disentuh hanya 3 file**: `content_item.dart`,
`api_service.dart`, `home.dart`. Halaman lain cukup menyesuaikan nama field.

---

## 2. Alur parsing: dari halaman list ke halaman detail

Tiga fase yang selalu sama: **Fetch -> Parse -> Render**.

```
[ListPage]
  1. FETCH   ApiService.getList('articles')  -> GET .../v4/articles/
  2. PARSE   String JSON --jsonDecode--> Map
             data['results'] (List) --map--> List<ContentItem>
  3. RENDER  ListView.builder menampilkan item.title, item.author, item.imageUrl
  4. TAP     Navigator.push(DetailPage(menu: item.type, id: item.id))
                                      |
                                      v   (yang dikirim cuma menu + id)
[DetailPage]
  5. FETCH   ApiService.getDetail(menu, id)  -> GET .../v4/articles/{id}/
  6. PARSE   String JSON --jsonDecode--> Map (satu objek, TANPA 'results')
             ContentItem.fromJson(map, menu)
  7. RENDER  setState(item = data) -> tampilkan item!.title, item!.summary, dst
```

### Langkah A. Parsing list (di `api_service.dart`)

```dart
static Future<List<ContentItem>> getList(String menu) async {
  final response = await http.get(Uri.parse('$baseUrl/$menu/?limit=50'));
  if (response.statusCode == 200) {
    final data = jsonDecode(utf8.decode(response.bodyBytes)); // String -> Map
    final List results = data['results'];                     // ambil list-nya
    return results.map((e) => ContentItem.fromJson(e, menu)).toList(); // tiap Map -> object
  } else {
    throw Exception('Failed to load $menu');
  }
}
```

### Langkah B. Mengubah satu Map menjadi object (di `content_item.dart`)

```dart
factory ContentItem.fromJson(Map<String, dynamic> json, [String? type]) {
  return ContentItem(
    id: json['id'],
    type: type ?? json['type'] ?? '',
    title: json['title'] ?? '',
    author: json['news_site'] ?? 'Unknown',
    imageUrl: json['image_url'] ?? '',
    url: json['url'] ?? '',
    summary: json['summary'] ?? '',
    publishedAt: json['published_at'] ?? '',
  );
}
```

### Langkah C. Kirim data ke detail (di `list_page.dart`)

```dart
onTap: () {
  Navigator.push(
    context,
    MaterialPageRoute(
      builder: (context) => DetailPage(menu: item.type, id: item.id),
    ),
  );
},
```

### Langkah D. Terima di detail dan fetch ulang (di `detail_page.dart`)

```dart
class DetailPage extends StatefulWidget {
  final String menu;
  final int id;
  const DetailPage({super.key, required this.menu, required this.id});
  ...
}

class _DetailPageState extends State<DetailPage> {
  ContentItem? item;        // nullable: data belum ada saat halaman baru dibuka
  bool isLoading = true;

  Future<void> fetchDetail() async {
    try {
      final data = await ApiService.getDetail(widget.menu, widget.id);
      setState(() {
        item = data;
        isLoading = false;
      });
    } catch (e) {
      setState(() => isLoading = false);
      print('Error: $e');
    }
  }

  @override
  void initState() {
    super.initState();
    fetchDetail();            // dipanggil sekali saat halaman dibuka
  }
}
```

Pakai di UI: `item!.title` (tanda `!` aman karena sudah dijaga `isLoading` dan cek `item == null`).

### Perbedaan respons list vs detail (sumber salah paling sering)

| | List | Detail |
|---|---|---|
| URL | `/{menu}/` | `/{menu}/{id}/` |
| Bentuk JSON | `{ "results": [ {...}, {...} ] }` | `{ ... }` (satu objek langsung) |
| Cara ambil | `data['results']` lalu `.map(...)` | `ContentItem.fromJson(data, menu)` |

### Dua cara mengirim data ke detail

| | Cara 1: kirim id, fetch ulang (yang diminta soal) | Cara 2: kirim object langsung |
|---|---|---|
| Kode | `DetailPage(menu: item.type, id: item.id)` | `DetailPage(item: item)` |
| Request baru | Ya, ada loading di detail | Tidak ada |
| Cocok kalau | soal menulis "diakses berdasarkan id" | list sudah berisi semua data detail |

Soal bilang "detail diakses berdasarkan id", jadi pakai **Cara 1**.

---

## 3. Checklist kalau API-nya diganti

Urutan kerja yang aman:

### Langkah 0. Lihat JSON-nya dulu (jangan langsung ngoding)
Buka URL API di browser, atau cetak di console:

```dart
print(response.body.substring(0, 500));
```

Catat empat hal:
1. List-nya ada di key apa? (`results`, `products`, `characters`, `data`, atau langsung `[ ... ]`)
2. Nama field tiap data (judul, gambar, penulis, dsb.)
3. Pola URL detail (`/{id}/` atau `/characters/{id}` atau pakai query)
4. Apakah detail dibungkus key lain? (`{ "data": {...} }`)

### Langkah 1. `api_service.dart`

```dart
static const String baseUrl = 'https://API-BARU';        // ganti alamat dasar
```

```dart
final data = jsonDecode(utf8.decode(response.bodyBytes));
final List results = data['results'];      // ganti nama key list. 'products' / 'characters' / 'data'
// kalau responsnya langsung berupa list:
// final List results = data;
```

```dart
Uri.parse('$baseUrl/$menu/$id/')           // sesuaikan pola URL detail
// contoh pola lain:  '$baseUrl/characters/$id'    atau   '$baseUrl/detail?id=$id'
```

```dart
// kalau detail dibungkus key lain:
return ContentItem.fromJson(data['data'], menu);
```

Parameter tambahan di URL: `?limit=50`, `?page=1`, `?search=kata`, ganti sesuai dokumentasi API.

API yang butuh API key:

```dart
final response = await http.get(
  Uri.parse(url),
  headers: {'Authorization': 'Bearer KEY_KAMU'},   // atau 'x-api-key': 'KEY_KAMU'
);
```

### Langkah 2. `content_item.dart` (model)
Ubah isi `fromJson` supaya nama key-nya cocok dengan JSON baru. Jangan lupa `toJson()`,
karena dipakai untuk menyimpan favorit. Kalau ada field baru, tambahkan di **tiga tempat**:

```dart
final String clan;                                  // 1. deklarasi field + constructor

clan: json['personal']?['clan'] ?? '-',             // 2. fromJson

'personal': {'clan': clan},                         // 3. toJson (agar favorit tetap bisa dibaca ulang)
```

> Kalau di `toJson()` nama key-nya beda dengan yang dibaca `fromJson()`, favorit akan rusak
> setelah aplikasi dibuka ulang. Pastikan keduanya konsisten.

Tidak perlu mengganti nama class (`ContentItem`). Cukup ubah isinya, itu menghemat waktu.

### Langkah 3. `home.dart` (daftar menu)

```dart
static const List<Map<String, dynamic>> _menus = [
  {'title': 'Karakter', 'menu': 'characters', 'icon': Icons.person, 'desc': '...'},
];
```

`menu` = potongan endpoint di URL. Kalau API baru hanya punya satu endpoint, cukup satu kartu.

### Langkah 4. Sesuaikan tampilan

- `list_page.dart`: field yang ditampilkan (`item.title`, `item.author`) dan field untuk chip filter
  (`authors` getter, ganti `e.author` jadi field lain, misalnya `e.clan`).
- `detail_page.dart`: field yang ditampilkan.
- `favorite_service.dart`: biasanya tidak diubah (kuncinya `type_id`).

---

## 4. Tabel penerjemah JSON -> Dart

| Bentuk di JSON | Cara baca di Dart |
|---|---|
| `"title": "abc"` | `json['title'] ?? ''` |
| `"id": 5` | `json['id']` (tipe `int`) |
| `"id": "5"` (String) | ubah tipe `id` jadi `String`, atau `int.parse(json['id'])` |
| `"price": 9` atau `9.5` | `(json['price'] as num).toDouble()` |
| `"images": ["a", "b"]` | `(json['images'] as List).isNotEmpty ? json['images'][0] : ''` |
| `"personal": { "clan": "x" }` | `json['personal']?['clan'] ?? '-'` |
| `"jutsu": ["a", "b", "c"]` | `List<String>.from(json['jutsu'] ?? [])` lalu `.take(3)` untuk 3 pertama |
| `"authors": [ {"name": "X"} ]` | `json['authors'][0]['name']` (cek dulu list tidak kosong) |
| `"published_at": "2026-11-07T02:18:04Z"` | disimpan String, atau `DateTime.parse(...)` |

Aturan baca: `{}` itu **Map** (akses dengan `['key']`), `[]` itu **List** (akses dengan `[index]`).
Ikuti dari luar ke dalam: `data['results'][0]['personal']['clan']`.

Contoh menampilkan beberapa jutsu di detail:

```dart
...item!.jutsu.take(3).map((j) => Text('• $j')),
```

---

## 5. Skenario perubahan yang sering muncul

| Perubahan di soal | Yang diubah |
|---|---|
| API baru, struktur mirip | `baseUrl`, nama key list, `fromJson`, `_menus` |
| Tipe id jadi String | `int id` -> `String id` di model, `DetailPage`, `getDetail`, `ListPage` (navigasi) |
| Tambah field di detail (mis. tanggal lahir) | model (3 tempat), lalu `detail_page.dart` |
| Filter berdasarkan field lain | getter `authors` dan `where(...)` di `list_page.dart` |
| Endpoint tunggal (tanpa 3 menu) | `home.dart` jadi 1 kartu, atau langsung ke `ListPage` |
| Gambar tidak ada / kosong | sudah dijaga `isEmpty` + `errorBuilder`, tidak perlu diubah |
| List berupa grid, bukan list | `ListView.builder` -> `GridView.builder` (Modul 2) |
| Data detail tidak ada endpoint id | pakai Cara 2: kirim object (`DetailPage(item: item)`) |

---

## 6. Error umum dan artinya

| Pesan di console | Penyebab | Solusi |
|---|---|---|
| `type 'Null' is not a subtype of type 'String'` | field null di JSON | tambahkan `?? ''` |
| `type 'String' is not a subtype of type 'int'` | id berupa String | ubah tipe, atau `int.parse(...)` |
| `type '_Map<String, dynamic>' is not a subtype of type 'List<dynamic>'` | salah ambil key; ternyata data berupa objek | cek bentuk JSON lagi |
| `NoSuchMethodError: '[]' was called on null` | key salah atau nested kosong | cek ejaan key, pakai `?.` |
| List kosong tapi tidak ada error | `statusCode` bukan 200, atau error tertelan di `catch` | `print('Error: $e')` di catch |
| Detail 404 | pola URL detail atau id salah | cetak URL: `print('GET: $url')` |
| Karakter aneh pada judul | `response.body` | pakai `utf8.decode(response.bodyBytes)` |
| `setState() called after dispose()` | halaman sudah ditutup saat `await` selesai | `if (!mounted) return;` |
| Package baru tidak terbaca | hot reload tidak cukup | stop lalu run ulang |
| Tombol buka web tidak jalan | `<queries>` belum ada di `AndroidManifest.xml` | tambahkan lalu run ulang |

---

## 7. Urutan kerja saat responsi

1. Baca soal sampai habis, tandai fitur dan poinnya.
2. Buka API di browser, catat key list, nama field, dan pola URL detail.
3. Ubah `content_item.dart` -> `api_service.dart` -> `home.dart`.
4. Tes list dulu (cetak jumlah data), baru tampilan.
5. Tes detail dengan satu item dari tiap menu.
6. Favorit dan filter terakhir.
7. Jalankan semua alur di emulator/HP sebelum dikumpulkan.

Rumus pegangan: **Fetch (http.get) -> Parse (jsonDecode + fromJson) -> Render (setState + widget)**.
Setiap fitur baru hampir selalu hanya mengulang tiga langkah ini.
