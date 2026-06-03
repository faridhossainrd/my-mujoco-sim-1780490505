# ROBA MuJoCo Simulator - Complete Technical Documentation

## Table of Contents
1. [Overview](#overview)
2. [Architecture](#architecture)
3. [How Position Actuators Work](#how-position-actuators-work)
4. [Asset Loading Pipeline](#asset-loading-pipeline)
5. [Three.js Rendering Engine](#threejs-rendering-engine)
6. [GUI Controls System](#gui-controls-system)
7. [Physics Simulation Loop](#physics-simulation-loop)
8. [Coordinate System Conversions](#coordinate-system-conversions)
9. [Adding New Scenes](#adding-new-scenes)
10. [Adding New Meshes](#adding-new-meshes)
11. [Customization Guide](#customization-guide)
12. [Troubleshooting](#troubleshooting)
13. [Performance Optimization](#performance-optimization)

---

## Overview

The ROBA MuJoCo Simulator is a **WebAssembly-based physics simulation platform** that runs entirely in the browser. It combines:

- **MuJoCo 2.3.1 WebAssembly** - Physics engine compiled to WASM
- **Three.js** - 3D rendering library
- **lil-gui** - Modern GUI control panel

**Key Features:**
- No server-side physics computation required
- Real-time actuator control via GUI sliders or JavaScript code
- Custom ROBA-branded dark theme UI
- Support for complex mesh geometry (STL/OBJ files)
- Mesh bundle system for fast asset loading
- WebGL rendering with shadows and lighting

---

## Architecture

### File Structure
```
project/
├── index.html                    # Entry point with GUI styling
├── package.json                  # Dependencies (three, mujoco-js, esbuild)
├── README.md                     # Quick start guide
├── FULL_DOCS.md                  # This documentation
├── src/
│   ├── main.js                   # Main Three.js + MuJoCo setup, render loop
│   ├── mujocoUtils.js            # Asset loading, GUI setup, scene management
│   └── utils/
│       ├── DragStateManager.js   # Mouse drag interaction for bodies
│       ├── Debug.js              # Debug utilities
│       └── Reflector.js          # Floor reflection effects
├── assets/
│   ├── favicon.png               # ROBA logo
│   └── scenes/
│       ├── *.xml                 # MuJoCo scene definition files
│       ├── index.json            # Scene file manifest for the loader
│       ├── meshes_bundle.gz      # Compressed mesh bundle (if meshes present)
│       └── meshes/               # Individual mesh files (STL/OBJ)
```

### Data Flow
```
index.html
  └── src/main.js (MuJoCoDemo class)
        ├── Loads MuJoCo WASM module
        ├── Downloads scene assets via mujocoUtils.js
        │     ├── Fetches index.json (file manifest)
        │     ├── Loads meshes_bundle.gz (if present, single compressed download)
        │     │     └── Decompresses via native DecompressionStream API
        │     └── Loads XML scene files and remaining assets
        ├── Creates MuJoCo model from XML
        ├── Builds Three.js scene from MuJoCo bodies
        └── Render loop:
              ├── Read GUI actuator values → data.ctrl[]
              ├── Execute your animation code
              ├── mujoco.mj_step() (physics step)
              └── Sync Three.js meshes with MuJoCo body positions
```

---

## How Position Actuators Work

MuJoCo supports several actuator types. This simulator primarily uses **position actuators** with PD control:

```
force = kp * (target_angle - current_angle) - kd * velocity
```

- `data.ctrl[i]` = target angle in **radians** (not torque/force)
- The actuator generates whatever force is needed to reach the target (within `forcerange` limits)
- `kp` = proportional gain (stiffness). Higher = faster, stiffer response
- Setting all `ctrl` to 0 returns to the default pose

### Practical Usage
```javascript
// Set joint target to -1.5 radians
this.data.ctrl[0] = -1.5;

// Smooth sinusoidal motion
this.data.ctrl[0] = 0.5 * Math.sin(this.data.time * 3);

// Eased transition
const phase = (this.data.time % 2.0) / 2.0;
const ease = (1 - Math.cos(phase * Math.PI)) / 2;
this.data.ctrl[0] = -1.5 * ease;
```

---

## Asset Loading Pipeline

### Scene Asset Loading (mujocoUtils.js)

1. **Fetch `index.json`** - Lists all scene files to load
2. **Try mesh bundle** - Fetch `meshes_bundle.gz` (single compressed file with all STL/OBJ meshes)
   - Uses native `DecompressionStream` for gzip decompression
   - Parses binary manifest to extract individual files
   - Writes each mesh to MuJoCo's WASM virtual filesystem
3. **Fall back to individual files** - If no bundle, load files listed in `index.json` individually
4. **Load XML scenes** - Write scene XML files to WASM filesystem
5. **Initialize model** - `mujoco.mj_loadXML()` creates the physics model

### Mesh Bundle Format
```
[4 bytes LE: header length][JSON manifest][concatenated binary data]
```
- Manifest maps relative paths to `{offset, size}` in the data section
- Entire file is gzip compressed
- Reduces 50+ HTTP requests to 1, and compresses binary meshes ~60%

### Adding Files to the Virtual Filesystem
```javascript
// Write a file to MuJoCo's WASM filesystem
mujoco.FS.writeFile('/working/filename.xml', textContent);
mujoco.FS.writeFile('/working/meshes/part.stl', binaryUint8Array);

// Create directories
mujoco.FS.mkdir('/working/meshes');
```

---

## Three.js Rendering Engine

### Scene Setup
- **Camera**: PerspectiveCamera at position (2.0, 1.7, 1.7), 45-degree FOV
- **Lighting**: Ambient light + directional light with shadow mapping
- **Controls**: OrbitControls for camera rotation/zoom, DragStateManager for body interaction
- **Background**: Black with optional skybox from MuJoCo scene definition

### Body Synchronization
Each render frame, Three.js objects are synced with MuJoCo body positions:
```javascript
// For each body in the MuJoCo model:
// 1. Get position from MuJoCo (Z-up)
// 2. Convert to Three.js (Y-up): (x, z, -y)
// 3. Get quaternion from MuJoCo and convert
// 4. Apply to Three.js mesh
```

### Mesh Rendering
- MuJoCo geom types map to Three.js geometries (box, sphere, capsule, mesh)
- STL/OBJ meshes loaded from WASM filesystem via Three.js loaders
- Materials use MuJoCo rgba colors with optional reflectance

---

## GUI Controls System

The GUI panel (right side) is built with lil-gui and provides:

### Controls
- **Scene selector**: Switch between available XML scenes
- **Pause/Resume**: Toggle simulation
- **Actuator sliders**: One slider per actuator, range = ctrlrange from XML
- **Noise settings**: Add random noise to control signals for testing

### GUI Styling (in index.html)
```css
.lil-gui {
    --background-color: #0D0D0D;
    --title-text-color: #22C55E;    /* Green accent */
    --widget-color: #1a1a2e;
    --hover-color: #252540;
    --number-color: #22C55E;
    --string-color: #22C55E;
}
```

### Customizing GUI
GUI folders and sliders are created automatically from the MuJoCo model's actuator definitions. To add custom controls, modify `setupGUI()` in `mujocoUtils.js`.

---

## Physics Simulation Loop

### Core Loop (in main.js render function)
```javascript
if (!this.params["paused"]) {
    // 1. Apply GUI slider values to actuators
    // 2. YOUR ANIMATION CODE HERE (set data.ctrl[] values)
    // 3. Add control noise if enabled
    // 4. Step physics
    mujoco.mj_step(this.model, this.data);
}

// 5. Sync Three.js visuals with MuJoCo state
// 6. Render Three.js scene
```

### Timestep
- Default physics timestep: 0.002s (500 Hz)
- Configurable in XML: `<option timestep="0.002"/>`
- Multiple physics steps per render frame maintain real-time

### Accessing Simulation Data
```javascript
this.data.time          // Current simulation time (seconds)
this.data.ctrl[i]       // Actuator control input (write target angles here)
this.data.qpos[i]       // Joint positions (read-only for animation)
this.data.qvel[i]       // Joint velocities
this.data.sensordata[i] // Sensor readings (if defined in XML)
```

---

## Coordinate System Conversions

### MuJoCo (Z-up) to Three.js (Y-up)

| MuJoCo | Three.js |
|--------|----------|
| X (forward) | X (same) |
| Y (left) | -Z (into screen) |
| Z (up) | Y (up) |

Position conversion: `three.set(mj.x, mj.z, -mj.y)`

Quaternion conversion: `three.set(mj.x, mj.z, -mj.y, mj.w)`

---

## Adding New Scenes

1. Create your `.xml` scene file following MuJoCo XML format
2. Place it in `assets/scenes/`
3. Add any mesh files (STL/OBJ) to `assets/scenes/meshes/`
4. Update `assets/scenes/index.json` to include the new XML filename
5. Reference meshes in the XML:
```xml
<asset>
    <mesh name="my_part" file="meshes/my_part.stl" />
</asset>
<worldbody>
    <body name="my_body" pos="0 0 0.5">
        <geom type="mesh" mesh="my_part" rgba="0.5 0.5 0.5 1" />
    </body>
</worldbody>
```

---

## Adding New Meshes

### Supported Formats
- **STL** (recommended) - Binary or ASCII stereolithography
- **OBJ** - Wavefront OBJ with optional MTL materials

### Mesh Placement
1. Place mesh files in `assets/scenes/meshes/`
2. Reference in XML with `<mesh>` element:
```xml
<mesh name="part_name" file="meshes/part_name.stl" />
```
3. Use in body geometry:
```xml
<geom type="mesh" mesh="part_name" rgba="0.7 0.7 0.7 1" />
```

### Mesh Bundle Regeneration
If you add new meshes, the mesh bundle will need to be regenerated for optimal loading. Individual mesh files will still be loaded correctly as a fallback.

---

## Customization Guide

### Visual Appearance

**Ground plane** (in scene XML):
```xml
<texture type="2d" name="groundplane" builtin="checker"
    rgb1="0.2 0.3 0.4" rgb2="0.1 0.2 0.3" width="300" height="300"/>
<material name="groundplane" texture="groundplane" texrepeat="5 5"/>
```

**Sky gradient** (in scene XML):
```xml
<texture type="skybox" builtin="gradient" rgb1="0.3 0.5 0.7" rgb2="0 0 0"
    width="512" height="3072"/>
```

**Body colors**: Each `<geom>` has an `rgba` attribute (red, green, blue, alpha, 0-1 range)

### Camera
```javascript
// In src/main.js constructor
this.camera.position.set(2.0, 1.7, 1.7);  // Adjust viewing angle
```

### Lighting
```javascript
// In src/main.js
this.ambientLight = new THREE.AmbientLight(0xffffff, 0.1 * 3.14);
// Add directional lights in the XML <light> elements
```

---

## Troubleshooting

### Robot Collapses or Explodes
- Ensure actuator `ctrl` values are within `ctrlrange` bounds
- Check that joints aren't fighting each other
- If using free joint, the model needs a balance controller

### Joints Don't Move
- Set `this.data.ctrl[i]`, not `this.data.qpos[i]`
- Code must be inside `if (!this.params["paused"])` block
- Code must be before `mujoco.mj_step()`
- Check you're addressing the correct scene: `if (this.params["scene"] === "your_scene.xml")`

### Black Screen / Meshes Not Loading
- Check browser console for network errors
- Verify `assets/scenes/index.json` has correct file list
- Ensure mesh files exist (either as bundle or individual files)
- Check CORS if serving from different domain

### Jerky Animation
- Use `this.data.time` for timing (not frame-dependent)
- Apply easing: `(1 - Math.cos(phase * Math.PI)) / 2`
- Avoid sudden large changes to ctrl values

---

## Performance Optimization

### Mesh Bundle
- Reduces HTTP requests from 50+ to 1
- Compresses binary meshes ~60% via gzip
- Uses native `DecompressionStream` (no library needed)

### Rendering
- Use `contype="0" conaffinity="0"` for visual-only geometry (no collision cost)
- Minimize shadow-casting lights
- Reduce mesh polygon counts for faster rendering

### Memory
- MuJoCo WASM module: ~30MB
- Mesh data: varies by model complexity
- Three.js scene: 10-20MB typical
