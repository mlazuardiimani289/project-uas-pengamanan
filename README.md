# JWT CRUD API UAS Starter

Silakan lanjutkan pengembangan API dan perbaiki bug yang ditemukan.
# UAS Keamanan Komputer

## Identitas
- Nama: Muhammad Lazuardi Imani 
- NIM: 181080200289
- Kelas: Pengamanan Sistem Komputer 6/B4
- Repo GitHub: [link]

---

Bagian A
1.Token tidak valid saat login ulang
Penjelasan: Saat user login ulang, token yang diberikan dianggap tidak valid ketika digunakan untuk mengakses endpoint yang memerlukan otentikasi.

Perbaikannya
*Login:

app.post('/login', (req, res) => {
  const user = { id: 1, username: 'admin' }; // misalnya dari DB
  const token = jwt.sign(user, SECRET_KEY, { expiresIn: '1h' });

  // Optionally generate refresh token
  const refreshToken = jwt.sign(user, SECRET_KEY, { expiresIn: '7d' });

  res.json({ token, refreshToken });
});

*Middleware Autentikasi:

function authenticateToken(req, res, next) {
  const authHeader = req.headers['authorization'];
  if (!authHeader?.startsWith('Bearer ')) return res.sendStatus(401);

  const token = authHeader.split(' ')[1];
  jwt.verify(token, SECRET_KEY, (err, user) => {
    if (err) return res.status(403).json({ error: 'Token invalid or expired' });
    req.user = user;
    next();
  });
}

*Login Ulang (Refresh Token):

app.post('/refresh', (req, res) => {
  const { refreshToken } = req.body;
  if (!refreshToken) return res.sendStatus(401);

  jwt.verify(refreshToken, SECRET_KEY, (err, user) => {
    if (err) return res.sendStatus(403);

    const newToken = jwt.sign({ id: user.id }, SECRET_KEY, { expiresIn: '1h' });
    res.json({ token: newToken });
  });
});

2.User lain bisa mengakses data user lain
Token sah, tapi endpoint tidak mengecek bahwa ID user di route/body sama dengan ID user di token.
Contoh bug:

// Route akses data user by ID
app.get('/user/:id', authenticateToken, (req, res) => {
  const userId = req.params.id;

  // BUG: siapa pun bisa akses user lain selama token valid
  const user = users.find(u => u.id === userId);
  res.json(user);
});

Solusi
*Simpan ID user di token saat login

const token = jwt.sign({ id: user.id, username: user.username }, SECRET_KEY, { expiresIn: '1h' });
*Validasi kepemilikan data di route(Batasi akses hanya ke datanya sendiri)
app.get('/user/:id', authenticateToken, (req, res) => {
  const userId = parseInt(req.params.id);
  
  // Cek apakah user yang sedang login sama dengan user yang diminta
  if (userId !== req.user.id) {
    return res.status(403).json({ error: 'Access denied: not your data' });
  }

  const user = users.find(u => u.id === userId);
  if (!user) return res.status(404).json({ error: 'User not found' });

  res.json(user);
});

*Contoh token payload aman

// Saat login
const token = jwt.sign({
  id: user.id,
  username: user.username,
  role: user.role
}, SECRET_KEY, { expiresIn: '1h' });

3.Logout tidak menghapus token aktif
Saat user logout, token masih bisa digunakan hingga habis masa berlakunya.

Ini terjadi karena:

JWT tidak tersimpan di server ⇒ tidak bisa dihapus.

Logout hanya menghapus token dari client (misalnya localStorage), tapi server tetap menganggap token itu sah.Solusinya:
*Gunakan Token Blacklist di Server (stateful logout)
Saat logout, simpan token di daftar blacklist (misalnya Redis atau in-memory).

Setiap request masuk → cek apakah token termasuk dalam blacklist.

const blacklist = new Set();

app.post('/logout', authenticateToken, (req, res) => {
  const token = req.token; // Ambil dari middleware
  blacklist.add(token); // Simpan token dalam blacklist hingga expired
  res.json({ message: 'Logged out successfully' });
});

// Ubah middleware autentikasi:

function authenticateToken(req, res, next) {
  const authHeader = req.headers['authorization'];
  if (!authHeader?.startsWith('Bearer ')) return res.sendStatus(401);
  const token = authHeader.split(' ')[1];

  if (blacklist.has(token)) return res.status(401).json({ error: 'Token is blacklisted' });

  jwt.verify(token, SECRET_KEY, (err, user) => {
    if (err) return res.sendStatus(403);
    req.user = user;
    req.token = token; // Simpan token untuk logout
    next();
  });
}

*Gunakan Refresh Token + Rotasi
-Akses token pendek (15 menit), refresh token panjang (7 hari).
-Simpan refresh token di database.
-Saat logout, hapus refresh token dari DB.
-Akses token yang sudah expired → tidak bisa diperbarui → user harus login ulang.
Code:

app.post('/logout', (req, res) => {
  const { refreshToken } = req.body;
  // Hapus refreshToken dari database
  db.deleteRefreshToken(refreshToken);
  res.json({ message: 'Logged out successfully' });
});

## Bagian A – Bug Fixing JWT REST API

### Bug 1: Token tetap aktif setelah logout
**Penjelasan:**  
JWT tidak disimpan di server secara default, sehingga tidak bisa dihapus/diblokir dari sisi server setelah logout, kecuali kamu menerapkan strategi manual.

**Solusi:**  
1.Token Blacklist (Pencegahan Token Reuse Setelah Logout)
Simpan token yang sudah dipakai logout ke dalam blacklist (misal Redis/DB/In-memory), lalu saat ada request masuk, periksa apakah token itu termasuk blacklist.
2.Gunakan Refresh Token + Rotasi + Penyimpanan Server
Akses token → umur pendek (misal 15 menit)
Refresh token → umur panjang (7 hari), disimpan di DB
Saat logout:
-Akses token dibiarkan expired sendiri
-Refresh token dihapus dari DB
-Tidak bisa generate akses token baru
## Bagian B – Simulasi Serangan dan Solusi

### Jenis Serangan: Broken Access Control  
**Simulasi Postman:**  
1.Pilih method: POST
URL: http://yourapi.com/api/users/update
Header:
Authorization: Bearer <access_token_user_biasa>
Content-Type: application/json
Body → raw → JSON:
{
  "user_id": "999",
  "email": "admin@example.com",
  "role": "user"
}

**Solusi Implementasi:**  
Tujuan Middleware
Mencegah user biasa mengakses/mengubah resource milik user lain

Membatasi akses berdasarkan:

ID yang ada di JWT vs ID target di request

Role (admin vs user)

Contoh Penggunaan Middleware di Route
const express = require('express');
const app = express();
const authenticateToken = require('./middleware/authenticateToken');
const authorizeSelfOrAdmin = require('./middleware/authorizeSelfOrAdmin');

app.use(express.json());

// Route untuk update user
app.put('/api/users/:id', authenticateToken, authorizeSelfOrAdmin(req => req.params.id), (req, res) => {
  // ✅ Di sini hanya admin atau pemilik akun yang bisa sampai
  const userId = req.params.id;

  // Lanjutkan proses update
  res.json({ message: `User ${userId} updated successfully.` });
});




## Bagian C – Refleksi Teori & Etika

### 1. CIA Triad dalam Keamanan Informasi  
- Confidentiality (Keamanan), Menjamin bahwa informasi hanya dapat diakses oleh pihak yang berwenang. Contoh: enkripsi data.
- Integrity (Integritas), Memastikan bahwa informasi tidak diubah oleh pihak yang tidak berwenang, baik sengaja maupun tidak sengaja. Contoh: hash, checksums.
- Avaibility (Ketersediaan), Informasi tersedia dan dapat diakses oleh pengguna yang berhak saat dibutuhkan. Contoh: backup.

### 2. UU ITE yang relevan  
 1. Pasal 30 – Akses Tanpa Hak ke Sistem Elektronik:Setiap orang dilarang mengakses sistem elektronik milik orang lain tanpa hak, melawan hukum, atau dengan cara apapun.
 2. Pasal 34 – Pemalsuan Informasi atau Dokumen Elektronik:Dilarang memalsukan dokumen/informasi elektronik seolah-olah sah, padahal tidak.

### 3. Pandangan Al-Qur'an  
- Surah Al-Baqarah: 205  
1. 🔧 Larangan Keras atas Perusakan Sistem Kehidupan
Al-Qur’an dalam ayat ini dengan tegas mengecam tindakan yang menyebabkan kerusakan sistem – baik sosial, ekonomi, lingkungan, maupun moral.

"Yufsidu fi al-ardh" (mengadakan kerusakan di bumi) menunjukkan segala bentuk pelanggaran terhadap keteraturan hidup, termasuk korupsi, fitnah, kerusakan alam, dan konflik sosial.

Perusakan bukan hanya fisik (lingkungan atau harta), tapi juga merusak tatanan nilai, keadilan, dan kemanusiaan.

2. 🌾 Simbol Tanaman dan Ternak: Fondasi Sistem Ekonomi
Allah menyebut “merusak tanaman dan binatang ternak” sebagai perwujudan nyata dari fasād (kerusakan):

Tanaman = lambang pertanian dan sumber pangan

Ternak = lambang produktivitas dan ekonomi

➤ Ini menunjukkan larangan atas perusakan sistem ekonomi dan lingkungan, karena kedua hal itu adalah fondasi keberlangsungan hidup manusia.

3. ❌ Allah Tidak Menyukai Kerusakan
Ayat ini ditutup dengan:

“Wallāhu lā yuḥibbul-fasād” – “Dan Allah tidak menyukai kerusakan.”

Artinya:

Segala tindakan yang mengacaukan sistem yang telah Allah tetapkan di bumi bukan hanya dilarang, tetapi dibenci oleh Allah.

Islam mendorong manusia untuk menjadi pembangun (islah), bukan perusak (fasād).

4. ⚠️ Kerusakan Sistem = Akibat dari Munafik dan Orang Zalim
Konteks ayat ini berbicara tentang orang munafik yang menipu dengan kata-kata manis, tapi diam-diam menghancurkan sistem.

Perusakan sistem bisa disebabkan oleh keserakahan, kekuasaan yang disalahgunakan, dan niat jahat yang tersembunyi.

Ini adalah peringatan keras terhadap manipulasi sistem politik, hukum, dan ekonomi demi keuntungan pribadi.

### 4. Etika Cyber dan Kejujuran  
1. Nilai Kejujuran dalam Cybersecurity
💡 Apa itu kejujuran di sini?
Kejujuran berarti bertindak sesuai dengan fakta dan integritas profesional, tidak menyalahgunakan akses atau informasi.
Contoh Penerapan:
Situasi	Tindakan Jujur
🔍 Audit sistem keamanan	Melaporkan semua kerentanan dengan jujur, tanpa ditutup-tutupi
👨‍💻 Akses sistem	Tidak mengakses data yang tidak relevan atau tidak diberi izin
💬 Komunikasi ke klien	Tidak memberikan jaminan palsu atau laporan palsu atas keamanan sistem
3. Nilai Amanah dalam Cybersecurity
💡 Apa itu amanah di sini?
Amanah berarti menjaga kepercayaan dan tidak menyalahgunakan hak akses terhadap data, sistem, atau informasi sensitif.
Contoh Penerapan:
Situasi	Bentuk Amanah
🧑‍💼 Admin sistem	Menjaga kerahasiaan data pengguna dan tidak menyalahgunakannya
🛠 Ethical hacker	Menggunakan akses hanya untuk pengujian yang disepakati
💾 Pengelolaan data	Menjaga agar data tidak bocor, dijual, atau diakses pihak luar tanpa izin



