# Geo Annotation Demo

A web application that visualizes IIIF manifest annotations on both a high-resolution image viewer and a geographic map, with synchronized interaction between the two.

The left panel shows the original IIIF image using [OpenSeadragon](https://openseadragon.github.io/) with annotation polygons rendered as SVG overlays. The right panel displays the same annotations transformed to geographic coordinates on a [Leaflet](https://leafletjs.com/) map, using ground control points from [Allmaps](https://allmaps.org/).

## Features

- **Dual-panel view**: OpenSeadragon (IIIF image) and Leaflet (geographic map) side by side
- **Synchronized interaction**: clicking or hovering an annotation highlights it on both panels
- **Click-to-pan**: click anywhere on the image to fly to the corresponding location on the map
- **Keyboard navigation**: use ← / → arrow keys to step through annotations
- **SVG polygon overlays**: annotations render as accurate polygon shapes (not just bounding boxes)
- **Multiple selector formats**: supports `SvgSelector` (polygons), `FragmentSelector` (`xywh=pixel:...`), and plain `#xywh=` string targets

## Tech Stack

- [Svelte 5](https://svelte.dev/) + [Vite](https://vite.dev/)
- [Tailwind CSS 4](https://tailwindcss.com/) + [daisyUI 5](https://daisyui.com/)
- [OpenSeadragon](https://openseadragon.github.io/) — IIIF image viewer
- [Leaflet](https://leafletjs.com/) — interactive map
- [@allmaps/transform](https://allmaps.org/) — image-to-geo coordinate transformation via GCPs

## Setup

```bash
npm install
```

## Development

```bash
npm run dev
```

## Build

```bash
npm run build
```

Bundles everything into the `dist/` directory for static hosting. Uses relative paths (`base: './'`) so it works under any repository subpath (e.g. GitHub Pages).
