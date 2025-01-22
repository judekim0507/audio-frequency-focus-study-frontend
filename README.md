# Audio Frequency Focus Study (Frontend)

A web-based frontend application for studying the effects of binaural beats and pure tones on focus levels using EEG data from any Muse EEG headband.

![Focus Study Interface](/assets/screenshot_1.png)

## Overview

This is the frontend portion of the focus study project. It interfaces with a separate backend service that handles EEG data processing from any Muse EEG headband. The application features:

- Real-time EEG focus level monitoring
- Binaural beats generation (40Hz and 15Hz)
- Pure tone control (240Hz)
- Focus level calibration
- Data collection and export

## Features

- **Multiple Frequency Modes:**
  - 40Hz Binaural Beat (200Hz L, 240Hz R)
  - 15Hz Binaural Beat (200Hz L, 215Hz R)
  - 240Hz Pure Tone Control

- **Real-time Monitoring:**
  - Live focus level visualization
  - Connection status monitoring
  - Session history tracking

- **Research Tools:**
  - Focus level calibration
  - CSV data export
  - Participant management
  - Automated timing controls

## Technical Details

- **Frontend Stack:**
  - Svelte + TypeScript
  - TailwindCSS for styling
  - Web Audio API for sound generation

- **Backend Integration:**
  - WebSocket connection (ws://localhost:8000/ws)
  - Requires separate backend service for EEG data processing
  - Compatible with any Muse EEG headband

## Setup

1. Clone the repository:

```bash
git clone https://github.com/judekim0507/audio-frequency-focus-study-frontend.git
```

2. Install dependencies:

```bash
npm install
```

3. Start the development server:

```bash
npm run dev
```

4. Ensure the EEG WebSocket backend server is running at `ws://localhost:8000/ws`

## Backend Requirements

This frontend application requires a compatible backend service that:
- Connects to any Muse EEG headband
- Processes raw EEG data
- Provides focus level calculations
- Handles WebSocket connections
- Manages calibration process

[For the backend implementation, please refer to the separate backend repository.](https://github.com/judekim0507/audio-frequency-focus-study-backend)

## Usage

1. Start the backend server
2. Launch the frontend application
3. Enter participant name
4. Perform focus level calibration
5. Select desired frequency mode
6. Run 2-minute sessions
7. Export collected data as CSV

## Research Protocol

Each session consists of:
- 2-minute calibration period
- 2-minute frequency exposure
- 1-minute cooldown between sessions
- Automated data collection

## License

MIT License

## Contributors

- Jude Kim - Burnaby North Secondary School