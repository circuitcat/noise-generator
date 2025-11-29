# Noise Helper

A beautiful, privacy-focused web app for generating ambient noise to help with sleep, study, and focus. All audio is generated locally in your browser using the Web Audio API—no server, no downloads, no tracking.

## Features

### 🎵 Five Noise Types
- **White Noise** - Bright and neutral, perfect for masking distractions
- **Pink Noise** - Balanced hiss, often preferred for sleep
- **Brown Noise** - Deep and warm, lower frequencies for relaxation
- **Fan** - Soft whoosh, simulating a gentle fan
- **Rain** - Rainfall sounds for a calming atmosphere

### 🔊 Volume Control
- Adjustable volume slider (0-100%)
- Smooth volume transitions
- Settings persist across sessions

### ⏱️ Timer
- Set automatic stop times: 15, 30, 60, or 90 minutes
- Real-time countdown display
- Automatic fade-out when timer completes

### 📱 Mobile Optimized
- **Wake Lock API** - Prevents iPhone from sleeping while noise is playing (iOS 16.4+)
- Responsive design that works beautifully on all screen sizes
- Touch-friendly controls

### 🔒 Privacy First
- All audio generated locally in your browser
- No network requests after initial page load
- No data collection or tracking
- Settings stored locally in your browser

## How to Use

1. Open `index.html` in a modern web browser
2. Select your preferred noise type
3. Adjust the volume to a comfortable level
4. Optionally set a timer
5. Click **Play** to start

The app will remember your preferences (noise type, volume, timer) for your next visit.

## Technical Details

### Web Audio API
The app uses the Web Audio API to generate noise in real-time:
- White noise is generated using random values
- Different noise colors are created using biquad filters (lowpass/highpass)
- Smooth fade-in/fade-out transitions for better user experience

### Wake Lock API
On supported devices (iOS Safari 16.4+, Chrome, Edge, Firefox), the app uses the Screen Wake Lock API to prevent the device from sleeping while noise is playing. This is especially useful for sleep sessions on mobile devices.

The wake lock:
- Activates automatically when playback starts
- Releases when playback stops
- Re-acquires if the browser tab becomes visible again
- Handles system releases gracefully (low battery, manual lock, etc.)

### Browser Compatibility

**Fully Supported:**
- Chrome/Edge 84+
- Firefox 110+
- Safari 16.4+ (iOS 16.4+)
- Opera 70+

**Partially Supported:**
- Older Safari versions (no Wake Lock API, but audio works)
- Older mobile browsers (may not support Wake Lock API)

The app gracefully degrades on unsupported browsers—all features work except Wake Lock on older devices.

## Setup

No build process or dependencies required! Just:

1. Clone or download this repository
2. Open `index.html` in your web browser
3. That's it!

For local development, you can use any static file server:

```bash
# Using Python
python -m http.server 8000

# Using Node.js (http-server)
npx http-server

# Using PHP
php -S localhost:8000
```

Then open `http://localhost:8000` in your browser.

## Project Structure

```
noise-generator/
├── index.html    # Single-file application (HTML, CSS, JavaScript)
└── README.md     # This file
```

The entire application is contained in a single HTML file for maximum portability and simplicity.

## License

This project is open source and available for personal and commercial use.

## Contributing

Contributions are welcome! Feel free to open issues or submit pull requests.

## Tips

- **For all-night use**: Keep your device on charge
- **Volume safety**: Always keep volume at a comfortable level to protect your hearing
- **Battery**: On mobile devices, Wake Lock will keep the screen on, which uses more battery. Consider keeping your device plugged in for extended sessions
- **Background playback**: The app works best when the browser tab is active. Some browsers may pause audio when the tab is in the background

---

Made with ❤️ for better sleep and focus

