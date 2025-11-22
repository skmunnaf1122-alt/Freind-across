import React, {useEffect, useState} from 'react';
import { View, Text, TextInput, Button, FlatList } from 'react-native';
import { io } from 'socket.io-client';

export default function ChatScreen({route}){
  const { token, conversationId } = route.params;
  const [socket,setSocket] = useState(null);
  const [text,setText] = useState('');
  const [messages,setMessages] = useState([]);
  useEffect(()=>{
    // --- EDIT HERE --- replace with your server URL
    const s = io('http://REPLACE_WITH_BACKEND:4000', { auth: { token } });
    setSocket(s);
    s.on('connect', ()=> console.log('socket connected'));
    s.on('message', m=> setMessages(prev=>[...prev,m]));
    s.emit('joinConversation', conversationId);
    return ()=> s.disconnect();
  },[]);
  const send = ()=>{
    if(!socket) return;
    socket.emit('sendMessage', { conversationId, message: { text } });
    setText('');
  };
  return (
    <View style={{flex:1,padding:10}}>
      <FlatList data={messages} keyExtractor={(item,idx)=>String(idx)} renderItem={({item})=> <Text>{item.message.text}</Text>} />
      <TextInput value={text} onChangeText={setText} style={{borderWidth:1,padding:8}} />
      <Button title="Send" onPress={send} />
    </View>
  );
}
import React, {useEffect, useState} from 'react';
import { View, Text, Button, FlatList } from 'react-native';
import api from '../utils/api';

export default function FeedScreen({route, navigation}){
  const { token } = route.params || {};
  const [me,setMe] = useState(null);
  const [posts,setPosts] = useState([]);
  useEffect(()=>{
    if(!token) return;
    api.get('/users/me', { headers: { Authorization: `Bearer ${token}` } }).then(r=> setMe(r.data)).catch(()=>{});
    api.get('/posts?limit=10').then(r=> setPosts(r.data)).catch(()=>{});
  },[]);
  return (
    <View style={{flex:1,padding:20}}>
      <Text>Welcome, {me?.display_name||'friend'}</Text>
      <Button title="Open Chat (demo)" onPress={()=> navigation.navigate('Chat', { token, conversationId: 'demo-conv' })} />
      <FlatList data={posts} keyExtractor={i=>i.id} renderItem={({item})=> <Text style={{padding:8,borderBottomWidth:1}}>{item.display_name}: {item.text}</Text>} />
    </View>
  );
}
import React, {useState} from 'react';
import { View, TextInput, Button, StyleSheet, Text } from 'react-native';
import api from '../utils/api';

export default function LoginScreen({navigation}){
  const [email,setEmail] = useState('');
  const [password,setPassword] = useState('');
  const [err,setErr] = useState('');
  const onLogin = async ()=>{
    try{
      const r = await api.post('/auth/login', { email, password });
      const token = r.data.accessToken;
      // For now we pass token via navigation params
      navigation.replace('Feed', { token });
    }catch(e){
      console.error(e);
      setErr('Login failed - check console for details');
    }
  };
  return (
    <View style={styles.container}>
      <Text style={styles.title}>FriendAcross</Text>
      <TextInput placeholder="email" style={styles.input} value={email} onChangeText={setEmail} />
      <TextInput placeholder="password" style={styles.input} secureTextEntry value={password} onChangeText={setPassword} />
      {err ? <Text style={{color:'red'}}>{err}</Text> : null}
      <Button title="Login" onPress={onLogin} />
      <Text style={{marginTop:10}}>Tip: use the register endpoint to create an account first.</Text>
    </View>
  );
}

const styles = StyleSheet.create({
  container:{flex:1,justifyContent:'center',padding:20},
  input:{borderWidth:1,padding:10,marginBottom:10},
  title:{fontSize:24,marginBottom:20,textAlign:'center'}
});
/**
 * API helper.
 * --- EDIT HERE --- set API_BASE to your backend URL before running on device.
 * For Android emulator with local machine backend: use http://10.0.2.2:4000
 */
import axios from 'axios';
export const API_BASE = 'http://REPLACE_WITH_BACKEND:4000/api/v1';
const instance = axios.create({ baseURL: API_BASE, timeout: 10000 });
export default instance;
/**
 * Expo entry for FriendAcross mobile prototype.
 * Edit mobile/utils/api.js -> API_BASE to point to your backend (e.g., http://10.0.2.2:4000)
 */
import React from 'react';
import { NavigationContainer } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import LoginScreen from './screens/LoginScreen';
import FeedScreen from './screens/FeedScreen';
import ChatScreen from './screens/ChatScreen';

const Stack = createNativeStackNavigator();
export default function App(){
  return (
    <NavigationContainer>
      <Stack.Navigator initialRouteName="Login">
        <Stack.Screen name="Login" component={LoginScreen} />
        <Stack.Screen name="Feed" component={FeedScreen} />
        <Stack.Screen name="Chat" component={ChatScreen} />
      </Stack.Navigator>
    </NavigationContainer>
  );
}
{
  "name": "friendacross-mobile",
  "version": "0.1.0",
  "main": "node_modules/expo/AppEntry.js",
  "scripts": {
    "start": "expo start",
    "android": "expo run:android"
  },
  "dependencies": {
    "expo": "^48.0.0",
    "react": "18.2.0",
    "react-native": "0.71.8",
    "axios": "^1.3.0",
    "socket.io-client": "^4.7.0",
    "@react-navigation/native": "^6.0.13",
    "@react-navigation/native-stack": "^6.9.0"
  }
}/**
 * Simple JWT auth middleware.
 * It expects header: Authorization: Bearer <token>
 * The token is created in auth.js routes.
 */
const jwt = require('jsonwebtoken');
require('dotenv').config();

module.exports = function(req,res,next){
  const h = req.headers.authorization;
  if(!h) return res.status(401).json({ error: 'no auth header' });
  const token = h.split(' ')[1];
  try{
    req.user = jwt.verify(token, process.env.JWT_SECRET);
    return next();
  }catch(e){
    return res.status(401).json({ error: 'invalid token' });
  }
};
/**
 * Auth routes (register/login).
 * Edit the fields you require. Currently uses email + password.
 * After successful register/login returns { accessToken }.
 * Replace `REPLACE_ME` comments to customize.
 */

const express = require('express');
const router = express.Router();
const db = require('../db');
const bcrypt = require('bcrypt');
const jwt = require('jsonwebtoken');
require('dotenv').config();

// POST /api/v1/auth/register
// Body: { email, password, display_name }
router.post('/register', async (req,res)=>{
  const { email, password, display_name } = req.body;
  if(!email || !password) return res.status(400).json({ error: 'missing fields' });
  const hashed = await bcrypt.hash(password, 10);
  try{
    const r = await db.query(
      'INSERT INTO users (email, hashed_password, display_name, created_at) VALUES ($1,$2,$3,NOW()) RETURNING id, email, display_name',
      [email, hashed, display_name||email]
    );
    const user = r.rows[0];
    const token = jwt.sign({ id: user.id, email: user.email, display_name: user.display_name }, process.env.JWT_SECRET, { expiresIn: '7d' });
    return res.json({ accessToken: token });
  }catch(e){
    console.error('register error', e);
    return res.status(500).json({ error: 'server error' });
  }
});

// POST /api/v1/auth/login
// Body: { email, password }
router.post('/login', async (req,res)=>{
  const { email, password } = req.body;
  if(!email || !password) return res.status(400).json({ error: 'missing fields' });
  try{
    const r = await db.query('SELECT id, email, hashed_password, display_name FROM users WHERE email=$1', [email]);
    if(r.rowCount===0) return res.status(400).json({ error: 'invalid credentials' });
    const user = r.rows[0];
    const ok = await bcrypt.compare(password, user.hashed_password);
    if(!ok) return res.status(400).json({ error: 'invalid credentials' });
    const token = jwt.sign({ id: user.id, email: user.email, display_name: user.display_name }, process.env.JWT_SECRET, { expiresIn: '7d' });
    return res.json({ accessToken: token });
  }catch(e){
    console.error('login error', e);
    return res.status(500).json({ error: 'server error' });
  }
});

module.exports = router;
/**
 * Simple posts (moments) endpoints.
 * For simplicity this stores only text posts. You can extend with media (S3).
 */
const express = require('express');
const router = express.Router();
const db = require('../db');
const auth = require('../middleware/auth');

// POST /api/v1/posts
// Body: { text }
router.post('/', auth, async (req,res)=>{
  const { text } = req.body;
  try{
    const r = await db.query('INSERT INTO posts (user_id, text, created_at) VALUES ($1,$2,NOW()) RETURNING id', [req.user.id, text]);
    return res.json({ id: r.rows[0].id });
  }catch(e){
    console.error('create post error', e);
    return res.status(500).json({ error: 'server error' });
  }
});

// GET /api/v1/posts?limit=20
router.get('/', async (req,res)=>{
  const limit = parseInt(req.query.limit) || 20;
  try{
    const r = await db.query('SELECT p.id, p.text, p.created_at, u.display_name FROM posts p JOIN users u ON u.id=p.user_id ORDER BY p.created_at DESC LIMIT $1', [limit]);
    return res.json(r.rows);
  }catch(e){
    console.error('list posts error', e);
    return res.status(500).json({ error: 'server error' });
  }
});

module.exports = router;
/**
 * Users routes: fetch profile and update profile.
 * Protected endpoints using auth middleware.
 */
const express = require('express');
const router = express.Router();
const db = require('../db');
const auth = require('../middleware/auth');

// GET /api/v1/users/me
router.get('/me', auth, async (req,res)=>{
  try{
    const r = await db.query('SELECT id, email, display_name, bio, country, avatar_url FROM users WHERE id=$1', [req.user.id]);
    return res.json(r.rows[0]);
  }catch(e){
    console.error('get me error', e);
    return res.status(500).json({ error: 'server error' });
  }
});

// PUT /api/v1/users/me
// Body: { display_name, bio, country, avatar_url }
router.put('/me', auth, async (req,res)=>{
  const { display_name, bio, country, avatar_url } = req.body;
  try{
    await db.query('UPDATE users SET display_name=$1, bio=$2, country=$3, avatar_url=$4, updated_at=NOW() WHERE id=$5',
      [display_name, bio, country, avatar_url, req.user.id]);
    return res.json({ success:true });
  }catch(e){
    console.error('update me error', e);
    return res.status(500).json({ error: 'server error' });
  }
});

module.exports = router;
/**
 * Database client
 * Edit DATABASE_URL in server/.env.example or set environment variable.
 */
const { Client } = require('pg');
require('dotenv').config();
const client = new Client({ connectionString: process.env.DATABASE_URL });
client.connect().then(()=> console.log('Postgres connected')).catch(err=> console.error('PG connect error', err));
module.exports = client;
/**
 * Entry point for the backend.
 * Edit .env and set values before running.
 *
 * Important placeholders you'll edit:
 *  - server/.env: JWT_SECRET, DATABASE_URL
 *
 * Endpoints:
 *  - /api/v1/auth/*   -> register/login
 *  - /api/v1/users/*  -> profile (protected)
 *  - /api/v1/posts/*  -> moments/posts
 */
require('dotenv').config();
const express = require('express');
const http = require('http');
const cors = require('cors');
const bodyParser = require('body-parser');

const authRoutes = require('./routes/auth');
const userRoutes = require('./routes/users');
const postRoutes = require('./routes/posts');

const db = require('./db');
const socketSetup = require('./socket');

const app = express();
app.use(cors());
app.use(bodyParser.json());

// mount routes
app.use('/api/v1/auth', authRoutes);
app.use('/api/v1/users', userRoutes);
app.use('/api/v1/posts', postRoutes);

// basic root
app.get('/', (req,res)=> res.send('FriendAcross API'));

// start server + socket.io
const server = http.createServer(app);
const { Server } = require('socket.io');
const io = new Server(server, { cors: { origin: '*' } });
socketSetup(io);

const PORT = process.env.PORT || 4000;
server.listen(PORT, ()=> console.log('Server listening on', PORT));
