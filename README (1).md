# Contekan Responsi Mobile (Flutter)

Aplikasi ini: **Spaceflight News**
Alurnya: Login/Register → Home (3 menu) → List (ada filter) → Detail (favorit + buka web) → Favorite.

Cara pakai contekan ini: cari soal yang berubah → salin kodenya → ganti bagian yang bertanda `// GANTI`.

---

## 0. Siapa ngurus apa

| File | Gunanya |
|---|---|
| `main.dart` | Cek sudah login atau belum |
| `models/content_item.dart` | Bentuk data (judul, gambar, dll) |
| `services/api_service.dart` | Ambil data dari API |
| `services/auth_service.dart` | Simpan akun, login, logout |
| `services/favorite_service.dart` | Simpan favorit |
| `pages/login.dart`, `register.dart` | Form login dan daftar |
| `pages/home.dart` | Daftar menu |
| `pages/list_page.dart` | Daftar data + filter |
| `pages/detail_page.dart` | Isi detail |
| `pages/favorite_page.dart` | Daftar favorit |
| `pubspec.yaml` | Daftar package |

---

## 1. API / studi kasus baru

Ubah berurutan: **api_service → model → home → list → detail**.

**① `api_service.dart`**
```dart
static const String baseUrl = 'https://api.contoh.com/v1'; // GANTI: alamat API baru

// ambil list
final response = await http.get(Uri.parse('$baseUrl/$menu/?limit=50')); // GANTI: path sesuai API
final data = jsonDecode(utf8.decode(response.bodyBytes));
final List results = data['results']; // GANTI: 'results' sesuai JSON (bisa 'data', 'meals', dll)

// kalau JSON-nya langsung list [ {...}, {...} ], pakai:
// final List results = data;

// kalau API butuh key:
// http.get(Uri.parse(url), headers: {'Authorization': 'Bearer KEY_KAMU'});
```

**② `content_item.dart`** → samakan nama kunci dengan JSON dari API
```dart
factory ContentItem.fromJson(Map<String, dynamic> json, [String? type]) {
  return ContentItem(
    id: json['id'],                  // GANTI kunci sesuai JSON
    type: type ?? json['type'] ?? '',
    title: json['title'] ?? '',      // GANTI
    author: json['news_site'] ?? 'Unknown', // GANTI
    imageUrl: json['image_url'] ?? '',      // GANTI
    url: json['url'] ?? '',
    summary: json['summary'] ?? '',
    publishedAt: json['published_at'] ?? '',
  );
}
// toJson() harus sama persis kuncinya dengan fromJson (dipakai untuk favorit)
```
Kalau `id` bukan angka tapi teks: ubah `final int id` jadi `final String id`, lalu `getDetail(String menu, String id)` dan `DetailPage` juga.

**③ `home.dart`** → ubah isi `_menus`
```dart
static const List<Map<String, dynamic>> _menus = [
  {
    'title': 'Resep',          // tulisan di kartu
    'menu': 'recipes',         // path endpoint API
    'icon': Icons.restaurant,  // ikon
    'desc': 'Kumpulan resep',  // deskripsi
  },
  // tambah menu = tambah satu {...} lagi
];
```

**④ `list_page.dart` / `detail_page.dart`** → tinggal ganti field yang ditampilkan
```dart
Text(item.title)        // judul
Text(item.author)       // penulis / sumber
Text(item.summary)      // ringkasan
```

### Nambah kolom baru di model (field baru dari API)
Tambah di **4 tempat** di `content_item.dart`:
```dart
final String category;                 // 1. atribut
required this.category,                // 2. constructor
category: json['category'] ?? '',      // 3. di fromJson
'category': category,                  // 4. di toJson
```
Lalu tampilkan: `Text(item.category)`.

---

## 2. Nambah kolom input (TextEditingController)

Contoh: tambah kolom **Email** di Register. Selalu 4 langkah ini:

**`register.dart`**
```dart
// 1. Buat controller (di dalam class _RegisterPageState)
final TextEditingController _emailController = TextEditingController();

// 2. Pasang TextField di dalam Column (children)
Padding(
  padding: const EdgeInsets.symmetric(horizontal: 20, vertical: 10),
  child: TextField(
    controller: _emailController,
    keyboardType: TextInputType.emailAddress, // keyboard khusus email
    decoration: InputDecoration(
      labelText: 'Email',
      border: OutlineInputBorder(borderRadius: BorderRadius.circular(8)),
    ),
  ),
),

// 3. Tutup controller di dispose()
@override
void dispose() {
  _usernameController.dispose();
  _passwordController.dispose();
  _emailController.dispose(); // tambah ini
  super.dispose();
}

// 4. Ambil nilainya di _register()
Future<void> _register() async {
  final username = _usernameController.text.trim();
  final password = _passwordController.text.trim();
  final email = _emailController.text.trim(); // tambah ini

  if (username.isEmpty || password.isEmpty || email.isEmpty) {
    _showError('Semua kolom harus diisi');
    return;
  }
  if (!email.contains('@')) {            // contoh validasi email
    _showError('Email tidak valid');
    return;
  }
  if (password.length < 6) {             // contoh validasi password
    _showError('Password minimal 6 karakter');
    return;
  }

  final success = await AuthService.register(username, password, email); // tambah email
  ...
}
```

**`auth_service.dart`** → simpan kolom baru (ganti 3 function ini)
```dart
static Future<Map<String, dynamic>> _getUsers() async {
  final prefs = await SharedPreferences.getInstance();
  final raw = prefs.getString(_usersKey);
  if (raw == null) return {};
  return Map<String, dynamic>.from(jsonDecode(raw));
}

static Future<bool> register(String username, String password, String email) async {
  final users = await _getUsers();
  if (users.containsKey(username)) return false;
  users[username] = {'password': password, 'email': email}; // simpan sebagai objek
  final prefs = await SharedPreferences.getInstance();
  await prefs.setString(_usersKey, jsonEncode(users));
  return true;
}

static Future<bool> login(String username, String password) async {
  final users = await _getUsers();
  final user = users[username];
  if (user != null && user['password'] == password) { // cek password dari objek
    final prefs = await SharedPreferences.getInstance();
    await prefs.setString(_loggedInKey, username);
    return true;
  }
  return false;
}
```

**Kalau cuma tambah validasi** (tanpa kolom baru): cukup tambah `if (...) { _showError('...'); return; }` di `_register()` atau `_login()`. `auth_service` tidak perlu diubah.

**Kalau login pakai email, bukan username:** ganti nama controller jadi `_emailController` dan teruskan nilainya ke `AuthService.login(email, password)`.

---

## 3. Navigasi

### 3a. Pindah halaman biasa (`Navigator`)
```dart
// MAJU ke halaman lain (bisa tekan back)
Navigator.push(
  context,
  MaterialPageRoute(builder: (context) => const FavoritePage()),
);

// MAJU sambil kirim data (data lewat constructor)
Navigator.push(
  context,
  MaterialPageRoute(
    builder: (context) => DetailPage(menu: item.type, id: item.id),
  ),
);

// KEMBALI
Navigator.pop(context);

// GANTI halaman (tidak bisa back). Dipakai setelah login berhasil
Navigator.pushReplacement(
  context,
  MaterialPageRoute(builder: (context) => HomePage(username: username)),
);

// HAPUS SEMUA halaman sebelumnya. Dipakai saat logout
Navigator.pushAndRemoveUntil(
  context,
  MaterialPageRoute(builder: (context) => const LoginPage()),
  (route) => false,
);
```
Halaman tujuan menerima data seperti ini:
```dart
class DetailPage extends StatefulWidget {
  final String menu;
  final int id;
  const DetailPage({super.key, required this.menu, required this.id});
  ...
}
// di dalam State dipakai sebagai: widget.menu dan widget.id
```

### 3b. Bottom Navigation (menu di bawah)
Buat file `lib/pages/main_navigation.dart` (sudah ada filenya, ini isinya):
```dart
import 'package:flutter/material.dart';
import 'home.dart';
import 'favorite_page.dart';

class MainNavigation extends StatefulWidget {
  final String username;
  const MainNavigation({super.key, required this.username});

  @override
  State<MainNavigation> createState() => _MainNavigationState();
}

class _MainNavigationState extends State<MainNavigation> {
  int _currentIndex = 0; // tab yang sedang aktif

  // urutan halaman HARUS sama dengan urutan item di bawah
  late final List<Widget> _pages = [
    HomePage(username: widget.username),
    const FavoritePage(),
  ];

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: IndexedStack(index: _currentIndex, children: _pages),
      bottomNavigationBar: BottomNavigationBar(
        currentIndex: _currentIndex,
        onTap: (index) => setState(() => _currentIndex = index),
        items: const [
          BottomNavigationBarItem(icon: Icon(Icons.home), label: 'Home'),
          BottomNavigationBarItem(icon: Icon(Icons.favorite), label: 'Favorite'),
        ],
      ),
    );
  }
}
```
Supaya dipakai, ganti `HomePage(username: ...)` jadi `MainNavigation(username: ...)` di **`main.dart`** dan **`login.dart`** (jangan lupa `import`).
Mau tambah tab: tambah 1 halaman di `_pages` dan 1 `BottomNavigationBarItem`.

### 3c. Drawer (menu geser dari kiri)
Taruh di dalam `Scaffold(...)`:
```dart
drawer: Drawer(
  child: ListView(
    children: [
      DrawerHeader(child: Text('Hai, $username')),
      ListTile(
        leading: const Icon(Icons.favorite),
        title: const Text('Favorite'),
        onTap: () {
          Navigator.pop(context); // tutup drawer dulu
          Navigator.push(
            context,
            MaterialPageRoute(builder: (context) => const FavoritePage()),
          );
        },
      ),
    ],
  ),
),
```

### 3d. TabBar (tab di atas)
```dart
DefaultTabController(
  length: 3, // jumlah tab
  child: Scaffold(
    appBar: AppBar(
      title: const Text('Judul'),
      bottom: const TabBar(
        tabs: [Tab(text: 'News'), Tab(text: 'Blog'), Tab(text: 'Reports')],
      ),
    ),
    body: const TabBarView(
      children: [
        Center(child: Text('Halaman 1')), // GANTI dengan halaman asli
        Center(child: Text('Halaman 2')),
        Center(child: Text('Halaman 3')),
      ],
    ),
  ),
)
```

---

## 4. Filter dan hal kecil

**Ganti filter chip** (misal dari author jadi kategori) di `list_page.dart`:
```dart
List<String> get authors => items.map((e) => e.category).toSet().toList();

List<ContentItem> get filteredItems => selectedAuthor == null
    ? items
    : items.where((e) => e.category == selectedAuthor).toList();
```

| Permintaan | Ubah di |
|---|---|
| Warna tema | `Color.fromARGB(255, 14, 127, 219)` (ada di semua page, cari & ganti semua) |
| Jumlah data | `?limit=50` di `api_service.dart` |
| Teks pesan error | `SnackBar(content: Text('...'))` |
| Favorit jadi "bookmark" | Cukup ganti teks dan ikon |
| Halaman setelah login | `HomePage(...)` di `login.dart` dan `main.dart` |

---

## 5. Tambah package (dependencies)

**Kapan?** Kalau kamu menulis `import 'package:...'` tapi package-nya belum ada di `pubspec.yaml` (errornya: *Target of URI doesn't exist*).

Sudah ada di project ini: `http` (ambil API), `shared_preferences` (simpan data lokal), `url_launcher` (buka link).

| Butuh | Package |
|---|---|
| Format tanggal / angka | `intl` |
| State management | `provider` |
| Cache gambar | `cached_network_image` |
| Database lokal | `sqflite` + `path` |

**Lewat VS Code:** `Ctrl + Shift + P` → ketik **Dart: Add Dependency** → Enter → ketik nama package → Enter.

**Lewat terminal / manual** (semua jadi satu kotak):
```bash
# Tambah package (otomatis edit pubspec.yaml dan pub get)
flutter pub add intl

# Atau edit pubspec.yaml manual, tambah di bawah dependencies: (spasi harus lurus dengan http:)
#   dependencies:
#     http: ^1.6.0
#     intl: ^0.20.2     <- baris baru
# lalu jalankan:
flutter pub get

# Kalau masih error
flutter clean
flutter pub get
```
Setelah tambah package: **stop lalu run ulang** aplikasi (hot reload tidak cukup).

---

## 6. Commit ke GitHub (dari terminal VS Code)

Buka terminal VS Code (`Ctrl + ~`) di folder project, lalu jalankan berurutan:

```bash
# 1. Aktifkan git di folder (sekali saja)
git init

# 2. Hubungkan ke repo GitHub (ganti URL dengan punyamu, sekali saja)
git remote add origin https://github.com/USERNAME/NAMA-REPO.git

# 3. Pilih semua file untuk di-commit
git add .

# 4. Simpan dengan pesan
git commit -m "responsi selesai"

# 5. Samakan nama branch jadi main
git branch -M main

# 6. Kirim ke GitHub (pertama kali pakai -u)
git push -u origin main

# ---- Commit berikutnya cukup 3 baris ini ----
# git add .
# git commit -m "pesan"
# git push
```

Tips:
- Cek sudah terhubung atau belum: `git remote -v`. Cek status file: `git status`
- Salah URL repo: `git remote set-url origin URL_YANG_BENAR`
- `git commit` minta identitas, jalankan sekali:
  `git config --global user.name "Nama"` dan `git config --global user.email "email@kamu.com"`
- Push ditolak karena repo GitHub sudah ada isinya (misal README): `git pull origin main --allow-unrelated-histories` lalu `git push -u origin main`
- Jangan bikin README pakai `echo "# ..." >> README.md` di PowerShell (hasilnya huruf berspasi seperti README lama, karena encoding UTF-16). Edit README langsung di VS Code.

---

## 7. Kalau error

| Masalah | Coba ini |
|---|---|
| Data kosong / gagal muat | Cek `data['results']` dan kunci JSON di `fromJson` |
| `type 'Null' is not a subtype...` | Ada kunci JSON salah, kasih default: `json['x'] ?? ''` |
| Package tidak ketemu | `flutter pub get` |
| Tidak ada internet di HP (release) | Tambah di `android/app/src/main/AndroidManifest.xml`, sebelum `<application>`: `<uses-permission android:name="android.permission.INTERNET"/>` |
| Ubah kode tapi tampilan tetap lama | Hot **Restart** (bukan hot reload) |
