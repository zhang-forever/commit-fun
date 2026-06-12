# Commit.fun

[![Svelte](https://img.shields.io/badge/Svelte-5-FF3E00?logo=svelte&logoColor=white)](https://svelte.dev)
[![Vite](https://img.shields.io/badge/Vite-6-646CFF?logo=vite&logoColor=white)](https://vitejs.dev)
[![License](https://img.shields.io/badge/license-MIT-purple)](./LICENSE)

🎮 **Your GitHub commit history, rendered as an 8-bit pixel game map.** Each commit is a tile. Streaks are power-ups. Repos are kingdoms.

## ✨ Features

- **Enter any GitHub username** — instantly render their commit history
- **8-bit pixel art** — commits become tiles on a retro game map
- **Repo kingdoms** — each repository is a themed level with unique tiles
- **Streak power-ups** — longest streaks become golden paths
- **Export as PNG** — share your game map on social media
- **Fast & lightweight** — pure client-side, no backend required

## 🚀 Quick Start

```bash
# Install dependencies
npm install

# Start dev server
npm run dev

# Open http://localhost:5173
```

## 🛠️ Tech Stack

- **Framework:** Svelte 5 + Vite
- **Rendering:** Canvas API for pixel-perfect 8-bit graphics
- **Data:** GitHub API for commit history
- **Aesthetic:** Retro 8-bit game visuals

## 📦 Build

```bash
npm run build
# Output in dist/
```

## 🎨 How It Works

1. Enter a GitHub username
2. Fetch commit history via GitHub API
3. Map commits to 8-bit tiles (color = contribution intensity)
4. Render on canvas with retro game aesthetics
5. Export as PNG for sharing

## 📄 License

MIT
