# Social Media Dashboard — Project README

## 1. Project Overview
A full‑stack social media dashboard application built using the MERN stack (MongoDB, Express, React, Node) and Socket.IO. The platform allows users to create profiles, share media, engage with others in real‑time, and track analytics on engagement.

**Core Features:**
- User profiles with media uploads
- Real‑time messaging with WebSockets (Socket.IO)
- Like, comment, and follow functionalities
- Analytics dashboard for user engagement

**Advanced Add‑On:**
- Notification system with Redis pub/sub for scalable real‑time notifications.

---

## 2. Tech Stack
- **Database:** MongoDB (Mongoose ODM)
- **Backend:** Node.js + Express.js
- **Frontend:** React.js (CRA or Vite)
- **Real‑Time:** Socket.IO (WebSockets)
- **Caching / Notifications:** Redis
- **Auth:** JWT (JSON Web Tokens)
- **Media Uploads:** Cloud storage (AWS S3, Cloudinary)

---

## 3. High‑Level Architecture

```
[React Frontend] <----> [Express API + Socket.IO] <----> [MongoDB]
                                       |
                                       +--> [Redis] (notifications)
                                       +--> [S3/Cloudinary] (media)
```

- React frontend interacts with Express REST API + Socket.IO for real‑time.
- MongoDB stores users, posts, comments, messages.
- Redis pub/sub used to push notifications across servers (useful for scaling Socket.IO horizontally).
- Media files stored in cloud.

---

## 4. Suggested Folder Structure
```
/social-media-dashboard
  /backend
    /src
      /controllers
      /models
      /routes
      /middlewares
      /services
      /utils
      server.js
    package.json
  /frontend
    /src
      /components
      /pages
      /contexts
      /hooks
      /services
      App.jsx
    package.json
  .env.example
  README.md
```

---

## 5. Environment Variables
**Backend `.env`**
```
PORT=5000
MONGO_URI=mongodb+srv://<user>:<pass>@cluster.mongodb.net/socialdb
JWT_SECRET=super_secret_key
CLOUDINARY_URL=cloudinary://<api_key>:<api_secret>@<cloud_name>
REDIS_URL=redis://localhost:6379
```

**Frontend `.env`**
```
VITE_API_BASE_URL=http://localhost:5000/api
VITE_SOCKET_URL=http://localhost:5000
```

---

## 6. Database Models

### User (models/User.js)
```js
const mongoose = require('mongoose')
const userSchema = new mongoose.Schema({
  username: { type: String, unique: true, required: true },
  email: { type: String, unique: true, required: true },
  password: { type: String, required: true },
  avatar: String,
  bio: String,
  followers: [{ type: mongoose.Schema.Types.ObjectId, ref: 'User' }],
  following: [{ type: mongoose.Schema.Types.ObjectId, ref: 'User' }],
  createdAt: { type: Date, default: Date.now }
})
module.exports = mongoose.model('User', userSchema)
```

### Post (models/Post.js)
```js
const postSchema = new mongoose.Schema({
  author: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  caption: String,
  mediaUrl: String,
  likes: [{ type: mongoose.Schema.Types.ObjectId, ref: 'User' }],
  comments: [{
    user: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
    text: String,
    createdAt: { type: Date, default: Date.now }
  }],
  createdAt: { type: Date, default: Date.now }
})
module.exports = mongoose.model('Post', postSchema)
```

### Message (models/Message.js)
```js
const messageSchema = new mongoose.Schema({
  sender: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  receiver: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  text: String,
  createdAt: { type: Date, default: Date.now }
})
module.exports = mongoose.model('Message', messageSchema)
```

---

## 7. Backend API — Endpoints
Base: `/api`

### Auth
- `POST /api/auth/register`
- `POST /api/auth/login`
- `GET /api/auth/me`

### User
- `GET /api/users/:id`
- `PUT /api/users/:id` (update profile)
- `POST /api/users/:id/follow`
- `POST /api/users/:id/unfollow`

### Posts
- `POST /api/posts` (upload + caption)
- `GET /api/posts` (feed)
- `GET /api/posts/:id`
- `POST /api/posts/:id/like`
- `POST /api/posts/:id/comment`
- `DELETE /api/posts/:id`

### Messages (REST fallback, most handled via Socket.IO)
- `GET /api/messages/:userId` — get chat history

### Analytics
- `GET /api/analytics/overview` (likes, comments, posts, followers count)
- `GET /api/analytics/user/:id` (user engagement stats)

---

## 8. Real‑Time Messaging (Socket.IO)

### Socket Events
- **Connection / Disconnect**
- `join_room` (private chat room for user pairs)
- `send_message` (payload: `{to, message}`)
- `receive_message` (emit to receiver)

**Example:**
```js
io.on('connection', socket => {
  console.log('User connected', socket.id)

  socket.on('join_room', room => {
    socket.join(room)
  })

  socket.on('send_message', data => {
    const { room, message } = data
    io.to(room).emit('receive_message', message)
  })

  socket.on('disconnect', () => {
    console.log('User disconnected')
  })
})
```

---

## 9. Notification System with Redis (Advanced)

Redis pub/sub helps push notifications across multiple backend instances.

**Use Cases:**
- User receives notification when someone likes/comments/follows.
- New message alert if offline.

**Flow:**
1. Backend publishes event to Redis: `redisClient.publish('notifications', JSON.stringify({ toUserId, type, payload }))`
2. All backend instances subscribed to `notifications` channel.
3. If recipient is online via Socket.IO, push notification in real‑time.
4. If offline, store in DB and deliver later.

**Example:**
```js
// publisher.js
redisClient.publish('notifications', JSON.stringify({ to: userId, type: 'like', postId }))

// subscriber.js
redisSubscriber.on('message', (channel, msg) => {
  const notif = JSON.parse(msg)
  io.to(notif.to).emit('notification', notif)
})
```

---

## 10. Frontend Structure
```
/src
  /pages
    Feed.jsx
    Profile.jsx
    Messages.jsx
    Analytics.jsx
  /components
    PostCard.jsx
    ChatBox.jsx
    NotificationBell.jsx
    Sidebar.jsx
  /contexts
    AuthContext.jsx
    SocketContext.jsx
  /services
    api.js
    socket.js
```

### Features
- **Feed:** posts, likes, comments
- **Profile:** user info, media gallery
- **Messages:** real‑time chat UI
- **Analytics:** charts for engagement (Recharts / Chart.js)
- **Notifications:** toast or dropdown from bell icon

---

## 11. Analytics Dashboard
Track:
- Total posts, likes, comments
- Followers growth over time
- Engagement per post (likes/comments count)
- Active users (daily/weekly)

**Libraries:** Recharts, Chart.js, or D3.

---

## 12. Security & Best Practices
- Password hashing with bcrypt
- JWT in httpOnly cookies (or localStorage with care)
- Input validation with Joi/Validator
- CORS restrictions
- Media upload filtering (file type/size)
- Socket authentication using JWT on connection handshake

---

## 13. Deployment
- **Backend:** Deploy on Railway/Heroku/VPS with PM2 or Docker.
- **Frontend:** Vercel/Netlify.
- **DB:** MongoDB Atlas.
- **Redis:** Managed Redis (Upstash, AWS ElastiCache).
- **Socket.IO Scaling:** Use Redis adapter for horizontal scaling.

---

## 14. Quick Start
1. Clone repo
2. Setup backend:
   ```bash
   cd backend
   npm install
   cp .env.example .env
   npm run dev
   ```
3. Setup frontend:
   ```bash
   cd frontend
   npm install
   npm run dev
   ```
4. Access app at `http://localhost:3000`

---

## 15. Next Steps / Enhancements
- Push notifications (Web Push API)
- Video uploads + streaming support
- Story feature (time‑bound posts)
- Explore tab with trending posts
- Role‑based admin panel
- AI content moderation

---

## 16. References
- [Socket.IO Docs](https://socket.io/docs/)
- [Redis Pub/Sub](https://redis.io/docs/interact/pubsub/)
- [Mongoose Docs](https://mongoosejs.com/)
- [Chart.js](https://www.chartjs.org/)

---

## 17. License
MIT

