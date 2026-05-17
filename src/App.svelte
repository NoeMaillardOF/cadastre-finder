<script>
  import Map from "./Map.svelte";
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
    } else if (geometry.type === "MultiPolygon" && geometry.coordinates[0] && geometry.coordinates[0][0]) {
      const coords = geometry.coordinates[0][0];
      let lonSum = 0, latSum = 0;
      coords.forEach(coord => {
        lonSum += coord[0];
        latSum += coord[1];
      });
      return {
        lon: lonSum / coords.length,
        lat: latSum / coords.length
      };
    }
    return null;
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

      features = features.filter((feature) => {
        const area = feature.properties?.surface_parcelle || 0;
        return area >= minThumb && area <= maxThumb;
      });

      features.sort((a, b) => {
        const areaA = a.properties?.surface_parcelle || 0;
        const areaB = b.properties?.surface_parcelle || 0;
        return areaB - areaA;
      });

      const parcelsToProcess = features.slice(0, 20);
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

      results = [...parcelsWithAddresses, ...features.slice(20)];
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
    <Map
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
