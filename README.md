# 3D Dice Roller

A realistic 3D dice roller application built with Vue 3, TypeScript, and CSS 3D transforms.

## Features

- Roll 1-3 dice with realistic physics and 3D animation
- View roll history alongside the dice
- Calculate total values automatically
- Toggle between light and dark theme
- Responsive design for all device sizes

## Technologies

- Vue 3 with Composition API
- TypeScript
- CSS 3D Transforms
- GSAP for animations
- Vite for build tooling

## Getting Started

```bash
# Install dependencies
npm install

# Start the development server
npm run dev

# Build for production
npm run build
```

## How It Works

The application uses CSS 3D transforms to create realistic dice that roll with physics-based animations. GSAP (GreenSock Animation Platform) is used to handle the complex animation sequences, including random rotations, realistic bounces, and subtle tilts for a more natural feel.

The dice values are generated randomly and stored in the roll history, which is displayed in a compact panel alongside the dice. Users can clear the history at any time.

## License

MIT
