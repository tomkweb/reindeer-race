# Reindeer Race

A real-time 3D reindeer race simulation that runs in the browser. Nine reindeer
gallop down a snowy track with broadcast-style camera work (aerial, chase, rail,
finish line). Each deer randomly takes off and floats back down mid-race. All
finishers land on the track.

## Running it

The model files are loaded over HTTP, so a static file server is required —
opening `index.html` directly with `file://` will not work.

From the project folder:

```
python -m http.server 8080
```

Then open `http://localhost:8080` in a browser.

## Controls

- **Start Race** — begins the countdown and the race
- **Auto / Aerial / Chase / Rail / Finish** — manual camera override

## Files

- `index.html` — the race app (Three.js, single file)
- `scene.gltf` / `scene.bin` — the deer model and skeleton

## Credits

The deer model is **"Deer Animation" by DarryleH**, available on Sketchfab under
**CC-BY-4.0**. Attribution is required wherever this project is used.

Source: https://sketchfab.com/3d-models/deer-animation-347e5d4fbee14e53811c5df2221be8d0
