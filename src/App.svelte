<script>
  import { onMount } from 'svelte';
  import axios from 'axios';
  import Map from './Map.svelte';
  import DoubleSlider from './lib/DoubleSlider.svelte';
  
  // State
  let inseeCode = '';
  let communeName = '';
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
  
  // Function to decompress gzipped data
  const decompressGzip = async (compressedData) => {
    const ds = new DecompressionStream('gzip');
    const decompressedStream = new Response(compressedData).body.pipeThrough(ds);
    return new Response(decompressedStream).text();
  };

  // Update sizes when DoubleSlider values change
  function updateSizes() {
    // Values are already in the correct range (0-3000) from the DoubleSlider
  }

  // Function to search for properties
  const searchProperties = async () => {
    if (!inseeCode) {
      error = 'Please enter an INSEE code';
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
        max_size: maxThumb.toString()
      });
      
      const response = await fetch(url, {
        headers: {
          'Accept-Encoding': 'gzip',
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
      results = data.features.map(feature => {
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
      results = results.filter(feature => {
        const area = feature.properties?.contenance || 0;
        return area >= minThumb && area <= maxThumb;
      });
      
      console.log('Results:', results);
    } catch (err) {
      console.error('Error fetching data:', err);
      error = 'Error fetching data. Please check the INSEE code and try again.';
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
      const response = await fetch(`https://geo.api.gouv.fr/communes?nom=${encodeURIComponent(communeName)}&boost=population&limit=10`);
      if (!response.ok) {
        throw new Error('Failed to fetch communes');
      }
      
      const data = await response.json();
      communeSearchResults = data.map(commune => ({
        name: commune.nom,
        inseeCode: commune.code,
        postalCode: commune.codesPostaux?.[0] || '',
        population: commune.population?.toLocaleString() || 'N/A'
      }));
      
      showCommuneResults = true;
    } catch (err) {
      console.error('Error searching for communes:', err);
      error = 'Failed to search for communes. Please try again.';
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

<main style="position: relative; overflow: hidden;">
  <div class="map-fullscreen">
    <Map {results} {selectedParcel} on:parcel-select={e => selectParcel(e.detail)} />
  </div>
  
  <div class="overlay-container">
    <div class="search-form">
      <div class="form-row">
        <div class="form-group" style="flex: 2; position: relative;">
          <label for="commune" class="form-label">Commune Name</label>
          <div class="input-with-button">
            <input 
              type="text" 
              id="commune" 
              bind:value={communeName} 
              on:input={searchCommune}
              placeholder="e.g., Paris"
              class="form-input"
              disabled={loading}
            />
            <button 
              class="search-button" 
              on:click={searchCommune} 
              type="button"
              disabled={loading || !communeName}
              title="Search communes"
              aria-label="Search communes"
            >
              {#if loading && !communeName}
                <span class="spinner"></span>
              {:else}
                <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
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
                  on:keydown={(e) => e.key === 'Enter' && selectCommune(commune)}
                  role="option"
                  tabindex="0"
                >
                  <div class="flex justify-between items-baseline">
                    <span class="font-medium text-gray-900">{commune.name}</span>
                    <span class="text-sm text-gray-500">{commune.population} hab.</span>
                  </div>
                  <div class="flex justify-between text-sm">
                    <span class="text-blue-600 font-mono">{commune.inseeCode}</span>
                    <span class="text-gray-500">{commune.postalCode}</span>
                  </div>
                </button>
              {/each}
            </div>
          {/if}
        </div>
        
      </div>
      
      <div class="form-group range-slider-container">
        <label class="form-label">Size Range (m²)</label>
        <div class="range-display">
          <span class="min-value">{Math.round(minThumb)} m²</span>
          <span class="max-value">{Math.round(maxThumb)} m²</span>
        </div>
        <DoubleSlider
          min={MIN_SIZE}
          max={MAX_SIZE}
          bind:start={minThumb}
          bind:end={maxThumb}
          on:change={updateSizes}
        />
        <div class="range-labels">
          <span>{MIN_SIZE} m²</span>
          <span>{MAX_SIZE} m²</span>
        </div>
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
            <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="mr-2">
              <circle cx="11" cy="11" r="8"></circle>
              <line x1="21" y1="21" x2="16.65" y2="16.65"></line>
            </svg>
            <span>Search Properties</span>
          {/if}
        </button>
      </div>
      
      {#if error}
        <div class="error-message">
          <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="mr-2">
            <circle cx="12" cy="12" r="10"></circle>
            <line x1="12" y1="8" x2="12" y2="12"></line>
            <line x1="12" y1="16" x2="12.01" y2="16"></line>
          </svg>
          {error}
        </div>
      {/if}
    </div>
    
    <div class="results-list">
      <h2>Results ({results.length})</h2>
      <div class="parcel-list">
        {#if results.length > 0}
          {#each results as parcel (parcel.id)}
            <div 
              class="parcel-item {selectedParcel === parcel ? 'selected' : ''}"
              on:click={() => selectParcel(parcel)}
              on:keydown={(e) => e.key === 'Enter' && selectParcel(parcel)}
              tabindex="0"
              role="button"
            >
              <h3>Parcel ID: {parcel.id}</h3>
              <p>Area: {parcel.properties.surface_parcelle || 'N/A'} m²</p>
              <p>Section: {parcel.properties.section || 'N/A'}</p>
              <p>Number: {parcel.properties.number || 'N/A'}</p>
            </div>
          {/each}
        {:else if loading}
          <div class="loading">Loading...</div>
        {:else if inseeCode && !loading}
          <p>No parcels found. Try adjusting your search criteria.</p>
        {/if}
      </div>
    </div>
  </div>
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

  .map-fullscreen {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    z-index: 1;
  }

  .map-fullscreen :global(.map) {
    width: 100% !important;
    height: 100% !important;
  }

  .overlay-container {
    position: relative;
    z-index: 2;
    display: flex;
    flex-direction: column;
    height: 100vh;
    padding: 20px;
    pointer-events: none;
  }

  .overlay-container > * {
    pointer-events: auto;
  }

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
    transition: border-color 0.15s ease-in-out, box-shadow 0.15s ease-in-out;
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
    box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1), 0 2px 4px -1px rgba(0, 0, 0, 0.06);
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
  
  .dropdown-item:hover, .dropdown-item:focus {
    background-color: #f1f5f9;
  }
  
  .dropdown-item:last-child {
    border-bottom: none;
  }
  
  .insee-code {
    font-size: 0.8em;
    color: #666;
    margin-top: 2px;
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
  
  .container {
    max-width: 100%;
    margin: 0;
    padding: 20px;
    width: 100%;
  }
  
  h1 {
    color: #2c3e50;
    text-align: center;
    margin-bottom: 30px;
  }
  
  .search-form {
    background: rgba(255, 255, 255, 0.9);
    padding: 15px;
    border-radius: 8px;
    margin-bottom: 20px;
    box-shadow: 0 2px 10px rgba(0, 0, 0, 0.2);
    max-width: 600px;
    backdrop-filter: blur(5px);
    border: 1px solid rgba(0, 0, 0, 0.1);
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
  
  button {
    background-color: #4CAF50;
    color: white;
    border: none;
    padding: 10px 20px;
    text-align: center;
    text-decoration: none;
    display: inline-block;
    font-size: 16px;
    margin: 10px 0;
    cursor: pointer;
    border-radius: 4px;
    transition: background-color 0.3s;
  }
  
  button:hover:not(:disabled) {
    background-color: #45a049;
  }
  
  button:disabled {
    background-color: #cccccc;
    cursor: not-allowed;
  }
  
  .error {
    color: #d32f2f;
    margin-top: 10px;
    padding: 10px;
    background-color: #ffebee;
    border-radius: 4px;
  }
  
  .results-container {
    display: flex;
    flex-direction: column;
    gap: 20px;
    flex: 1;
    max-width: 600px;
  }
  
  .results-list {
    flex: 0 0 auto;
    max-height: 50vh;
    overflow-y: auto;
    border: 1px solid rgba(0, 0, 0, 0.1);
    border-radius: 8px;
    padding: 15px;
    background: rgba(255, 255, 255, 0.9);
    box-shadow: 0 2px 10px rgba(0, 0, 0, 0.2);
    max-width: 400px;
    backdrop-filter: blur(5px);
  }
  
  .map-container {
    flex: 1;
    min-width: 0;
    border: 1px solid #e2e8f0;
    border-radius: 8px;
    overflow: hidden;
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
    margin-right: 0;
    width: 100%;
  }
  
  .parcel-list {
    display: flex;
    flex-direction: column;
    gap: 10px;
  }
  
  .parcel-item {
    padding: 12px;
    border: 1px solid rgba(0, 0, 0, 0.1);
    border-radius: 6px;
    cursor: pointer;
    transition: all 0.2s;
    background: rgba(255, 255, 255, 0.9);
    margin-bottom: 8px;
  }

  .parcel-item:focus {
    outline: 2px solid #3b82f6;
    outline-offset: 2px;
  }
  
  .parcel-item:hover {
    background-color: #f0f0f0;
  }
  
  .parcel-item.selected {
    border-color: #4CAF50;
    background-color: #e8f5e9;
  }
  
  .parcel-item h3 {
    margin: 0 0 5px 0;
    color: #2c3e50;
  }
  
  .parcel-item p {
    margin: 2px 0;
    font-size: 14px;
    color: #555;
  }
  
  .loading {
    text-align: center;
    padding: 20px;
    color: #666;
  }
  
  @media (max-width: 900px) {
    .results-container {
      flex-direction: column;
      height: auto;
    }
    
    .results-list {
      flex: 0 0 auto;
      max-height: 300px;
      width: 100%;
    }
    
    .map-container {
      height: 400px;
    }
  }
</style>
