# ROBA MuJoCo Simulator

A MuJoCo WebAssembly-based physics simulator running entirely in the browser.

## Quick Start

1. Install dependencies:
```bash
npm install
```

2. Start the local server:
```bash
python3 -m http.server 8100
```

3. Open your browser:
```
http://localhost:8100
```

## Project Structure

```
project/
├── src/
│   ├── main.js           # Main Three.js + MuJoCo setup, render loop
│   └── mujocoUtils.js    # Asset loading, GUI setup, scene management
├── assets/scenes/
│   ├── *.xml             # MuJoCo scene definition files
│   ├── index.json        # Scene file manifest
│   └── meshes/           # STL/OBJ mesh files
├── index.html            # Entry point with GUI styling
├── FULL_DOCS.md          # Complete technical documentation
└── package.json          # Dependencies
```

## How to Animate

Add animation code in `src/main.js` inside the render loop, before `mujoco.mj_step()`:

```javascript
if (!this.params["paused"]) {
  const t = this.data.time;

  // Set actuator targets (position actuators = target angle in radians)
  this.data.ctrl[0] = 0.5 * Math.sin(t * 3);  // sinusoidal motion

  // Smooth easing
  const phase = (t % 2.0) / 2.0;
  const ease = (1 - Math.cos(phase * Math.PI)) / 2;
  this.data.ctrl[1] = -1.5 * ease;

  mujoco.mj_step(this.model, this.data);
}
```

## Controls

The GUI panel (right side) provides:
- **Scene selector** and pause/resume
- **Actuator sliders** for real-time joint control
- **Noise settings** for robustness testing

## Technical Details

- **MuJoCo:** 2.3.1 (WebAssembly)
- **Three.js:** 3D rendering with WebGL
- **Physics Timestep:** 0.002s (500 Hz)
- **Actuator Type:** Position-controlled (PD controller)

See `FULL_DOCS.md` for complete technical documentation.

## Browser Compatibility

- Chrome/Edge (recommended)
- Firefox
- Safari (WebAssembly support required)
