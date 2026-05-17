<script>
  import ParcelMap from "./ParcelMap.svelte";
  import SearchForm from "./SearchForm.svelte";
  import ParcelCarousel from "./ParcelCarousel.svelte";

  let inseeCode = $state("");
  let communeName = $state("");
  let minThumb = $state(0);
  let maxThumb = $state(8000);
  let results = $state([]);
  let loading = $state(false);
  let error = $state(null);
  let selectedParcel = $state(null);
  let communeSearchResults = $state([]);
  let showCommuneResults = $state(false);
  let selectedParcelIndex = $state(-1);

  const decompressGzip = async (compressedData) => {
    const ds = new DecompressionStream("gzip");
    const decompressedStream = new Response(compressedData).body.pipeThrough(ds);
    return new Response(decompressedStream).text();
  };

  const getAddressFromCoordinates = async (lon, lat) => {
    try {
      const response = await fetch(
        `https://api-adresse.data.gouv.fr/reverse/?lon=${lon}&lat=${lat}`
      );
      if (!response.ok) return null;

      const data = await response.json();
      if (data.features && data.features.length > 0) {
        return data.features[0].properties.label;
      }
      return null;
    } catch (err) {
      console.error("Error fetching address:", err);
      return null;
    }
  };

  const getCenterCoordinates = (geometry) => {
    if (!geometry) return null;

    if (geometry.type === "Point") {
      return { lon: geometry.coordinates[0], lat: geometry.coordinates[1] };
    } else if (geometry.type === "Polygon" && geometry.coordinates[0]) {
      const coords = geometry.coordinates[0];
      let lonSum = 0, latSum = 0;
      coords.forEach(coord => {
        lonSum += coord[0];
        latSum += coord[1];
      });
      return {
        lon: lonSum / coords.length,
        lat: latSum / coords.length
      };
    } else if (geometry.type === "MultiPolygon") {
      let lonSum = 0, latSum = 0, count = 0;
      geometry.coordinates.forEach(polygon => {
        if (polygon[0]) {
          polygon[0].forEach(coord => {
            lonSum += coord[0];
            latSum += coord[1];
            count++;
          });
        }
      });
      if (count === 0) return null;
      return { lon: lonSum / count, lat: latSum / count };
    }
    return null;
  };

  const getEdgeKey = (p1, p2) => {
    const precision = 6;
    const a = [p1[0].toFixed(precision), p1[1].toFixed(precision)];
    const b = [p2[0].toFixed(precision), p2[1].toFixed(precision)];
    const keyA = `${a[0]},${a[1]}`;
    const keyB = `${b[0]},${b[1]}`;
    return keyA < keyB ? `${keyA}|${keyB}` : `${keyB}|${keyA}`;
  };

  const extractEdges = (geometry) => {
    const edges = new Set();
    if (!geometry) return edges;

    const processRing = (ring) => {
      for (let i = 0; i < ring.length - 1; i++) {
        edges.add(getEdgeKey(ring[i], ring[i + 1]));
      }
    };

    if (geometry.type === 'Polygon') {
      geometry.coordinates.forEach(ring => processRing(ring));
    } else if (geometry.type === 'MultiPolygon') {
      geometry.coordinates.forEach(polygon => {
        polygon.forEach(ring => processRing(ring));
      });
    }

    return edges;
  };

  const buildAdjacencyGraph = (features) => {
    const edgeToParcels = new Map();
    const graph = new Map();

    features.forEach((feature, i) => {
      const edges = extractEdges(feature.geometry);
      edges.forEach(edgeKey => {
        if (!edgeToParcels.has(edgeKey)) edgeToParcels.set(edgeKey, new Set());
        edgeToParcels.get(edgeKey).add(i);
      });
      graph.set(i, new Set());
    });

    for (const indices of edgeToParcels.values()) {
      if (indices.size >= 2) {
        const arr = [...indices];
        for (let a = 0; a < arr.length; a++) {
          for (let b = a + 1; b < arr.length; b++) {
            graph.get(arr[a]).add(arr[b]);
            graph.get(arr[b]).add(arr[a]);
          }
        }
      }
    }

    return graph;
  };

  const findConnectedComponents = (graph) => {
    const visited = new Set();
    const components = [];

    for (const [node] of graph) {
      if (visited.has(node)) continue;
      const component = [];
      const stack = [node];
      while (stack.length > 0) {
        const current = stack.pop();
        if (visited.has(current)) continue;
        visited.add(current);
        component.push(current);
        for (const neighbor of graph.get(current)) {
          if (!visited.has(neighbor)) stack.push(neighbor);
        }
      }
      components.push(component);
    }

    return components;
  };

  const createCombinedParcel = (parcels, minThumb, maxThumb) => {
    const totalArea = parcels.reduce((sum, p) => sum + (p.properties?.surface_parcelle || 0), 0);
    if (totalArea < minThumb || totalArea > maxThumb) return null;

    const coordinates = [];
    for (const p of parcels) {
      if (p.geometry.type === 'Polygon') {
        coordinates.push(p.geometry.coordinates);
      } else if (p.geometry.type === 'MultiPolygon') {
        coordinates.push(...p.geometry.coordinates);
      }
    }

    const combinedId = parcels.map(p => p.id).join('+');
    const center = getCenterCoordinates({ type: 'MultiPolygon', coordinates });

    return {
      type: 'Feature',
      id: combinedId,
      properties: {
        isCombined: true,
        combinedIds: parcels.map(p => p.id),
        combinedParcelCount: parcels.length,
        surface_parcelle: totalArea,
        coordinates: center,
      },
      geometry: { type: 'MultiPolygon', coordinates }
    };
  };

  const searchProperties = async () => {
    if (!inseeCode) {
      error = "Please enter an INSEE code";
      return;
    }

    loading = true;
    error = null;

    try {
      const department = inseeCode.substring(0, inseeCode.length === 5 ? 2 : 3);
      const url = `https://cadastre.data.gouv.fr/data/etalab-cadastre/2025-09-01/geojson/communes/${department}/${inseeCode}/cadastre-${inseeCode}-parcelles.json.gz`;

      const response = await fetch(url, {
        headers: {
          "Accept-Encoding": "gzip",
        },
      });

      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }

      const compressedData = await response.arrayBuffer();
      const jsonString = await decompressGzip(compressedData);
      const data = JSON.parse(jsonString);

      let features = data.features.map((feature) => {
        if (!feature.id) {
          feature.id = `${inseeCode}-${Math.random().toString(36).substr(2, 9)}`;
        }

        if (feature.properties && feature.properties.contenance) {
          feature.properties.surface_parcelle = feature.properties.contenance;
        }

        return feature;
      });

      const MAX_COMBINATION_SIZE = 4;
      const combinableThreshold = Math.max(minThumb * 0.05, 30);

      let individualCandidates = [];
      let combinablePool = [];

      for (const feature of features) {
        const area = feature.properties?.surface_parcelle || 0;
        if (area >= minThumb && area <= maxThumb) {
          individualCandidates.push(feature);
        } else if (area < minThumb && area >= combinableThreshold) {
          combinablePool.push(feature);
        }
      }

      individualCandidates.sort((a, b) => {
        const areaA = a.properties?.surface_parcelle || 0;
        const areaB = b.properties?.surface_parcelle || 0;
        return areaB - areaA;
      });

      const graph = buildAdjacencyGraph(combinablePool);
      const components = findConnectedComponents(graph);
      const combinedFeatures = [];

      for (const component of components) {
        if (component.length < 2 || component.length > MAX_COMBINATION_SIZE) continue;
        const parcels = component.map(i => combinablePool[i]);
        const combined = createCombinedParcel(parcels, minThumb, maxThumb);
        if (combined) combinedFeatures.push(combined);
      }

      combinedFeatures.sort((a, b) => {
        const areaA = a.properties?.surface_parcelle || 0;
        const areaB = b.properties?.surface_parcelle || 0;
        return areaB - areaA;
      });

      const parcelsToProcess = individualCandidates.slice(0, 20);
      const parcelsWithAddresses = await Promise.all(
        parcelsToProcess.map(async (feature) => {
          const center = getCenterCoordinates(feature.geometry);
          if (center) {
            const address = await getAddressFromCoordinates(center.lon, center.lat);
            feature.properties.address = address;
            feature.properties.coordinates = center;
          }
          return feature;
        })
      );

      const combinedToProcess = combinedFeatures.slice(0, 20);
      const combinedWithAddresses = await Promise.all(
        combinedToProcess.map(async (feature) => {
          const center = getCenterCoordinates(feature.geometry);
          if (center) {
            const address = await getAddressFromCoordinates(center.lon, center.lat);
            feature.properties.address = address;
            feature.properties.coordinates = center;
          }
          return feature;
        })
      );

      results = [...parcelsWithAddresses, ...individualCandidates.slice(20), ...combinedWithAddresses, ...combinedFeatures.slice(20)];
    } catch (err) {
      console.error("Error fetching data:", err);
      error = "Error fetching data. Please check the INSEE code and try again.";
    } finally {
      loading = false;
    }
  };

  const searchCommune = async () => {
    if (!communeName || communeName.length < 2) {
      communeSearchResults = [];
      showCommuneResults = false;
      return;
    }

    try {
      const response = await fetch(
        `https://geo.api.gouv.fr/communes?nom=${encodeURIComponent(communeName)}&boost=population&limit=10`,
      );
      if (!response.ok) {
        throw new Error("Failed to fetch communes");
      }

      const data = await response.json();
      communeSearchResults = data.map((commune) => ({
        name: commune.nom,
        inseeCode: commune.code,
        postalCode: commune.codesPostaux?.[0] || "",
        population: commune.population?.toLocaleString() || "N/A",
      }));

      showCommuneResults = true;
    } catch (err) {
      console.error("Error searching for communes:", err);
      error = "Failed to search for communes. Please try again.";
      communeSearchResults = [];
      showCommuneResults = false;
    }
  };

  const selectCommune = (commune) => {
    inseeCode = commune.inseeCode;
    communeName = commune.name;
    communeSearchResults = [];
    showCommuneResults = false;
  };

  const selectParcel = (parcel, index) => {
    selectedParcel = parcel;
    selectedParcelIndex = index !== undefined ? index : results.findIndex(p => p.id === parcel.id);
  };

  $effect(() => {
    const onParcelSelect = (e) => selectParcel(e.detail);
    window.addEventListener("parcel-select", onParcelSelect);
    return () => {
      window.removeEventListener("parcel-select", onParcelSelect);
    };
  });
</script>

<main class="main-container">
  <div class="map-container">
    <ParcelMap
      {results}
      {selectedParcel}
    />
  </div>

  <SearchForm
    bind:inseeCode
    bind:communeName
    bind:minThumb
    bind:maxThumb
    bind:communeSearchResults
    bind:showCommuneResults
    {loading}
    {error}
    onsearch={searchProperties}
    onsearchcommune={searchCommune}
    onselectcommune={selectCommune}
  />

  <ParcelCarousel
    {results}
    bind:selectedParcel
    bind:selectedParcelIndex
    onselectparcel={selectParcel}
  />
</main>

<style>
  :global(body) {
    margin: 0;
    padding: 0;
    height: 100vh;
    width: 100vw;
    overflow: hidden;
  }

  :global(#app) {
    height: 100vh;
    width: 100vw;
    overflow: hidden;
  }

  :global(*:focus) {
    outline: 2px solid #3b82f6;
    outline-offset: 2px;
  }

  .main-container {
    position: relative;
    overflow: hidden;
    height: 100vh;
    display: flex;
    flex-direction: column;
  }

  .map-container {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    z-index: 1;
    width: 100%;
    height: 100%;
  }
</style>
