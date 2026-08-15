<div align="center">
<figure aria-label="ASCII art logo for LiveCollab ">
<pre>

╔════════════════════════════════════════════════════════════════════════════════╗
║                                                                                ║
║  ██╗     ██╗██╗   ██╗███████╗ ██████╗ ██████╗ ██╗     ██╗      █████╗ ██████╗  ║
║  ██║     ██║██║   ██║██╔════╝██╔════╝██╔═══██╗██║     ██║     ██╔══██╗██╔══██╗ ║
║  ██║     ██║██║   ██║█████╗  ██║     ██║   ██║██║     ██║     ███████║██████╔╝ ║
║  ██║     ██║╚██╗ ██╔╝██╔══╝  ██║     ██║   ██║██║     ██║     ██╔══██║██╔══██╗ ║
║  ███████╗██║ ╚████╔╝ ███████╗╚██████╗╚██████╔╝███████╗███████╗██║  ██║██████╔╝ ║
║  ╚══════╝╚═╝  ╚═══╝  ╚══════╝ ╚═════╝ ╚═════╝ ╚══════╝╚══════╝╚═╝  ╚═╝╚═════╝  ║
║                                                                                ║
╚════════════════════════════════════════════════════════════════════════════════╝
</pre>
</figure>


**An Engineer-First, Real-Time Collaborative Code Editor & Development Environment**

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Release](https://img.shields.io/badge/release-v0.1.0-blue.svg)]()
[![Build Status](https://img.shields.io/badge/Build-Passing-brightgreen.svg)]()
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)
[![Status](https://img.shields.io/badge/Status-Active%20Development-orange.svg)]()

<p align="center">
  <a href="#overview">Overview</a> •
  <a href="#key-features">Key Features</a> •
  <a href="#tech-stack">Tech Stack</a> •
  <a href="#architecture">Architecture</a> •
  <a href="#getting-started">Getting Started</a> •
  <a href="#contributing">Contributing</a> •
  <a href="#license">License</a>
</p>

</div>

---

## 📌 Table of Contents
- [System Architecture](#-system-architecture)
- [End-to-End User & Data Flow](#-end-to-end-user--data-flow)
- [Core Engineering Features](#-core-engineering-features)
- [Deep Dive: Project Structure](#-deep-dive-project-structure)
- [Local Setup & Deployment](#local-setup-deployment)
- [Visual Gallery](#-visual-gallery)

---

## 🏗️ System Architecture

LiveCollab is built on a **Custom Next.js Server** architecture to natively support WebSockets alongside Next.js App Router for server-rendered UI. 

```mermaid
graph TD
    Client[Client Browser]
    
    subgraph Frontend [Next.js App Router]
        Monaco[Monaco Editor]
        Whiteboard[Canvas API]
        ChatUI[Chat Interface]
        Sidebar[File Explorer]
    end

    subgraph Backend [Custom Node Server]
        NextHandle[Next.js Request Handler]
        SocketIO[Socket.IO Server]
        RoomStore[(In-Memory Room State)]
    end

    subgraph External [External Services]
        Piston[Piston Code Execution Engine]
    end

    Client -->|HTTP/Next.js| NextHandle
    Client <-->|WebSocket| SocketIO
    
    Monaco <-->|Cursor/Code Sync| SocketIO
    Whiteboard <-->|Draw Line Sync| SocketIO
    ChatUI <-->|Message Event| SocketIO
    
    SocketIO <--> RoomStore
    Monaco -->|Compile Request| Piston
```

### Architectural Decisions

1. **Custom HTTP Server (`server.ts`)**: Instead of relying on standard serverless API routes (which cannot maintain stateful WebSocket connections), LiveCollab employs a custom HTTP server wrapping both `next/server` and `socket.io`.
2. **In-Memory State Management**: For optimal real-time performance without database bottlenecks, session states (files, user lists, active cursors) are managed via an in-memory `Map` data structure on the Node server.

---

## 🔄 End-to-End User & Data Flow

### 1. Room Creation & Hydration
- User navigates to the landing page and clicks **Create Room**.
- The client generates a unique `roomId` (via `nanoid`) and redirects to `/room/[roomId]`.
- Upon mounting, `useSocket` establishes a persistent TCP connection to the Socket.IO server.
- The server initializes an isolated namespace for the `roomId` inside the `rooms` Map.
- **Hydration:** The server emits `sync-files` and `active-users` to the new client, seeding the initial Monaco editor state and file tree.

### 2. Bi-Directional Event Broadcasting
- **File Edits**: Typing in the Monaco Editor triggers `onChange`. The frontend emits a `code-change` event containing `{ roomId, fileName, code }`.
- **Cursor Tracking**: Monaco's cursor observer fires `cursor-move`. The server broadcasts the exact `{ line, column }` alongside the user's generated hex color code to other clients.
- **Whiteboard Engine**: Canvas `mousemove` events generate coordinate deltas `(prevPoint, currentPoint)` sent as `draw-line` events. The server performs fan-out broadcasting to render lines simultaneously on all peers.

### 3. Code Execution
- User selects a file and clicks the **Run** button.
- The frontend packages the raw string content and the designated language.
- A REST `POST` payload is dispatched to the Piston API.
- Piston spins up an ephemeral Docker container, executes the code, and pipes `stdout`/`stderr` back to the client's terminal UI component.

---

## ⚡ Core Engineering Features

### 🧩 Monaco Editor Integration
Utilizing `@monaco-editor/react` to provide a VS Code-equivalent editing experience on the web. Features integrated:
- Rich syntax highlighting and auto-completion.
- Custom decorations injected dynamically for remote cursor rendering.
- Virtualized DOM rendering for massive files.

### 🔌 Socket.IO Event Infrastructure
An intricate set of WebSockets events manages the entire collaborative lifecycle:
- `join-room` / `disconnect`: Manages the ephemeral user session lifecycle and auto-purges zombies.
- `selection-change`: Broadcasts text highlighting selections in real-time.
- `create-file` / `delete-file`: Mutations to the virtual file tree are synced with optimistic UI updates.

### 🎨 Fluid UI & Animation System
Built with **Tailwind CSS v4** and **Framer Motion**:
- Hardware-accelerated UI transitions.
- Dynamic theme switching (VS Dark, VS Light, High Contrast) injected dynamically into Monaco and Tailwind context.

---

## 📁 Deep Dive: Project Structure

```text
LiveCollab/
├── src/
│   ├── app/
│   │   ├── page.tsx              # Landing page component
│   │   ├── room/[roomId]/page.tsx# Core room engine (initializes socket)
│   │   ├── layout.tsx            # Global metadata and providers
│   │   └── globals.css           # Tailwind v4 injection
│   ├── components/
│   │   ├── CodeEditor.tsx        # Monaco wrapper with socket sync logic
│   │   ├── Chat.tsx              # Real-time WebSocket chat layer
│   │   ├── Whiteboard.tsx        # 2D Canvas context controller
│   │   ├── Sidebar.tsx           # Virtual file tree state manager
│   │   ├── LanguageSelector.tsx  # Piston API language parser mapping
│   │   └── ThemeSelector.tsx     # Context-aware theme injector
│   ├── hooks/
│   │   └── useSocket.ts          # Singleton pattern for Socket.io-client
│   └── lib/
│       ├── utils.ts              # clsx + tailwind-merge utilities
│       └── piston.ts             # Piston API execution wrapper
├── server.ts                     # The Heart: Custom Node/Express/Socket server
├── package.json                  # Dependencies (tsx, next, socket.io)
└── tsconfig.json                 # Strict TypeScript configuration
```

---

<a id="local-setup-deployment"></a>
## 🛠️ Local Setup & Deployment

### Prerequisites
- Node.js `20.x+` (due to Next.js 14+ requirements)
- Optional: Python/C++ compiler locally if attempting to self-host Piston, but by default, it relies on the public Piston API.

### Development Environment

```bash
git clone https://github.com/Rakesh2901/LiveCollab.git
cd LiveCollab

# Install standard dependencies
npm install

# Run the custom server (DO NOT use 'next dev' directly)
npm run LiveCollab
# Under the hood, this runs: `tsx server.ts`
```

### Production Build
Because of the custom server, standard Vercel deployments (which enforce serverless architecture) are not supported. LiveCollab must be deployed to a persistent container/VPS service like **Railway**, **Render**, or **AWS EC2**.

```bash
npm run build
npm start # Runs NODE_ENV=production tsx server.ts
```

---

## 📸 Visual Gallery

### Application Interface
<img width="1920" height="1080" alt="image" src="public\Images\Screenshot 2026-08-15 053729.png" />
### Code Editor in Action
<img width="1920" height="1080" alt="image" src="public\Images\Screenshot 2026-08-15 053434.png" />
<img width="1920" height="1080" alt="image" src="public\Images\Screenshot 2026-08-15 053604.png" />

---

## 👨‍💻 Author

**Rakesh**
- GitHub: [@Rakesh2901](https://github.com/Rakesh2901)
- LinkedIn: [@Rakesh](https://www.linkedin.com/in/rakesh-712923324/)

*Made with ❤️ by Rakesh*
