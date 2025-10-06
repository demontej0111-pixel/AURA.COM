/*******************************
 * server/package.json
 *******************************/
{
  "name": "demon-aura-server",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": { "start": "node server.js" },
  "dependencies": {
    "express": "^4.18.2",
    "mongoose": "^7.3.1",
    "cors": "^2.8.5",
    "dotenv": "^16.0.3",
    "bcryptjs": "^2.4.3",
    "jsonwebtoken": "^9.0.0"
  }
}

/*******************************
 * server/.env.example
 *******************************/
MONGO_URI=your_mongodb_connection_string
JWT_SECRET=your_secret_key

/*******************************
 * server/server.js
 *******************************/
const express = require("express");
const mongoose = require("mongoose");
const cors = require("cors");
require("dotenv").config();

const animeRoutes = require("./routes/anime");
const authRoutes = require("./routes/auth");
const watchlistRoutes = require("./routes/watchlist");

const app = express();
app.use(cors());
app.use(express.json());

mongoose.connect(process.env.MONGO_URI, { useNewUrlParser: true, useUnifiedTopology: true })
  .then(() => console.log("MongoDB connected"))
  .catch(err => console.log(err));

app.use("/anime", animeRoutes);
app.use("/auth", authRoutes);
app.use("/watchlist", watchlistRoutes);

app.listen(5000, () => console.log("Server running on port 5000"));

/*******************************
 * server/models/User.js
 *******************************/
const mongoose = require("mongoose");

const userSchema = new mongoose.Schema({
  username: { type: String, required: true, unique: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  watchlist: [{ type: mongoose.Schema.Types.ObjectId, ref: "Anime" }]
});

module.exports = mongoose.model("User", userSchema);

/*******************************
 * server/models/Anime.js
 *******************************/
const mongoose2 = require("mongoose");
const animeSchema = new mongoose2.Schema({
  title: String,
  description: String,
  genre: [String],
  thumbnail: String,
  episodes: [
    { episodeNumber: Number, title: String, videoUrl: String }
  ]
});
module.exports = mongoose2.model("Anime", animeSchema);

/*******************************
 * server/routes/anime.js
 *******************************/
const express1 = require("express");
const routerAnime = express1.Router();
const Anime = require("../models/Anime");

routerAnime.get("/", async (req, res) => {
  const anime = await Anime.find();
  res.json(anime);
});

routerAnime.get("/:id", async (req, res) => {
  const anime = await Anime.findById(req.params.id);
  res.json(anime);
});

module.exports = routerAnime;

/*******************************
 * server/routes/auth.js
 *******************************/
const express2 = require("express");
const routerAuth = express2.Router();
const bcrypt = require("bcryptjs");
const jwt = require("jsonwebtoken");
const User = require("../models/User");

// Register
routerAuth.post("/register", async (req, res) => {
  const { username, email, password } = req.body;
  const existingUser = await User.findOne({ email });
  if (existingUser) return res.status(400).json({ msg: "User exists" });

  const hashedPassword = await bcrypt.hash(password, 10);
  const user = new User({ username, email, password: hashedPassword });
  await user.save();

  const token = jwt.sign({ id: user._id }, process.env.JWT_SECRET, { expiresIn: "1d" });
  res.json({ token, user: { id: user._id, username, email, watchlist: [] } });
});

// Login
routerAuth.post("/login", async (req, res) => {
  const { email, password } = req.body;
  const user = await User.findOne({ email });
  if (!user) return res.status(400).json({ msg: "User not found" });

  const isMatch = await bcrypt.compare(password, user.password);
  if (!isMatch) return res.status(400).json({ msg: "Invalid credentials" });

  const token = jwt.sign({ id: user._id }, process.env.JWT_SECRET, { expiresIn: "1d" });
  res.json({ token, user: { id: user._id, username: user.username, email, watchlist: user.watchlist } });
});

module.exports = routerAuth;

/*******************************
 * server/routes/watchlist.js
 *******************************/
const express3 = require("express");
const routerWatchlist = express3.Router();
const jwt2 = require("jsonwebtoken");
const User2 = require("../models/User");

const authMiddleware = (req, res, next) => {
  const token = req.header("Authorization");
  if (!token) return res.status(401).json({ msg: "No token" });
  try {
    req.user = jwt2.verify(token, process.env.JWT_SECRET);
    next();
  } catch {
    res.status(401).json({ msg: "Invalid token" });
  }
};

routerWatchlist.post("/:animeId", authMiddleware, async (req, res) => {
  const user = await User2.findById(req.user.id);
  if (!user.watchlist.includes(req.params.animeId)) {
    user.watchlist.push(req.params.animeId);
    await user.save();
  }
  res.json(user.watchlist);
});

module.exports = routerWatchlist;

/*******************************
 * client/package.json
 *******************************/
{
  "name": "demon-aura-client",
  "version": "1.0.0",
  "private": true,
  "dependencies": {
    "axios": "^1.4.0",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "react-router-dom": "^6.14.1",
    "react-scripts": "5.0.1",
    "react-slick": "^0.30.0",
    "slick-carousel": "^1.8.1",
    "react-player": "^2.12.0"
  },
  "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build"
  }
}

/*******************************
 * client/src/index.js
 *******************************/
import React from "react";
import ReactDOM from "react-dom/client";
import App from "./App";
import "./index.css";

const root = ReactDOM.createRoot(document.getElementById("root"));
root.render(<App />);

/*******************************
 * client/src/index.css
 *******************************/
body {
  margin: 0;
  font-family: Arial, sans-serif;
  background-color: #000;
  color: #fff;
}

.slick-slide { padding: 5px; }
.slick-slide img { border-radius: 5px; }

/*******************************
 * client/src/App.js
 *******************************/
import React from "react";
import { BrowserRouter as Router, Routes, Route } from "react-router-dom";
import Home from "./pages/Home";
import AnimeDetail from "./pages/AnimeDetail";
import Player from "./pages/Player";
import Login from "./pages/Login";
import Register from "./pages/Register";

function App() {
  return (
    <Router>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/anime/:id" element={<AnimeDetail />} />
        <Route path="/player/:id" element={<Player />} />
        <Route path="/login" element={<Login />} />
        <Route path="/register" element={<Register />} />
      </Routes>
    </Router>
  );
}

export default App;

/*******************************
 * client/src/components/Navbar.js
 *******************************/
import React from "react";
import { Link } from "react-router-dom";

const Navbar = () => (
  <div className="bg-black text-white flex justify-between items-center p-4 fixed w-full z-50">
    <Link to="/" className="text-2xl font-bold">Demon Aura</Link>
    <div>
      <Link to="/login" className="mr-4 hover:text-red-500 transition">Login</Link>
      <Link to="/register" className="hover:text-red-500 transition">Register</Link>
    </div>
  </div>
);

export default Navbar;

/*******************************
 * client/src/components/AnimeCard.js
 *******************************/
import React from "react";
import { Link } from "react-router-dom";

const AnimeCard = ({ anime }) => (
  <Link to={`/anime/${anime._id}`}>
    <div className="bg-gray-900 rounded-lg overflow-hidden hover:scale-105 transition transform">
      <img src={anime.thumbnail} alt={anime.title} className="w-full h-48 object-cover"/>
      <div className="p-2">
        <h2 className="text-white font-bold">{anime.title}</h2>
      </div>
    </div>
  </Link>
);

export default AnimeCard;