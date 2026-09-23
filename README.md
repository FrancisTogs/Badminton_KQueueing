# Badminton Queue Manager 🏸

A real-time, interactive web application built with Svelte 5 to streamline court management, player registration, and matchmaking for badminton sessions. 

## Features

* **Court Management:** Add, remove, and monitor active courts in real-time. Displays current matches and live elapsed play times.
* **Player Registry & Skill Tiers:** Register players and automatically categorize them into Beginner, Intermediate, and Advanced tiers based on skill level (Low/High variants).
* **Smart Matchmaking & Queueing:** 
  * Select 4 available players directly from the dashboard to form a Doubles Match.
  * Visual indicators for Team 1 (Blue) and Team 2 (Red).
  * Automatically prevents duplicate player selection.
* **Live Timers:** Highly optimized, isolated live timers track how long players have been waiting and how long matches have been playing without causing UI lag.
* **Match History & Scoring:** End matches, record final scores, and log them into a persistent Match History ledger.
* **Edit & Modify:** Full modal support to edit player details, update queued matchups, or delete entries on the fly.

## Tech Stack

* **Frontend:** Svelte 5 (utilizing Runes: `$state`, `$derived`, `$effect`, `$props`)
* **Language:** TypeScript / HTML5
* **Styling:** Vanilla CSS (Flexbox/Grid, Custom Themes)

## Prerequisites

Make sure you have [Node.js](https://nodejs.org/) installed on your machine.

## Installation

1. **Clone the repository:**
   ```bash
   git clone [https://github.com/FrancisTogs/Badminton_KQueueing.git](https://github.com/FrancisTogs/Badminton_KQueueing.git)
   cd Badminton_KQueueing