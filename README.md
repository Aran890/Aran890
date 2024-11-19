// server.js
const express = require('express');
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
const multer = require('multer');
const http = require('http');
const { Server } = require('socket.io');

const app = express();
const server = http.createServer(app);
const io = new Server(server);

const JWT_SECRET = 'supersecretclave'; // Cambiar por una clave más segura

// Configuración de MongoDB
mongoose.connect('mongodb://localhost:27017/appglobal', {
  useNewUrlParser: true,
  useUnifiedTopology: true,
});
const db = mongoose.connection;
db.once('open', () => console.log('Conectado a MongoDB'));

// Middleware
app.use(express.json());

// Configuración para subir fotos/videos
const storage = multer.diskStorage({
  destination: 'uploads/',
  filename: (req, file, cb) => cb(null, Date.now() + '-' + file.originalname),
});
const upload = multer({ storage });

// Esquema de Usuario
const UserSchema = new mongoose.Schema({
  username: String,
  password: String,
  socialLinks: [String],
  profilePicture: String,
});
const User = mongoose.model('User', UserSchema);

// Esquema de Mensaje
const MessageSchema = new mongoose.Schema({
  sender: String,
  content: String,
  timestamp: { type: Date, default: Date.now },
});
const Message = mongoose.model('Message', MessageSchema);

// Registro de usuarios
app.post('/register', async (req, res) => {
  const { username, password } = req.body;
  const hashedPassword = await bcrypt.hash(password, 10);
  const user = new User({ username, password: hashedPassword, socialLinks: [] });
  await user.save();
  res.send({ message: 'Usuario registrado' });
});

// Inicio de sesión
app.post('/login', async (req, res) => {
  const { username, password } = req.body;
  const user = await User.findOne({ username });
  if (!user) return res.status(400).send({ error: 'Usuario no encontrado' });

  const isMatch = await bcrypt.compare(password, user.password);
  if (!isMatch) return res.status(400).send({ error: 'Contraseña incorrecta' });

  const token = jwt.sign({ id: user._id }, JWT_SECRET, { expiresIn: '1h' });
  res.send({ message: 'Inicio de sesión exitoso', token });
});

// Subir foto de perfil
app.post('/upload/profile', upload.single('image'), async (req, res) => {
  const token = req.headers.authorization.split(' ')[1];
  const { id } = jwt.verify(token, JWT_SECRET);

  const user = await User.findById(id);
  if (!user) return res.status(404).send({ error: 'Usuario no encontrado' });

  user.profilePicture = req.file.path;
  await user.save();
  res.send({ message: 'Foto de perfil actualizada', path: req.file.path });
});

// Videollamadas (placeholder para WebRTC)
io.on('connection', (socket) => {
  socket.on('video-call', (data) => {
    io.emit('video-call', data);
  });
});

server.listen(3000, () => console.log('Servidor en http://localhost:3000'));
