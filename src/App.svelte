<script>
  import { onMount } from "svelte";
  import { fade, slide } from "svelte/transition";
  import { quintOut } from "svelte/easing";
  import axios from "axios";
  import Map from "./Map.svelte";
  import DoubleSlider from "./lib/DoubleSlider.svelte";

  // State
  let inseeCode = "";
  let communeName = "";
  const MIN_SIZE = 0;
  const MAX_SIZE = 3000;
  let minThumb = MIN_SIZE;
  let maxThumb = MAX_SIZE;
  let results = [];
  let loading = false;
  let error = null;
  let selectedParcel = null;
  let communeSearchResults = [];
  let showCommuneResults = false;
  let isMobile = false;
  let currentSlide = 0;
  let carouselOpen = true;
  let searchFocused = false;

  // Check if mobile on mount and on resize
  function checkIfMobile() {
    isMobile = window.innerWidth <= 768;
  }

  function toggleCarousel() {
    carouselOpen = !carouselOpen;
  }

  onMount(() => {
    checkIfMobile();
    window.addEventListener("resize", checkIfMobile);
    return () => window.removeEventListener("resize", checkIfMobile);
  });

  function toggleSidebar() {
    sidebarOpen = !sidebarOpen;

    // Use requestAnimationFrame to ensure the DOM has updated
    requestAnimationFrame(() => {
      // Trigger map resize after a short delay
      setTimeout(() => {
        // Use type assertion to access window.leafletMap
        const leafletMap = window.leafletMap;
        if (leafletMap && typeof leafletMap.invalidateSize === "function") {
          leafletMap.invalidateSize(true);
        }
      }, 300); // Match this with your CSS transition duration
    });
  }

  // Function to decompress gzipped data
  const decompressGzip = async (compressedData) => {
    const ds = new DecompressionStream("gzip");
    const decompressedStream = new Response(compressedData).body.pipeThrough(
      ds,
    );
    return new Response(decompressedStream).text();
  };

  // Update sizes when DoubleSlider values change
  function updateSizes() {
    // Values are already in the correct range (0-3000) from the DoubleSlider
  }

  // Function to search for properties
  const searchProperties = async () => {
    if (!inseeCode) {
      error = "Please enter an INSEE code";
      return;
    }

    loading = true;
    error = null;

    try {
      // Format INSEE code to get the department (first 2-3 digits)
      const department = inseeCode.substring(0, inseeCode.length === 5 ? 2 : 3);
      const url = `https://cadastre.data.gouv.fr/data/etalab-cadastre/2025-09-01/geojson/communes/${department}/${inseeCode}/cadastre-${inseeCode}-parcelles.json.gz`;

      // Fetch the gzipped data
      const params = new URLSearchParams({
        insee: inseeCode,
        min_size: minThumb.toString(),
        max_size: maxThumb.toString(),
      });

      const response = await fetch(url, {
        headers: {
          "Accept-Encoding": "gzip",
        },
      });

      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }

      // Get the response as an ArrayBuffer
      const compressedData = await response.arrayBuffer();

      // Decompress the gzipped data
      const jsonString = await decompressGzip(compressedData);
      const data = JSON.parse(jsonString);

      // Process the features and filter by size if needed
      results = data.features.map((feature) => {
        // Ensure the feature has a valid ID
        if (!feature.id) {
          feature.id = `${inseeCode}-${Math.random().toString(36).substr(2, 9)}`;
        }

        // Add the area to the properties for easier access
        if (feature.properties && feature.properties.contenance) {
          feature.properties.surface_parcelle = feature.properties.contenance;
        }

        return feature;
      });

      // Filter by size using minThumb and maxThumb
      results = results.filter((feature) => {
        const area = feature.properties?.contenance || 0;
        return area >= minThumb && area <= maxThumb;
      });

      console.log("Results:", results);
    } catch (err) {
      console.error("Error fetching data:", err);
      error = "Error fetching data. Please check the INSEE code and try again.";
    } finally {
      loading = false;
    }
  };

  const selectParcel = (parcel) => {
    selectedParcel = parcel;
  };

  // Function to search for communes by name
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

  // Function to select a commune from search results
  const selectCommune = (commune) => {
    inseeCode = commune.inseeCode;
    communeName = commune.name;
    communeSearchResults = [];
    showCommuneResults = false;
  };

  // Function to update thumb values while maintaining constraints
  function updateThumbs(newMin, newMax) {
    // Ensure min is within bounds and less than max
    newMin = Math.max(MIN_SIZE, Math.min(newMin, maxThumb - 1));
    // Ensure max is within bounds and greater than min
    newMax = Math.min(MAX_SIZE, Math.max(newMax, newMin + 1));

    // Only update if values actually changed to prevent infinite loops
    if (minThumb !== newMin || maxThumb !== newMax) {
      minThumb = newMin;
      maxThumb = newMax;
    }
  }

  // Update thumbs when they change
  $: updateThumbs(minThumb, maxThumb);
</script>

<main class="main-container">
  <div class="map-container">
    <Map
      {results}
      {selectedParcel}
      on:parcel-select={(e) => selectParcel(e.detail)}
    />
  </div>

  <!-- Search Form Overlay -->
  <div
    class="search-form-overlay {searchFocused ? 'focused' : ''}"
    on:click|self={() => (searchFocused = false)}
  >
    <div class="search-form-container">
      <div class="search-form">
        <!-- Form content -->
        <div class="form-row">
          <div class="form-group" style="flex: 2; position: relative;">
            <div class="input-with-button">
              <input
                type="text"
                id="commune"
                bind:value={communeName}
                on:input={searchCommune}
                placeholder="Search by commune name..."
                class="form-input"
                aria-label="Search by commune name"
              />
              <button
                class="search-button"
                on:click={searchProperties}
                disabled={!inseeCode || loading}
                type="button"
                title="Search properties"
                aria-label="Search properties"
              >
                {#if loading}
                  <svg
                    class="animate-spin -ml-1 mr-2 h-5 w-5 text-white"
                    xmlns="http://www.w3.org/2000/svg"
                    fill="none"
                    viewBox="0 0 24 24"
                  >
                    <circle
                      class="opacity-25"
                      cx="12"
                      cy="12"
                      r="10"
                      stroke="currentColor"
                      stroke-width="4"
                    ></circle>
                    <path
                      class="opacity-75"
                      fill="currentColor"
                      d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"
                    ></path>
                  </svg>
                {:else}
                  <svg
                    xmlns="http://www.w3.org/2000/svg"
                    width="16"
                    height="16"
                    viewBox="0 0 24 24"
                    fill="none"
                    stroke="currentColor"
                    stroke-width="2"
                    stroke-linecap="round"
                    stroke-linejoin="round"
                  >
                    <circle cx="11" cy="11" r="8"></circle>
                    <line x1="21" y1="21" x2="16.65" y2="16.65"></line>
                  </svg>
                {/if}
              </button>
            </div>
            {#if showCommuneResults && communeSearchResults.length > 0}
              <div class="dropdown" role="listbox">
                {#each communeSearchResults as commune (commune.inseeCode)}
                  <button
                    class="dropdown-item"
                    on:click|stopPropagation={() => selectCommune(commune)}
                    on:keydown={(e) =>
                      e.key === "Enter" && selectCommune(commune)}
                    role="option"
                    tabindex="0"
                  >
                    <div class="flex justify-between items-baseline">
                      <span class="font-medium text-gray-900"
                        >{commune.name}</span
                      >
                      <span class="text-sm text-gray-500"
                        >{commune.population} hab.</span
                      >
                    </div>
                    <div class="flex justify-between text-sm">
                      <span class="text-blue-600 font-mono"
                        >{commune.inseeCode}</span
                      >
                      <span class="text-gray-500">{commune.postalCode}</span>
                    </div>
                  </button>
                {/each}
              </div>
            {/if}
          </div>
        </div>

        <div class="form-group range-slider-container">
          <div class="range-display">
            <span class="min-value">{Math.round(minThumb)}–{Math.round(maxThumb)} m²</span>
          </div>
          <DoubleSlider
            min={MIN_SIZE}
            max={MAX_SIZE}
            bind:start={minThumb}
            bind:end={maxThumb}
            on:change={updateSizes}
          />
        </div>

        <div class="form-actions">
          <button
            on:click={searchProperties}
            disabled={loading || (!inseeCode && !communeName)}
            class="search-submit"
          >
            {#if loading}
              <span class="spinner"></span>
              <span>Searching...</span>
            {:else}
              <svg
                xmlns="http://www.w3.org/2000/svg"
                width="16"
                height="16"
                viewBox="0 0 24 24"
                fill="none"
                stroke="currentColor"
                stroke-width="2"
                stroke-linecap="round"
                stroke-linejoin="round"
                class="mr-2"
              >
                <circle cx="11" cy="11" r="8"></circle>
                <line x1="21" y1="21" x2="16.65" y2="16.65"></line>
              </svg>
              <span>Search</span>
            {/if}
          </button>
        </div>
      </div>
      {#if error}
        <div class="error-message">
          <svg
            xmlns="http://www.w3.org/2000/svg"
            width="16"
            height="16"
            viewBox="0 0 24 24"
            fill="none"
            stroke="currentColor"
            stroke-width="2"
            stroke-linecap="round"
            stroke-linejoin="round"
            class="mr-2"
          >
            <circle cx="12" cy="12" r="10"></circle>
            <line x1="12" y1="8" x2="12" y2="12"></line>
            <line x1="12" y1="16" x2="12.01" y2="16"></line>
          </svg>
          {error}
        </div>
      {/if}
    </div>
  </div>
  <!-- Results Carousel -->
  {#if results.length > 0}
    <div class="carousel-container {!carouselOpen ? 'collapsed' : ''}">

      <div
        class="carousel-track"
        style="transform: translateX(-${currentSlide * 100}%)"
      >
        {#each results as parcel, i (parcel.id)}
          <div
            class="carousel-slide {selectedParcel?.id === parcel.id
              ? 'active'
              : ''}"
            on:click={() => selectParcel(parcel)}
            on:keydown={(e) => e.key === "Enter" && selectParcel(parcel)}
            tabindex="0"
          >
            <div class="parcel-card">
              <div class="parcel-area">
                <span class="area-value">{parcel.properties?.surface_parcelle || "N/A"}</span>
                <span class="area-unit">m²</span>
              </div>
              <div class="parcel-details">
                <span class="parcel-id">#{parcel.properties?.id || "N/A"}</span>
              </div>
            </div>
          </div>
        {/each}
      </div>

    </div>
  {/if}
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


  /* Search Form Overlay */
  .search-form-overlay {
    position: absolute;
    top: 20px;
    left: 20px;
    right: 20px;
    max-width: 500px;
    z-index: 1000;
    transition: all 0.3s ease;
  }

  .search-form-container {
    background: white;
    border-radius: 8px;
    box-shadow: 0 2px 10px rgba(0, 0, 0, 0.2);
    overflow: hidden;
  }

  .search-form-overlay.focused {
    max-width: 600px;
  }

  /* Removed unused selectors */

  :global(*:focus) {
    outline: 2px solid #3b82f6;
    outline-offset: 2px;
  }

  .input-with-button {
    display: flex;
    position: relative;
    height: 2.5rem;
    align-items: stretch;
  }

  .input-with-button .form-input {
    flex: 1;
    padding: 0.5rem 1rem;
    border: 1px solid #d1d5db;
    border-radius: 0.375rem 0 0 0.375rem;
    border-right: none;
    font-size: 0.875rem;
    line-height: 1.5;
    color: #111827;
    background-color: #ffffff;
    transition:
      border-color 0.15s ease-in-out,
      box-shadow 0.15s ease-in-out;
    height: auto;
    box-sizing: border-box;
    margin: 0;
  }

  .form-input:focus {
    outline: none;
    border-color: #3b82f6;
    box-shadow: 0 0 0 1px #3b82f6;
  }

  .input-with-button input:focus {
    border-color: #3b82f6;
    box-shadow: 0 0 0 1px #3b82f6;
  }

  .search-button {
    background: #3b82f6;
    color: white;
    border: 1px solid #3b82f6;
    border-radius: 0 0.375rem 0.375rem 0;
    padding: 0 1rem;
    display: flex;
    align-items: center;
    justify-content: center;
    cursor: pointer;
    transition: all 0.2s ease-in-out;
    height: 100%;
    min-width: 3rem;
    box-sizing: border-box;
    margin: 0;
  }

  .search-button:hover {
    background: #2563eb;
  }

  .search-button:active {
    transform: translateY(1px);
  }

  .search-button:disabled {
    background: #cbd5e1;
    color: #64748b;
    cursor: not-allowed;
    transform: none;
    box-shadow: none;
  }

  .dropdown {
    position: absolute;
    top: calc(100% + 4px);
    left: 0;
    right: 0;
    background: white;
    border: 1px solid #e2e8f0;
    border-radius: 0.5rem;
    box-shadow:
      0 4px 6px -1px rgba(0, 0, 0, 0.1),
      0 2px 4px -1px rgba(0, 0, 0, 0.06);
    z-index: 50;
    max-height: 20rem;
    overflow-y: auto;
    margin-top: 0.25rem;
    padding: 0.5rem 0;
  }

  .dropdown-item {
    padding: 0.5rem 1rem;
    cursor: pointer;
    text-align: left;
    width: 100%;
    background: none;
    border: none;
    font-size: 0.875rem;
    color: #1e293b;
    transition: background-color 0.1s;
    display: flex;
    flex-direction: column;
    gap: 0.125rem;
  }

  .dropdown-item:hover,
  .dropdown-item:focus {
    background-color: #f1f5f9;
  }

  .dropdown-item:last-child {
    border-bottom: none;
  }

  .form-row {
    display: flex;
    gap: 15px;
    margin-bottom: 15px;
  }

  .form-group {
    flex: 1;
    position: relative;
  }

  .search-form {
    background: white;
    padding: 15px;
    border-radius: 8px;
    margin-bottom: 20px;
  }

  .search-form h2 {
    margin-top: 0;
    margin-bottom: 20px;
    color: #333;
  }

  .form-row {
    display: flex;
    gap: 15px;
  }

  .form-row .form-group {
    flex: 1;
  }

  label {
    display: block;
    margin-bottom: 5px;
    font-weight: bold;
    color: #333;
  }

  input {
    width: auto;
    padding: 8px 12px;
    border: 1px solid #ddd;
    border-radius: 4px;
    font-size: 16px;
  }

  .error-message {
    color: #d32f2f;
    margin-top: 10px;
    padding: 10px;
    background-color: #ffebee;
    border-radius: 4px;
    font-size: 0.9em;
  }

  /* Carousel Container */
  .carousel-container {
    position: fixed;
    bottom: 0;
    left: 0;
    right: 0;
    height: 160px; /* Reduced height for more compact look */
    z-index: 900;
    display: flex;
    align-items: center;
    padding: 10px 20px;
    pointer-events: none; /* Allow clicks to pass through to elements below */
  }

  .carousel-track {
    display: flex;
    gap: 12px;
    overflow-x: auto;
    scroll-snap-type: x mandatory;
    height: 100%;
    scrollbar-width: none; /* Firefox */
    -ms-overflow-style: none; /* IE and Edge */
    width: 100%;
    padding: 10px 0;
    box-sizing: border-box;
    pointer-events: auto; /* Re-enable pointer events for the track */
  }

  .carousel-track::-webkit-scrollbar {
    display: none; /* Chrome, Safari, Opera */
  }

  .carousel-slide {
    flex: 0 0 200px;
    scroll-snap-align: start;
    transition: all 0.2s ease;
    height: 80px;
    display: flex;
    flex-direction: column;
    background: rgba(255, 255, 255, 0.95);
    border-radius: 8px;
    backdrop-filter: blur(4px);
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
    margin: 4px 0;
  }

  .carousel-slide:focus {
    outline: 2px solid #3b82f6;
    outline-offset: 2px;
  }

  .parcel-card {
    background: transparent;
    padding: 8px 12px;
    height: 100%;
    cursor: pointer;
    transition: all 0.15s ease;
    display: flex;
    align-items: center;
    gap: 12px;
    flex: 1;
  }

  .parcel-card:hover,
  .carousel-slide.active .parcel-card {
    transform: translateY(-1px);
  }
  
  .carousel-slide.active {
    background: rgba(255, 255, 255, 0.98);
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.12);
    border: 1px solid #e2e8f0;
  }

  .parcel-card h3 {
    margin: 0 0 4px 0;
    color: #1a202c;
    font-size: 0.9375rem;
    font-weight: 600;
  }

  .parcel-card p {
    margin: 2px 0;
    color: #4a5568;
    font-size: 0.8125rem;
    opacity: 0.9;
  }

  .main-container {
    position: relative;
    overflow: hidden;
    height: 100vh;
    display: flex;
    flex-direction: column;
  }

  .map-container {
    flex: 1;
    min-width: 0;
    border: none;
    border-radius: 0;
    overflow: hidden;
    box-shadow: none;
    margin: 0;
    width: 100%;
    height: 100%;
  }

  .loading {
    text-align: center;
    padding: 20px;
    color: #666;
  }

  /* Mobile Toggle Button */
  .mobile-toggle {
    position: fixed;
    top: 16px;
    left: 16px;
    z-index: 1000;
    background: white;
    border: none;
    border-radius: 50%;
    width: 48px;
    height: 48px;
    display: flex;
    align-items: center;
    justify-content: center;
    box-shadow: 0 2px 10px rgba(0, 0, 0, 0.2);
    cursor: pointer;
    z-index: 1001;
  }

  .mobile-toggle svg {
    width: 24px;
    height: 24px;
  }

  @media (max-width: 768px) {
    .search-form-overlay {
      top: 10px;
      left: 10px;
      right: 10px;
      max-width: none;
    }

    .carousel-slide {
      flex: 0 0 200px;
    }

    .parcel-card {
      padding: 10px;
    }

    .parcel-card h3 {
      font-size: 0.9375rem;
    }

    .parcel-card p {
      font-size: 0.8125rem;
    }
  }

  @media (max-width: 900px) {
    .results-container {
      flex-direction: column;
      height: auto;
    }

    .results-list {
      flex: 0 0 auto;
      max-height: 50vh;
      width: 100%;
      max-width: 100%;
      margin-bottom: 20px;
    }

    .search-form {
      max-width: 100%;
      margin-bottom: 20px;
    }

    .form-row {
      flex-direction: column;
      gap: 10px;
    }

    .form-group {
      width: 100%;
    }

    .map-container {
      width: 100%;
      margin-right: 0;
    }

    .input-with-button {
      width: 100%;
    }

    button[type="submit"] {
      width: 100%;
    }
  }

  @media (max-width: 480px) {
    .search-form {
      padding: 12px;
    }

    .results-list {
      padding: 12px;
    }

    .overlay-container {
      padding: 16px;
    }
  }
</style>
