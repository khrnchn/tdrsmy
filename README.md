# tadarus.my (tdrsmy)

**[tadarus.my](https://tadarus.my)** is a web app for Malaysian Muslims to track group Quran reading (tadarus) during Ramadan. Join a "geng," log your pages, and khatam the Quran together—featuring a leaderboard and a shareable "Tadarus Wrapped" summary by Ramadan’s end. Built for #GodamSahur 2025, it’s a modern take on a Ramadan tradition.

## Features

### 1. Tadarus Circles
- **Create or Join**: Start your own "Tadarus Geng" or join one with an invite code.
- **Group Goal**: Set a target (default: 604 pages, one Quran) and watch your circle’s progress grow.
- **Real-time Updates**: Progress bar updates live as members log their tadarus, powered by Pusher.

### 2. Tadarus Logging
- **Quick Logging**: Add your pages read (e.g., "5 pages" or "Surah Al-Baqarah") in a snap.
- **Your Contribution**: See how much you’ve added to your circle’s total, keeping it group-focused.
- **Smart Checks**: Ensures valid page counts and links logs to your circle.

### 3. Leaderboard
- **Top Gengs**: Ranks the top 10 circles by total pages read, refreshed in real-time.
- **Friendly Rivalry**: Spot which "Geng Quran" is ahead—perfect for some Ramadan motivation.
- **Clean Design**: Uses Shadcn/UI’s Table component for a sharp, modern layout.

### 4. Tadarus Wrapped
- **Ramadan Recap**: Rolls out after March 31, 2025—shows your circle’s total pages, your share, and the top reader.
- **Shareable Card**: Get a custom PNG (e.g., “Geng Masjid KL khatam 2x!”) via Canvas API.
- **Digital Memory**: A keepsake to share on WhatsApp or X, celebrating your Ramadan journey.

## Tech Stack
- **Framework**: Next.js 14 (App Router)
- **Runtime**: Bun
- **UI**: Shadcn/UI with Tailwind CSS
- **Database**: Drizzle ORM + Vercel Postgres
- **Auth**: Clerk
- **Real-time**: Pusher
- **Deployment**: Vercel (live at tadarus.my)

## Development
- **Setup**: `bun create next-app tdrsmy --typescript`, then `bun add drizzle-orm @vercel/postgres @clerk/nextjs pusher pusher-js shadcn-ui`.
- **Run**: `bun run dev`
- **Deploy**: `bunx vercel --prod` (linked to tadarus.my).

## Status
- Built during Ramadan 2025 (March 9–April 1).
- Submitted to #GodamSahur 2025 by April 1.

---

**tdrsmy** – Khatam made simple. Ramadan made social.