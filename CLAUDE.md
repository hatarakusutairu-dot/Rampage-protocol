# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

RAMPAGE PROTOCOL is a browser-based 3D multiplayer cooperative game where players work together to stop a rampaging giant robot. The entire application is contained in a single `index.html` file.

## Technology Stack

- **Three.js (r128)**: 3D rendering via CDN
- **Supabase**: Real-time multiplayer synchronization and database
- **Vanilla JavaScript**: No build tools or frameworks

## Development

Open `index.html` directly in a browser. No build step required.

## Architecture

### Game Flow
1. Title screen → Team creation/join → Lobby (role selection) → 3D Game

### Multiplayer System
- Teams identified by 4-digit codes
- Supabase tables: `teams`, `players`, `game_states`, `npc_states`
- Real-time sync via Supabase Realtime subscriptions (`subscribeToTeam()`)

### 3D World (Three.js)
- First-person camera with WASD movement + mouse look
- Robot with animated parts (`lArmGrp`, `rArmGrp`, `headMesh`)
- Interactive objects: 3 control panels (A, B, main), 6 NPCs

### Game State
Global `G` object holds all game state:
- `time`, `rampage`, `score`, `rescued`, `recovery`
- `panelA`, `panelB`, `panelMain` (puzzle progression)

### Core Functions
- `initThreeJS()`: Scene setup, robot creation, NPC/panel placement
- `animate()`: Main game loop (movement, interaction, rampage escalation)
- `processInteraction()` / `completeInteraction()`: E-key hold mechanics
- `startGame()` / `endGame()`: Game lifecycle

### Role System
Three roles: Commander (指揮官), Rescue (レスキュー), Scout (情報員) - stored in `players.role`
