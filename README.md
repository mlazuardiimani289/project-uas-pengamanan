# JWT CRUD API UAS Starter

Silakan lanjutkan pengembangan API dan perbaiki bug yang ditemukan.

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


