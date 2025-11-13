# CLAUDE.md - Piano Fluid Interactive Art Project

## Project Overview

**Piano Fluid** is an interactive WebGL-based fluid simulation artwork that responds to piano key presses. Each piano key triggers a unique color palette and fluid animation pattern, creating living visual art through the CABLES.gl framework.

### Key Technologies
- **CABLES.gl**: Visual programming framework for WebGL (patch-based system)
- **WebGL**: Hardware-accelerated fluid dynamics simulation
- **Vanilla JavaScript**: Event handling and animation control
- **HTML5 Canvas**: Rendering surface

### Project Purpose
This is an experimental art project that creates dynamic, organic fluid animations triggered by piano input. Each key press spawns fluid with specific color palettes, creating an ever-evolving visual composition.

---

## Repository Structure

```
/home/user/fluden/
├── index.html                          # Main production file (PRIMARY FILE)
├── js/
│   └── patch.js                        # CABLES.gl compiled patch (1.9MB, DO NOT EDIT)
├── fluden/                             # Original CABLES.gl export
│   ├── index.html                      # Original CABLES export version
│   ├── doc.md                          # Patch variable documentation
│   ├── js/patch.js                     # Patch backup
│   ├── screenshot.png                  # Project screenshot
│   ├── cables.txt                      # CABLES ASCII art
│   ├── LICENCE                         # License info
│   ├── credits.txt                     # Credits
│   └── legal.txt                       # Legal info
├── cables.txt                          # CABLES branding ASCII art
├── [Multiple timestamped backups]      # Development snapshots with Turkish names
└── CLAUDE.md                           # This file
```

### File Naming Convention

The repository contains many backup files with **Turkish names and timestamps**:
- Pattern: `[description] [time]` or `[description]`
- Examples: `"en iyisi 02:50"` (the best 02:50), `"mükemmel 03:29"` (perfect 03:29)
- These represent iterative development snapshots
- **DO NOT DELETE** these backups without explicit user permission
- When making significant changes, consider creating a similar timestamped backup

---

## Core Architecture

### 1. Piano Key to Color Mapping

Located in `index.html` lines 78-95, the `PIANO_COLORS` object maps piano keys (C3-D5) to color palettes:

```javascript
const PIANO_COLORS = {
    "c3": [{r: 253, g: 1, b: 0}, ...],  // 10 colors per key
    "d3": [{r: 215, g: 228, b: 196}, ...],
    // ... 15 total keys mapped
}
```

**Key Details:**
- **15 piano keys** mapped (C3, D3, E3, F3, G3, A3, B3, C4, D4, E4, F4, G4, A4, B4, C5, D5)
- **10 RGB colors** per key
- Colors are in 0-255 range, converted to 0-1 for WebGL
- Each key has a carefully curated palette

### 2. Fluid Session Management

The system manages fluid animations through "sessions":

**Session Rules** (lines 107-110):
- Maximum **7 keys per session** (`MAX_KEYS_PER_SESSION = 7`)
- First key press: Creates NEW session with new starting position
- Keys 2-7: Add new paths to EXISTING session
- Key 8+: Ends current session, starts NEW session

**Session Object Structure** (lines 347-376):
```javascript
session = {
    isActive: true,           // Session running state
    shouldEnd: false,         // Signal to end session
    currentX, currentY,       // Current fluid position
    frameIndex: 0,            // Animation frame counter
    paths: [],                // Array of path objects
    addPath: function(...)    // Add new path to session
}
```

### 3. Flow Patterns

Seven distinct fluid motion patterns (lines 235-243):

| Pattern | Parameters | Behavior |
|---------|-----------|----------|
| `circle` | radius: 100-120px, clockwise: bool | Circular motion with "breathing" |
| `spiral` | startRadius: 40-110px, clockwise: bool | Expanding/contracting spiral |
| `s_curve` | amplitude: 70px, length: 240px | Sinusoidal S-shaped path |
| `wave` | amplitude: 60px, wavelength: 180px | Wave-like motion |

Each path includes Perlin-like noise for organic movement (lines 502-506).

### 4. Smart Positioning System

**Purpose**: Prevent fluid overlaps and distribute animations across the canvas

**Implementation** (lines 156-200):
- Maintains memory of last 5 fluid start positions (`POSITION_MEMORY_SIZE = 5`)
- Minimum distance between fluids: **180px** (`MIN_DISTANCE_BETWEEN_FLUIDS`)
- Uses rejection sampling: up to 20 attempts to find suitable position
- Margins: 80px from screen edges

### 5. Auto-Clear System

**Purpose**: Clean canvas when inactive to prevent performance degradation

**Mechanism** (lines 144-154):
- Checks every **5 seconds**
- Clears canvas if:
  - No active fluids (`activeFluidCount === 0`)
  - 5+ seconds since last activity
- Uses WebGL `gl.clear()` for proper cleanup

---

## Animation System

### Frame-by-Frame Animation

**Constants** (line 335-336):
- `TOTAL_FRAMES = 120`: Path length
- `FRAME_DURATION = 15ms`: ~67 FPS target

### Animation Lifecycle

1. **Initialization** (lines 387-399):
   - Set initial color (4x rapid calls for stability)
   - Dispatch `mousedown` event to CABLES patch
   - Wait 50ms before starting movement

2. **Animation Loop** (lines 401-486):
   - Updates every 15ms
   - Finds active path from session
   - Calculates color blend based on progress
   - Dispatches `mousemove` events to CABLES
   - Removes completed paths from session

3. **Termination** (lines 430-451):
   - Triggers when all paths complete
   - Dispatches `mouseup` event
   - Decrements `activeFluidCount`
   - Marks session as inactive

### Color Blending

**Algorithm** (lines 455-471):
```
progress = currentFrame / totalFrames
colorProgress = progress × (numColors - 1)
colorIndex = floor(colorProgress)
colorBlend = fractional part
finalColor = lerp(color[index], color[index+1], colorBlend)
```

This creates smooth color transitions through the entire palette.

---

## CABLES.gl Integration

### What is CABLES.gl?

CABLES is a **visual programming tool** for creating interactive WebGL content. It uses a node-based "patch" system (similar to Max/MSP or TouchDesigner).

### Key Integration Points

**File**: `/js/patch.js` (1.9MB compiled patch)
- **DO NOT EDIT** this file directly
- To modify the patch: Edit in CABLES.gl editor, re-export
- Contains all WebGL shaders, fluid physics, rendering logic

**Variables** (documented in `fluden/doc.md`):
- `SplatColor`: Current fluid color [r, g, b] (0-1 range)
- `MousePosX/Y`: Cursor position
- `MouseSplatForce`: Fluid injection strength
- `FluidVelocityDiffusion`: Viscosity control
- `FluidVorticity`: Curl/swirl intensity
- `FluidPressure`: Divergence solver pressure

**Patch Initialization** (lines 554-565):
```javascript
CABLES.patch = new CABLES.Patch({
    patch: CABLES.exportedPatch,
    glCanvasId: "glcanvas",
    glCanvasResizeToWindow: true,
    // ...
});
```

---

## Event System

### Message Events

The application listens for `window.postMessage` events:

**Event Types** (lines 245-271):

1. **`pianoKeyPress`**:
   ```javascript
   window.postMessage({
       type: 'pianoKeyPress',
       key: 'c4'  // Piano key name (lowercase)
   }, '*');
   ```
   - Triggers normal fluid animation
   - Used for actual piano input

2. **`ghostKeyPress`**:
   ```javascript
   window.postMessage({
       type: 'ghostKeyPress',
       key: 'c4'
   }, '*');
   ```
   - Same as pianoKeyPress but logged differently
   - Used for automated/programmatic input

3. **`parentResize`**:
   ```javascript
   window.postMessage({
       type: 'parentResize',
       width: 1920,
       height: 1080
   }, '*');
   ```
   - Updates canvas dimensions when embedded in iframe

### Canvas Events

The system dispatches synthetic mouse events to the CABLES canvas:

- `mousedown`: Start fluid injection
- `mousemove`: Continue fluid path
- `mouseup`: End fluid injection

---

## Development Workflows

### Making Changes to the Fluid Simulation

**Option 1: Edit index.html JavaScript (Recommended)**

For changes to:
- Color palettes
- Animation patterns
- Session management
- Event handling

Simply edit the `<script>` section in `index.html`.

**Option 2: Edit CABLES Patch (Advanced)**

For changes to:
- Fluid physics parameters
- Rendering effects
- Shader code

1. Visit [cables.gl](https://cables.gl)
2. Import the patch from `/fluden/` directory
3. Make changes in the visual editor
4. Export and replace `/js/patch.js`

### Testing Workflow

```javascript
// Manual test via browser console:
window.postMessage({ type: 'pianoKeyPress', key: 'c4' }, '*');
window.postMessage({ type: 'pianoKeyPress', key: 'e4' }, '*');
window.postMessage({ type: 'pianoKeyPress', key: 'g4' }, '*');
```

### Creating Backups

When making significant changes, create a backup:

```bash
# Get current time in HH:MM format
timestamp=$(date +"%H:%M")

# Create descriptive backup
cp index.html "new feature $timestamp"
```

---

## Git Workflow

### Branch Strategy

- **Main branch**: Not explicitly set (appears to be GitHub Pages default)
- **Development**: Currently on `claude/claude-md-mhxu9eftfm84kkll-01DAvD7A5JDVUeQsk9WnAQ2W`

### Commit Patterns

Analysis of git history shows:
- Frequent small updates to `index.html`
- Pattern: "Update index.html" for iterations
- Occasional renames of backup files
- No use of feature branches historically

### Best Practices for AI Assistants

1. **Always create a backup** before significant changes:
   ```bash
   cp index.html "backup $(date +'%H:%M %d %b')"
   ```

2. **Commit frequently** with descriptive messages:
   ```bash
   git add index.html
   git commit -m "Add new spiral flow pattern variant"
   ```

3. **Test before pushing**:
   - Open `index.html` in browser
   - Test with console commands
   - Verify no console errors

4. **Push to correct branch**:
   ```bash
   git push -u origin claude/claude-md-mhxu9eftfm84kkll-01DAvD7A5JDVUeQsk9WnAQ2W
   ```

---

## Key Constants and Parameters

### Performance Tuning

| Constant | Value | Purpose | File Location |
|----------|-------|---------|---------------|
| `TOTAL_FRAMES` | 120 | Path length | index.html:335 |
| `FRAME_DURATION` | 15ms | Animation speed | index.html:336 |
| `MAX_KEYS_PER_SESSION` | 7 | Keys before new session | index.html:110 |
| `MIN_DISTANCE_BETWEEN_FLUIDS` | 180px | Spacing | index.html:104 |
| `POSITION_MEMORY_SIZE` | 5 | Position history | index.html:105 |

### Visual Parameters

| Parameter | Value | Effect |
|-----------|-------|--------|
| Canvas background | `#000000` | Black |
| Margin from edges | 80px | Padding |
| Auto-clear interval | 5000ms | Cleanup frequency |
| Noise scale | 0.025 | Path organicness |

---

## Common Tasks for AI Assistants

### Adding a New Piano Key

1. Add color palette to `PIANO_COLORS` object:
   ```javascript
   "c6": [
       {r: 255, g: 100, b: 50},
       // ... 10 colors total
   ]
   ```

2. Key must be lowercase, format: `[note][octave]`

### Modifying Flow Patterns

1. Edit `FLOW_PATTERNS` array (line 235)
2. Add new pattern type in `generateBasePath()` switch statement (line 508)
3. Implement pattern mathematics

### Adjusting Session Length

Change `MAX_KEYS_PER_SESSION` (line 110):
- Lower value: More separate fluid animations
- Higher value: Longer continuous animations

### Changing Colors Mid-Session

Colors are determined at path creation. To enable dynamic color changes:
1. Store color array in session
2. Update in animation loop
3. Call `setPianoColor()` with new values

### Performance Optimization

If experiencing slowdown:
1. Reduce `TOTAL_FRAMES` (shorter animations)
2. Increase `FRAME_DURATION` (slower frame rate)
3. Decrease `MAX_KEYS_PER_SESSION` (fewer simultaneous paths)
4. Reduce `autoClearInterval` time (more frequent cleanup)

---

## Important Conventions

### Code Style

- **Language**: Primarily Turkish comments and variable names in backups, English in production
- **Indentation**: 4 spaces (JavaScript sections)
- **Quotes**: Single quotes for JS, double for HTML attributes
- **Console logs**: Use emoji prefixes (🎨, 👻, 🎹, ✅, ⚠️, etc.)

### Emoji Console Log System

| Emoji | Meaning | Example |
|-------|---------|---------|
| 🎨 | Fluid creation | "🎨 Fluid başlıyor" |
| 🎹 | Piano key press | "🎹 C4 → 10 renk" |
| 👻 | Ghost key press | "👻 GHOST C4 → 10 renk" |
| ✅ | Success/completion | "✅ Fluid tamamlandı" |
| 🧹 | Canvas cleared | "🧹 Canvas temizlendi" |
| ⚠️ | Warning | "⚠️ Renk bulunamadı" |
| 📍 | Position info | "📍 Yeni pozisyon" |
| 🆕 | New session | "🆕 YENİ SESSION" |
| ➕ | Path added | "➕ SESSION'A YENİ YOL" |
| 🏁 | Path complete | "🏁 Yol tamamlandı" |

### Naming Conventions

**Files**:
- Production: `index.html`
- Backups: Turkish description + timestamp
- Keep original CABLES structure in `/fluden/`

**Functions**:
- camelCase: `forceFullScreenCanvas()`, `getSmartRandomPosition()`
- Descriptive names: Function purpose clear from name

**Variables**:
- camelCase for local: `currentFluidSession`, `activeFluidCount`
- UPPER_SNAKE_CASE for constants: `PIANO_COLORS`, `MAX_KEYS_PER_SESSION`

---

## Debugging Tips

### Common Issues

**1. Fluid not appearing**
- Check console for color palette warnings
- Verify key name is lowercase and in `PIANO_COLORS`
- Ensure canvas is properly initialized

**2. Canvas not fullscreen**
- Multiple functions enforce fullscreen (lines 112-126, 207-213, 573-574)
- Stats panels are aggressively hidden (lines 222-233)
- Check browser console for WebGL errors

**3. Colors not changing**
- Verify `CABLES.patch.setVariable()` is called
- Check color values are normalized (0-1 range)
- Ensure patch is fully loaded before setting colors

**4. Performance issues**
- Check `activeFluidCount` (should be low)
- Verify auto-clear is working (check console logs)
- Consider reducing `TOTAL_FRAMES` or `MAX_KEYS_PER_SESSION`

### Debug Commands

```javascript
// Check current session state
console.log(currentFluidSession);

// Check active fluid count
console.log('Active fluids:', activeFluidCount);

// Force clear canvas
clearCanvas();

// Test specific key
window.postMessage({ type: 'pianoKeyPress', key: 'c4' }, '*');

// Check if CABLES is loaded
console.log(typeof CABLES !== 'undefined' ? 'Loaded' : 'Not loaded');
```

---

## Security Considerations

### User Input Validation

Currently, the system has minimal input validation:
- Key names are not sanitized
- No rate limiting on message events
- No origin checking on postMessage

**Recommendations for AI assistants**:
- Add origin validation if deploying publicly
- Implement rate limiting for key presses
- Validate key names against `PIANO_COLORS` keys

### XSS Prevention

- No user input is rendered to DOM
- Colors are numeric values only
- No `eval()` or `innerHTML` usage

---

## Browser Compatibility

### Requirements
- **WebGL 1.0** or **WebGL 2.0** support
- Modern JavaScript (ES6+)
- `postMessage` API support

### Tested Browsers
Based on code patterns, likely tested on:
- Chrome/Chromium (primary target)
- Safari (webkit prefixes present)
- Firefox (moz prefixes present)

### Mobile Support
- Touch events are handled (line 571)
- Responsive canvas sizing
- Viewport meta tag for mobile (line 58)

---

## Performance Characteristics

### Resource Usage

**Memory**:
- Base: ~2MB (patch.js loaded)
- Per session: ~10KB (path data)
- Canvas buffer: Depends on resolution

**CPU**:
- Animation loop: ~67 FPS target (15ms frame time)
- WebGL rendering: GPU-accelerated
- Auto-clear: Minimal impact (5s interval)

**GPU**:
- Fluid simulation: Shader-based (CABLES patch)
- Resolution-dependent performance
- Benefits from hardware acceleration

### Optimization Tips

1. **Reduce session complexity**: Lower `MAX_KEYS_PER_SESSION`
2. **Shorter animations**: Reduce `TOTAL_FRAMES`
3. **Slower frame rate**: Increase `FRAME_DURATION`
4. **Canvas resolution**: Modify canvas size (impacts quality vs performance)

---

## Future Enhancement Ideas

### Potential Features
- [ ] MIDI input support (direct piano keyboard integration)
- [ ] Save/export animations as video
- [ ] Custom color palette editor
- [ ] Recording and playback of key sequences
- [ ] Multiple simultaneous sessions
- [ ] Touch gesture support for mobile
- [ ] Audio visualization integration
- [ ] Preset pattern library

### Architectural Improvements
- [ ] Separate config file for constants
- [ ] TypeScript migration for type safety
- [ ] Module-based architecture
- [ ] Unit tests for animation logic
- [ ] Performance monitoring dashboard

---

## Troubleshooting for AI Assistants

### When Asked to Modify Fluid Physics

**Don't**: Edit `js/patch.js` directly
**Do**: Guide user to CABLES.gl editor or modify JavaScript parameters

### When Asked to Add Features

1. Check if feature requires CABLES patch changes
2. If yes: Explain CABLES workflow
3. If no: Implement in `index.html` script section

### When Asked to Debug

1. Check browser console first
2. Verify CABLES is loaded (`CABLES.jsLoaded` event)
3. Test with manual postMessage commands
4. Check canvas initialization

### When Asked to Deploy

**Current setup**: Appears to be GitHub Pages
- Main file: `index.html` in root
- No build process needed
- Direct deployment

**Deployment checklist**:
- [ ] Test all piano keys
- [ ] Verify fullscreen mode
- [ ] Check mobile responsiveness
- [ ] Test in multiple browsers
- [ ] Verify WebGL support detection

---

## Resources and References

### CABLES.gl Documentation
- Website: https://cables.gl
- Documentation: https://cables.gl/docs
- API Reference: https://cables.gl/api

### WebGL Fluid Simulation
- Based on Jos Stam's "Real-Time Fluid Dynamics for Games"
- GPU-based Navier-Stokes solver
- Grid-based velocity/pressure fields

### Color Theory
- 10 colors per key allow smooth gradients
- RGB color space (0-255) converted to normalized (0-1)
- Palettes appear manually curated for artistic effect

---

## Contact and Contribution

### Repository Information
- Platform: GitHub
- Current Branch: `claude/claude-md-mhxu9eftfm84kkll-01DAvD7A5JDVUeQsk9WnAQ2W`
- Language: JavaScript (vanilla), Turkish comments in backups

### For AI Assistants
- Respect existing code style and conventions
- Maintain backup file naming patterns
- Test thoroughly before committing
- Document significant changes
- Keep this CLAUDE.md updated with architectural changes

---

## Appendix A: Complete Piano Key Reference

| Key | Octave | Color Count | Example Colors |
|-----|--------|-------------|----------------|
| C3 | 3 | 10 | Reds, yellows, greens |
| D3 | 3 | 10 | Pastels, pinks |
| E3 | 3 | 10 | Magentas, cyans |
| F3 | 3 | 10 | Blues, aquas |
| G3 | 3 | 10 | Purples, greens |
| A3 | 3 | 10 | Blues, purples |
| B3 | 3 | 10 | Cyans, greens, magentas |
| C4 | 4 | 10 | Oranges, cyans |
| D4 | 4 | 10 | Greens, blues |
| E4 | 4 | 10 | Blues, pinks |
| F4 | 4 | 10 | Oranges, blues, purples |
| G4 | 4 | 10 | Cyans, blues, magentas |
| A4 | 4 | 10 | Pinks, whites, cyans |
| B4 | 4 | 10 | Reds, oranges |
| C5 | 5 | 10 | Greens, blues, yellows |
| D5 | 5 | 10 | Blacks, oranges, blues |

---

## Appendix B: File Size Reference

| File | Size | Purpose |
|------|------|---------|
| index.html | ~28KB | Main application |
| js/patch.js | 1.9MB | CABLES compiled patch |
| fluden/screenshot.png | ~240KB | Project screenshot |
| fluden/index.html | ~12KB | Original CABLES export |

---

*Last Updated: 2025-11-13*
*Document Version: 1.0*
*Created for AI assistant comprehension and development guidance*
