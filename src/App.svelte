<script>
  import { onMount, tick } from "svelte";
  import OpenSeadragon from "openseadragon";
  import L from "leaflet";
  import "leaflet/dist/leaflet.css";
  import { GcpTransformer } from "@allmaps/transform";
  import { parseAnnotation } from "@allmaps/annotation";
  import { WarpedMapLayer } from "@allmaps/leaflet";

  let osdContainer;
  let leafletContainer;
  let viewer;
  let map;

  // State
  let annotationsData = [];
  let geoJsonFeatures = [];
  let hoveredId = null;
  let layerMap = {};
  let currentIndex = -1;
  let transformer;
  let warpedMapLayer;
  let mapOpacity = 0.1;
  let mapMetadata = [];

  $: if (currentIndex >= 0) {
    scrollToCurrent();
  }

  $: if (warpedMapLayer) {
    warpedMapLayer.setOpacity(mapOpacity);
  }

  const scrollToCurrent = async () => {
    await tick();
    const el = document.getElementById(`sidebar-anno-${currentIndex}`);
    if (el) {
      el.scrollIntoView({ behavior: "smooth", block: "nearest" });
    }
  };

  // --- Utility functions ---

  const getI18nText = (obj) => {
    if (!obj) return "";
    if (typeof obj === "string") return obj;
    const values = Object.values(obj)[0];
    return Array.isArray(values) ? values.join(", ") : values;
  };

  /** Ray-casting point-in-polygon test */
  const pointInPolygon = (px, py, coords) => {
    let inside = false;
    for (let i = 0, j = coords.length - 1; i < coords.length; j = i++) {
      const [xi, yi] = coords[i];
      const [xj, yj] = coords[j];
      if (
        yi > py !== yj > py &&
        px < ((xj - xi) * (py - yi)) / (yj - yi) + xi
      ) {
        inside = !inside;
      }
    }
    return inside;
  };

  /** Compute axis-aligned bounding box for a set of [x,y] coordinates */
  const getBBox = (coords) => {
    let minX = Infinity,
      minY = Infinity,
      maxX = -Infinity,
      maxY = -Infinity;
    coords.forEach(([x, y]) => {
      if (x < minX) minX = x;
      if (x > maxX) maxX = x;
      if (y < minY) minY = y;
      if (y > maxY) maxY = y;
    });
    return { minX, minY, maxX, maxY, width: maxX - minX, height: maxY - minY };
  };

  // --- Annotation navigation ---

  const selectAnnotation = (index) => {
    if (index < 0 || index >= annotationsData.length) return;
    currentIndex = index;
    const id = annotationsData[index].id;
    centerLeaflet(id);
    centerOsd(id);
    resetLeaflet();
    highlightLeaflet(id);
    resetOsd();
    highlightOsd(id);
  };

  const prevAnnotation = () => {
    if (annotationsData.length === 0) return;
    let newIndex = currentIndex - 1;
    if (newIndex < 0) newIndex = annotationsData.length - 1;
    selectAnnotation(newIndex);
  };

  const nextAnnotation = () => {
    if (annotationsData.length === 0) return;
    let newIndex = currentIndex + 1;
    if (newIndex >= annotationsData.length) newIndex = 0;
    selectAnnotation(newIndex);
  };

  const handleKeydown = (e) => {
    if (
      document.activeElement.tagName === "INPUT" ||
      document.activeElement.tagName === "TEXTAREA"
    )
      return;
    if (e.key === "ArrowLeft") prevAnnotation();
    if (e.key === "ArrowRight") nextAnnotation();
  };

  const updateCurrentIndex = (id) => {
    const idx = annotationsData.findIndex((a) => a.id === id);
    if (idx !== -1) currentIndex = idx;
  };

  // --- Interaction helpers ---

  const highlightLeaflet = (id) => {
    if (layerMap[id]) layerMap[id].setStyle({ fillOpacity: 0.4 });
  };

  const resetLeaflet = () => {
    Object.values(layerMap).forEach((l) => l.setStyle({ fillOpacity: 0.2 }));
  };

  const centerLeaflet = (id) => {
    if (!layerMap[id]) return;
    const bounds = layerMap[id].getBounds();
    map.fitBounds(bounds, { padding: [20, 20], maxZoom: 17 });
    const anno = annotationsData.find((a) => a.id === id);
    if (anno?.popupHtml) {
      L.popup()
        .setLatLng(bounds.getCenter())
        .setContent(anno.popupHtml)
        .openOn(map);
    }
  };

  const highlightOsd = (id) => {
    const poly = document.querySelector(`#osd-overlay-${id} .osd-polygon`);
    if (poly) poly.setAttribute("fill", "rgba(239,68,68,0.4)");
  };

  const resetOsd = () => {
    document.querySelectorAll(".osd-polygon").forEach((poly) => {
      poly.setAttribute("fill", "rgba(239,68,68,0.2)");
    });
  };

  const centerOsd = (id) => {
    const anno = annotationsData.find((a) => a.id === id);
    if (!anno || !viewer) return;
    const { minX, minY, width, height } = getBBox(anno.coords);
    const rect = viewer.viewport.imageToViewportRectangle(
      minX,
      minY,
      width,
      height,
    );
    // Add padding around the annotation
    rect.x -= rect.width * 0.5;
    rect.y -= rect.height * 0.5;
    rect.width *= 2;
    rect.height *= 2;
    viewer.viewport.fitBounds(rect);
  };

  // --- Initialization ---

  onMount(async () => {
    // 1. Fetch manifest and Allmaps georeferencing data
    const [manifestResp, allmapsResp] = await Promise.all([
      fetch("./manifest.json"),
      fetch("https://annotations.allmaps.org/maps/435ccc0ea633df34"),
    ]);
    const manifestData = await manifestResp.json();
    const allmapsData = await allmapsResp.json();
    const parsedAllmaps = parseAnnotation(allmapsData);

    mapMetadata = manifestData.metadata || [];

    transformer = new GcpTransformer(parsedAllmaps[0].gcps);

    const imageServiceInfoUrl =
      manifestData.items[0].items[0].items[0].body.service[0]["@id"] +
      "/info.json";

    // 2. Parse annotations into coordinate arrays
    const rawAnnotations = manifestData.items[0].annotations?.[0]?.items || [];
    annotationsData = rawAnnotations
      .map((anno, idx) => {
        let coords = [];
        const selectorType = anno.target?.selector?.type;
        const selectorValue = anno.target?.selector?.value;

        if (selectorType === "SvgSelector" && selectorValue) {
          const doc = new DOMParser().parseFromString(
            selectorValue,
            "image/svg+xml",
          );
          const poly = doc.querySelector("polygon");
          if (poly) {
            coords = poly
              .getAttribute("points")
              .trim()
              .split(/\s+/)
              .map((p) => {
                const [x, y] = p.split(",").map(Number);
                return [x, y];
              });
          }
        } else if (selectorType === "FragmentSelector" && selectorValue) {
          const match = selectorValue.match(
            /xywh=(?:pixel:)?([0-9.]+),([0-9.]+),([0-9.]+),([0-9.]+)/,
          );
          if (match) {
            const [x, y, w, h] = match.slice(1).map(parseFloat);
            coords = [
              [x, y],
              [x + w, y],
              [x + w, y + h],
              [x, y + h],
            ];
          }
        } else if (
          typeof anno.target === "string" &&
          anno.target.includes("#xywh=")
        ) {
          const [x, y, w, h] = anno.target
            .split("#xywh=")[1]
            .split(",")
            .map(Number);
          coords = [
            [x, y],
            [x + w, y],
            [x + w, y + h],
            [x, y + h],
          ];
        }

        // Parse annotation body for label, snippet, and popup content
        let label = "Annotation";
        let popupHtml = "Annotation";
        let snippetHtml = "";
        const bodyValue = anno.body?.[0]?.value;
        if (bodyValue) {
          const doc = new DOMParser().parseFromString(bodyValue, "text/html");
          label = doc.body.textContent || "Annotation";
          popupHtml = bodyValue; // Preserve rich HTML for Leaflet popup
          snippetHtml = bodyValue;
        }

        return { ...anno, coords, id: idx, label, popupHtml, snippetHtml };
      })
      .filter((a) => a.coords?.length > 0);

    // 3. Setup OpenSeadragon (left panel)
    viewer = OpenSeadragon({
      element: osdContainer,
      prefixUrl: "https://openseadragon.github.io/openseadragon/images/",
      tileSources: [imageServiceInfoUrl],
      crossOriginPolicy: "Anonymous",
    });

    // Click: select annotation if hit, otherwise pan/zoom Leaflet to that geo position
    viewer.addHandler("canvas-click", (event) => {
      if (!event.quick || !transformer || !map) return;
      const viewportPoint = viewer.viewport.pointFromPixel(event.position);
      const imagePoint =
        viewer.viewport.viewportToImageCoordinates(viewportPoint);
      const px = imagePoint.x,
        py = imagePoint.y;

      const hitAnno = annotationsData.find((a) =>
        pointInPolygon(px, py, a.coords),
      );

      if (hitAnno) {
        updateCurrentIndex(hitAnno.id);
        resetLeaflet();
        highlightLeaflet(hitAnno.id);
        resetOsd();
        highlightOsd(hitAnno.id);
        centerLeaflet(hitAnno.id);
        centerOsd(hitAnno.id);
      } else {
        const [lon, lat] = transformer.transformToGeo([px, py]);
        map.flyTo([lat, lon], 17, { animate: true, duration: 0.8 });
      }
    });

    // Hover: highlight annotation under cursor on both maps
    osdContainer.addEventListener("mousemove", (e) => {
      if (!viewer?.viewport) return;
      const rect = osdContainer.getBoundingClientRect();
      const pixel = new OpenSeadragon.Point(
        e.clientX - rect.left,
        e.clientY - rect.top,
      );
      const viewportPoint = viewer.viewport.pointFromPixel(pixel);
      const imagePoint =
        viewer.viewport.viewportToImageCoordinates(viewportPoint);

      const newHoveredId =
        annotationsData.find((a) =>
          pointInPolygon(imagePoint.x, imagePoint.y, a.coords),
        )?.id ?? null;

      if (newHoveredId !== hoveredId) {
        resetOsd();
        resetLeaflet();
        hoveredId = newHoveredId;
        if (hoveredId !== null) {
          highlightOsd(hoveredId);
          highlightLeaflet(hoveredId);
        }
      }
      osdContainer.style.cursor = hoveredId !== null ? "pointer" : "";
    });

    osdContainer.addEventListener("mouseleave", () => {
      if (hoveredId !== null) {
        resetOsd();
        resetLeaflet();
        hoveredId = null;
      }
      osdContainer.style.cursor = "";
    });

    // Draw SVG polygon overlays once the image is loaded
    viewer.addHandler("open", () => {
      const svgNS = "http://www.w3.org/2000/svg";

      annotationsData.forEach((anno) => {
        const { minX, minY, width, height } = getBBox(anno.coords);
        const strokeW = Math.max(width, height) * 0.015;
        const rect = viewer.viewport.imageToViewportRectangle(
          minX,
          minY,
          width,
          height,
        );

        const svg = document.createElementNS(svgNS, "svg");
        svg.setAttribute("viewBox", `0 0 ${width} ${height}`);
        svg.setAttribute("preserveAspectRatio", "none");
        svg.style.width = "100%";
        svg.style.height = "100%";
        svg.style.overflow = "visible";
        svg.style.pointerEvents = "none";
        svg.id = `osd-overlay-${anno.id}`;
        svg.classList.add("osd-overlay");

        const polygon = document.createElementNS(svgNS, "polygon");
        polygon.setAttribute(
          "points",
          anno.coords.map(([x, y]) => `${x - minX},${y - minY}`).join(" "),
        );
        polygon.setAttribute("fill", "rgba(239,68,68,0.2)");
        polygon.setAttribute("stroke", "rgba(239,68,68,1)");
        polygon.setAttribute("stroke-width", String(strokeW));
        polygon.setAttribute("stroke-linejoin", "round");
        polygon.classList.add("osd-polygon");
        svg.appendChild(polygon);

        viewer.addOverlay({ element: svg, location: rect });
      });
    });

    // 4. Setup Leaflet (right panel)
    map = L.map(leafletContainer).setView([52.37, 4.9], 13);
    L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png", {
      maxZoom: 19,
      attribution: "&copy; OpenStreetMap contributors",
    }).addTo(map);

    // Add georeferenced warped map
    warpedMapLayer = new WarpedMapLayer(allmapsData).addTo(map);
    warpedMapLayer.setOpacity(mapOpacity);

    // Transform image coordinates to geographic coordinates
    geoJsonFeatures = annotationsData.map((anno) => {
      let geoCoords = anno.coords.map((coord) =>
        transformer.transformToGeo(coord),
      );
      // Close the polygon ring if needed
      if (
        geoCoords.length > 0 &&
        (geoCoords[0][0] !== geoCoords[geoCoords.length - 1][0] ||
          geoCoords[0][1] !== geoCoords[geoCoords.length - 1][1])
      ) {
        geoCoords.push([...geoCoords[0]]);
      }
      return {
        type: "Feature",
        properties: {
          id: anno.id,
          label: anno.label,
          popupHtml: anno.popupHtml,
        },
        geometry: { type: "Polygon", coordinates: [geoCoords] },
      };
    });

    // Draw GeoJSON layer with interactive polygons
    const geoLayer = L.geoJSON(
      { type: "FeatureCollection", features: geoJsonFeatures },
      {
        style: {
          color: "#ef4444",
          weight: 2,
          fillColor: "#ef4444",
          fillOpacity: 0.2,
        },
        onEachFeature: (feature, layer) => {
          layerMap[feature.properties.id] = layer;

          if (feature.properties?.popupHtml) {
            layer.bindPopup(feature.properties.popupHtml);
          }

          layer.on("mouseover", () => {
            hoveredId = feature.properties.id;
            layer.setStyle({ fillOpacity: 0.4 });
            highlightOsd(feature.properties.id);
          });
          layer.on("mouseout", () => {
            hoveredId = null;
            layer.setStyle({ fillOpacity: 0.2 });
            resetOsd();
          });
          layer.on("click", () => {
            updateCurrentIndex(feature.properties.id);
            centerOsd(feature.properties.id);
            centerLeaflet(feature.properties.id);
          });
        },
      },
    ).addTo(map);

    if (geoJsonFeatures.length > 0) {
      map.fitBounds(geoLayer.getBounds());
    }
  });
</script>

<svelte:window on:keydown={handleKeydown} />

<main
  class="w-screen h-screen flex flex-col bg-base-200 text-base-content font-sans"
  data-theme="light"
>
  <!-- Navbar -->
  <div class="navbar bg-base-100 shadow-sm px-4">
    <div class="navbar-start gap-2">
      <img src="./atm_logo.png" alt="ATM Logo" class="h-8" />
      <h1 class="text-xl font-bold">Geo Annotation Demo</h1>
    </div>
    <div class="navbar-center hidden md:flex">
      <p class="text-sm opacity-70">
        Left: Original map view (IIIF) · Right: Georeferenced via Allmaps ·
        Click to sync views
      </p>
    </div>
    <div class="navbar-end gap-2">
      <button
        class="btn btn-sm btn-ghost"
        on:click={() => document.getElementById("colofon_modal").showModal()}
        >About</button
      >
    </div>
  </div>

  <!-- Map panels -->
  <div
    class="flex flex-col md:flex-row flex-1 gap-4 p-4 min-h-0 w-full overflow-y-auto md:overflow-visible"
  >
    <div
      class="card bg-base-100 shadow-sm flex-1 min-h-[50vh] md:min-h-0 shrink-0 md:shrink"
    >
      <div class="card-body p-3 flex flex-col min-h-0">
        <h2 class="card-title text-sm">Image view (Original IIIF)</h2>
        <div
          bind:this={osdContainer}
          class="flex-1 rounded-lg overflow-hidden relative"
        ></div>
      </div>
    </div>

    <div
      class="card bg-base-100 shadow-sm flex-1 min-h-[50vh] md:min-h-0 shrink-0 md:shrink"
    >
      <div class="card-body p-3 flex flex-col min-h-0">
        <div class="flex items-center justify-between">
          <h2 class="card-title text-sm">Map view (Allmaps Transformed)</h2>
          <div class="flex items-center gap-2">
            <span class="text-xs opacity-70">Map Opacity:</span>
            <input
              type="range"
              min="0"
              max="1"
              step="0.01"
              bind:value={mapOpacity}
              class="range range-xs range-primary w-24"
            />
          </div>
        </div>
        <div
          bind:this={leafletContainer}
          class="flex-1 rounded-lg overflow-hidden relative z-0 mt-2"
        ></div>
      </div>
    </div>

    <!-- Sidebar -->
    <div
      class="card bg-base-100 shadow-sm w-full md:w-72 lg:w-80 shrink-0 min-h-[50vh] md:min-h-0 flex flex-col overflow-hidden"
    >
      <div
        class="p-3 border-b border-base-200 bg-base-100 z-10 sticky top-0 flex items-center justify-between"
      >
        <h2 class="card-title text-sm">Annotations</h2>
        {#if annotationsData.length > 0}
          <div class="join">
            <button on:click={prevAnnotation} class="btn btn-xs join-item"
              >←</button
            >
            <span
              class="btn btn-xs btn-ghost join-item no-animation pointer-events-none tabular-nums px-1"
            >
              {currentIndex >= 0
                ? currentIndex + 1
                : "–"}/{annotationsData.length}
            </span>
            <button on:click={nextAnnotation} class="btn btn-xs join-item"
              >→</button
            >
          </div>
        {/if}
      </div>
      <div class="flex-1 overflow-y-auto p-2 space-y-2 relative">
        {#if annotationsData.length === 0}
          <div class="text-sm opacity-50 p-2 text-center mt-4">
            Loading annotations...
          </div>
        {/if}
        {#each annotationsData as anno (anno.id)}
          <!-- svelte-ignore a11y-click-events-have-key-events -->
          <!-- svelte-ignore a11y-no-static-element-interactions -->
          <div
            id="sidebar-anno-{anno.id}"
            class="block w-full text-left p-3 rounded-lg border transition-all cursor-pointer {currentIndex ===
            anno.id
              ? 'bg-primary text-primary-content border-primary shadow-md'
              : 'border-base-200 hover:bg-base-200'}"
            on:click={() => selectAnnotation(anno.id)}
          >
            <div
              class="text-sm line-clamp-3 overflow-hidden text-ellipsis mb-1 annotation-snippet font-serif"
            >
              {#if anno.snippetHtml}
                {@html anno.snippetHtml}
              {:else}
                <span class="font-semibold"
                  >{anno.label || "Annotation " + (anno.id + 1)}</span
                >
              {/if}
            </div>
            {#if anno.motivation}
              <div class="text-xs opacity-70 mt-1 capitalize font-sans">
                {anno.motivation}
              </div>
            {/if}
            <div class="text-xs opacity-70 mt-1 font-sans">ID: {anno.id}</div>
          </div>
        {/each}
      </div>
    </div>
  </div>

  <dialog id="colofon_modal" class="modal">
    <div class="modal-box w-11/12 max-w-2xl max-h-[90vh] flex flex-col">
      <h3 class="font-bold text-lg mb-4">Colofon</h3>

      <div class="overflow-y-auto flex-1 space-y-4 text-sm">
        <p>
          The annotations for this map were made in <strong
            ><a
              href="https://liiive.now/"
              class="link link-primary"
              target="_blank"
              rel="noopener noreferrer">liiive.now</a
            ></strong
          > and then exported.
        </p>
        <p>
          We used <a
            href="https://allmaps.org/"
            class="link link-primary"
            target="_blank"
            rel="noopener noreferrer">Allmaps</a
          >
          to georeference the historical map. You can view the georeferenced version
          in the
          <a
            href="https://viewer.allmaps.org/?url=https://amsterdamtimemachine.github.io/berckenrode-iiif/manifest.json"
            class="link link-primary"
            target="_blank"
            rel="noopener noreferrer">Allmaps viewer</a
          >.
        </p>

        <h4 class="font-semibold text-md mt-4 border-b border-base-300 pb-1">
          Map Metadata
        </h4>
        {#if mapMetadata.length > 0}
          <table class="table table-sm">
            <tbody>
              {#each mapMetadata as item}
                <tr>
                  <th class="w-1/3 align-top whitespace-normal">
                    {getI18nText(item.label)}
                  </th>
                  <td class="align-top whitespace-normal">
                    {getI18nText(item.value)}
                  </td>
                </tr>
              {/each}
            </tbody>
          </table>
        {:else}
          <p class="opacity-70">No metadata available in the manifest.</p>
        {/if}
      </div>

      <div class="modal-action">
        <form method="dialog">
          <button class="btn">Close</button>
        </form>
      </div>
    </div>
    <form method="dialog" class="modal-backdrop">
      <button>close</button>
    </form>
  </dialog>
</main>
