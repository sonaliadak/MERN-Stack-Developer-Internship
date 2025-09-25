# E‑Commerce Platform — Project README

## 1. Project Overview
A full‑stack e‑commerce platform built with the MERN stack (MongoDB, Express, React, Node) that supports:

- User authentication (JWT)
- Product listings with search and filters
- Shopping cart, checkout and Stripe payment integration
- Admin dashboard for inventory & order management
- Advanced add‑on: AI‑powered product recommendations

This README contains architecture, setup instructions, API spec, data models, frontend structure, and implementation notes you can use to build, test and deploy the project.

---

## 2. Tech Stack
- **Database:** MongoDB (Mongoose)
- **Backend:** Node.js + Express.js
- **Frontend:** React (create‑react‑app or Vite)
- **Auth:** JSON Web Tokens (JWT)
- **Payments:** Stripe (Checkout or Payment Intents)
- **AI Recommendations (add‑on):** Embeddings + vector search (Pinecone/Weaviate/FAISS) or collaborative filtering
- **Dev tools:** ESLint, Prettier, dotenv, Git

---

## 3. High‑Level Architecture

```
[React Frontend] <--HTTPS--> [Express API (Node)] <--MongoDB--> [MongoDB Atlas]
                                   |
                                   +--> [Stripe API]
                                   +--> [Vector DB / Embeddings (optional)]
```

- Frontend calls REST API for auth, products, cart, orders.
- Backend verifies JWT, manages sessions statelessly.
- Stripe webhooks notify backend of payment events.
- Recommendation service either runs in backend or as a microservice.

---

## 4. Folder Structure (suggested)

```
/ecommerce-root
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

## 5. Environment Variables (`.env`)

Create `.env` in `/backend` and `/frontend` as needed.

**Backend `.env`**

```
PORT=4000
MONGO_URI=mongodb+srv://<user>:<pass>@cluster0.mongodb.net/ecommerce?retryWrites=true&w=majority
JWT_SECRET=super_secret_jwt_key
JWT_EXPIRES_IN=7d
STRIPE_SECRET_KEY=sk_test_XXXXXXXXXXXXXXXX
STRIPE_WEBHOOK_SECRET=whsec_XXXXXXXXXXXX
FRONTEND_URL=http://localhost:3000
RECOMMENDATION_VECTOR_DB_URL=...
OPENAI_API_KEY=sk-... (optional)
```

**Frontend `.env`**

```
VITE_API_BASE_URL=http://localhost:4000/api
VITE_STRIPE_PUBLISHABLE_KEY=pk_test_XXXXXXXX
```

---

## 6. Database Models (Mongoose)

### User (models/User.js)
```js
const mongoose = require('mongoose')
const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true }, // hashed
  role: { type: String, enum: ['user','admin'], default: 'user' },
  createdAt: { type: Date, default: Date.now }
})
module.exports = mongoose.model('User', userSchema)
```


### Product (models/Product.js)
```js
const productSchema = new mongoose.Schema({
  title: { type: String, required: true, index: true },
  description: String,
  price: { type: Number, required: true },
  currency: { type: String, default: 'USD' },
  images: [String],
  categories: [String],
  tags: [String],
  stock: { type: Number, default: 0 },
  attributes: Object, // e.g. { color: 'red', size: 'M' }
  createdAt: { type: Date, default: Date.now }
})
module.exports = mongoose.model('Product', productSchema)
```


### Cart & Order (models/Order.js)
```js
const orderSchema = new mongoose.Schema({
  user: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  items: [
    {
      product: { type: mongoose.Schema.Types.ObjectId, ref: 'Product' },
      quantity: Number,
      priceAtPurchase: Number
    }
  ],
  subtotal: Number,
  shipping: Number,
  total: Number,
  currency: { type: String, default: 'USD' },
  paymentStatus: { type: String, enum: ['pending','paid','failed'], default: 'pending' },
  stripePaymentIntentId: String,
  createdAt: { type: Date, default: Date.now }
})
module.exports = mongoose.model('Order', orderSchema)
```

---

## 7. Backend API — Key Endpoints
Base path: `/api`

### Auth
- `POST /api/auth/register` — body: `{name,email,password}` -> create user, return JWT
- `POST /api/auth/login` — body: `{email,password}` -> validate, return JWT
- `GET /api/auth/me` — header `Authorization: Bearer <token>` -> return user

### Products
- `GET /api/products` — query: `q`, `category`, `minPrice`, `maxPrice`, `sort`, `page`, `limit` (search + filters + pagination)
- `GET /api/products/:id` — product details
- `POST /api/products` — (admin) create product
- `PUT /api/products/:id` — (admin) update
- `DELETE /api/products/:id` — (admin) remove

Search implementation notes: use MongoDB text index on `title` and `description` and combine with filters.

### Cart / Checkout
- `POST /api/cart` — update cart server side (optional) or store cart in client
- `POST /api/checkout` — create Stripe PaymentIntent / Checkout Session
  - body: `{ items: [{productId, quantity}], shipping, currency }`
  - return client secret or checkout URL
- `POST /api/webhooks/stripe` — (Stripe webhook) — handle `payment_intent.succeeded` or `checkout.session.completed` to mark order paid

### Orders
- `GET /api/orders` — (user) list user orders
- `GET /api/orders/:id` — (user/admin) order details
- `PATCH /api/orders/:id` — (admin) update shipping / status

### Admin
- `GET /api/admin/stats` — (admin) sales numbers, low stock products
- `PUT /api/admin/product/:id/stock` — change inventory

---

## 8. JWT Auth Middleware (example)
```js
// middlewares/auth.js
const jwt = require('jsonwebtoken')
const User = require('../models/User')
module.exports = async function(req,res,next){
  const header = req.headers.authorization
  if(!header) return res.status(401).json({message:'No token'})
  const token = header.split(' ')[1]
  try{
    const payload = jwt.verify(token, process.env.JWT_SECRET)
    req.user = await User.findById(payload.id).select('-password')
    next()
  }catch(err){
    res.status(401).json({message:'Invalid token'})
  }
}
```

---

## 9. Stripe Integration (Server Side example)
Install Stripe SDK: `npm i stripe`

**Create Checkout Session**
```js
const stripe = require('stripe')(process.env.STRIPE_SECRET_KEY)
app.post('/api/checkout', async (req,res)=>{
  const { items, successUrl, cancelUrl } = req.body
  const line_items = items.map(it => ({ price_data: {
    currency: 'usd', product_data: { name: it.name }, unit_amount: Math.round(it.price * 100)
  }, quantity: it.quantity }))
  const session = await stripe.checkout.sessions.create({
    payment_method_types: ['card'],
    line_items,
    mode: 'payment',
    success_url: successUrl,
    cancel_url: cancelUrl
  })
  res.json({ url: session.url })
})
```

**Webhook handler**
```js
app.post('/api/webhooks/stripe', express.raw({type:'application/json'}), (req,res)=>{
  const sig = req.headers['stripe-signature']
  try{
    const event = stripe.webhooks.constructEvent(req.body, sig, process.env.STRIPE_WEBHOOK_SECRET)
    if(event.type === 'checkout.session.completed'){
      const session = event.data.object
      // fulfill order: find temporary cart by session.client_reference_id etc.
    }
    res.json({received:true})
  }catch(err){
    res.status(400).send(`Webhook error: ${err.message}`)
  }
})
```

---

## 10. Frontend — React App Structure
Suggested pages/components:

```
/src
  /pages
    Home.jsx
    ProductList.jsx
    ProductDetail.jsx
    Cart.jsx
    Checkout.jsx
    Profile.jsx
    AdminDashboard.jsx
  /components
    Header.jsx
    Footer.jsx
    ProductCard.jsx
    SearchBar.jsx
    Filters.jsx
  /contexts
    AuthContext.jsx
    CartContext.jsx
  /services
    api.js         // axios wrapper with JWT attached
    productService.js
    authService.js
```

### Auth flow (frontend)
- Save JWT in `localStorage` or `httpOnly cookie` (httpOnly is safer — requires server support)
- Provide `AuthContext` and `PrivateRoute` wrapper for protected pages

---

## 11. Search & Filters Implementation
- Add text index in MongoDB for `title` and `description`:
  ```js
  productSchema.index({ title: 'text', description: 'text' })
  ```
- Combine `text` search with field filters and range queries:
  - `categories`, `tags` as `$in` queries
  - `price` with `$gte`/`$lte`
- Support sorting: `price`, `createdAt`, `relevance`

---

## 12. Admin Dashboard
Key features:
- Product CRUD (images upload to S3 or other storage)
- Inventory and low stock alerts
- Order list and status updates
- Sales reports (basic): daily/weekly revenue, top products

Implementation notes:
- Protect routes with `role === 'admin'` middleware
- For file uploads, use signed S3 uploads or a server route that streams directly to cloud storage

---

## 13. AI‑Powered Product Recommendations (Add‑On)
Two approaches (pick one depending on scale and budget):

### A. Simple content‑based recommendations (no external services)
- Build product embeddings from product metadata (title, description, tags) using TF‑IDF or sentence embeddings (OpenAI / Hugging Face).
- Store embeddings in a vector index (FAISS or an in‑process matrix).
- At runtime, given a product or user history, compute embedding and return nearest neighbors by cosine similarity.

**Pros:** quick to prototype, no heavy infra.  
**Cons:** limited personalization if user history is small.

### B. Collaborative filtering (personalized)
- Use user purchase data to compute item‑item co‑occurrence matrix and recommend based on purchased together counts (item‑to‑item collaborative filtering).
- Maintain periodic batch jobs to compute similarities.

**Hybrid:** combine content and collaborative scores.

**Example pseudocode (Node.js) — cosine similarity on embedding vectors**
```js
function cosine(a,b){ /* dot / (||a||*||b||) */ }
// embeddings: { productId: [0.1,0.2,...], ... }
function recommendForProduct(productId, embeddings, topN=8){
  const base = embeddings[productId]
  const scores = Object.entries(embeddings).map(([id,vec])=>({ id, score: cosine(base,vec) }))
  return scores.sort((a,b)=>b.score-a.score).slice(1,topN+1)
}
```

**Production tip:** use a managed vector DB (Pinecone, Weaviate, Supabase Vector) for scaling and fast similarity search.

---

## 14. Dev & Testing
- Use Postman or Insomnia to test API endpoints.
- Unit tests: Jest (backend) + React Testing Library (frontend).
- Lint: ESLint + Prettier configured in repo.

---

## 15. Deployment
- **Backend:** Host on Heroku / Railway / Vercel (serverless functions) or VPS. Use environment variables and connect to managed MongoDB (Atlas).
- **Frontend:** Deploy to Vercel, Netlify or static hosting.
- **Stripe Webhook:** make sure endpoint is reachable (use ngrok locally for testing).
- **CORS:** restrict to your frontend domain(s).

---

## 16. Security & Best Practices
- Store passwords hashed with bcrypt (salt rounds 10–12).
- Use `httpOnly` cookies for JWT where possible to prevent XSS token theft.
- Rate limit auth endpoints and payment endpoints.
- Validate and sanitize all user inputs.
- Ensure CORS and CSRF protections (esp. when using cookies).
- Use strong secrets and rotate keys periodically.

---

## 17. Example: Minimal `server.js` (Backend Entry)
```js
require('dotenv').config()
const express = require('express')
const mongoose = require('mongoose')
const app = express()
app.use(express.json())

mongoose.connect(process.env.MONGO_URI)
  .then(()=>console.log('Mongo connected'))
  .catch(err=>console.error(err))

// mount routes (auth, products, orders...)
app.use('/api/auth', require('./routes/auth'))
app.use('/api/products', require('./routes/products'))
app.use('/api/checkout', require('./routes/checkout'))

app.listen(process.env.PORT || 4000, ()=>console.log('Server running'))
```

---

## 18. Quick Start (Development)
1. Clone repo
2. `cd backend` → `npm install` → set `.env` → `npm run dev`
3. `cd frontend` → `npm install` → set `.env` → `npm run dev`
4. Use Postman to create a user, create products (admin), then test checkout flow.

---

## 19. Next Steps / Enhancements
- Add product image processing and CDN.
- Add search autocomplete and fuzzy search (ElasticSearch or Typesense).
- Add server‑side rendering (Next.js) for SEO and initial load performance.
- Add AB testing to recommendation models.
- Add analytics and error tracking (Sentry, Segment).

---

## 20. References & Useful Libraries
- `bcryptjs` or `bcrypt` for password hashing
- `jsonwebtoken` for JWT
- `stripe` SDK
- `mongoose` for DB models
- `axios` for frontend API calls
- (Optional) `@pinecone-database/client`, `faiss`, `weaviate-client` for vector search

---

## 21. License
MIT
