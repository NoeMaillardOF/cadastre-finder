<script>
  const MIN_SIZE = 0;
  const MAX_SIZE = 8000;

  let {
    inseeCode = $bindable(""),
    communeName = $bindable(""),
    minThumb = $bindable(MIN_SIZE),
    maxThumb = $bindable(MAX_SIZE),
    loading = false,
    error = null,
    communeSearchResults = $bindable([]),
    showCommuneResults = $bindable(false),
    onsearch,
    onsearchcommune,
    onselectcommune,
  } = $props();

  let searchFocused = $state(false);

  function handleMinChange(event) {
    const value = parseInt(event.target.value) || MIN_SIZE;
    minThumb = Math.max(MIN_SIZE, Math.min(value, maxThumb));
  }

  function handleMaxChange(event) {
    const value = parseInt(event.target.value) || MAX_SIZE;
    maxThumb = Math.min(MAX_SIZE, Math.max(value, minThumb));
  }
</script>

<div
  class="search-form-overlay {searchFocused ? 'focused' : ''}"
  role="presentation"
  onclick={(e) => { if (e.target === e.currentTarget) searchFocused = false; }}
>
  <div class="search-form-container">
    <div class="search-form">
      <div class="form-row">
        <div class="form-group" style="flex: 2; position: relative;">
          <div class="input-with-button">
            <input
              type="text"
              id="commune"
              bind:value={communeName}
              oninput={onsearchcommune}
              placeholder="Search by commune name..."
              class="form-input"
              aria-label="Search by commune name"
            />
            <button
              class="search-button"
              onclick={onsearch}
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
                  onclick={(e) => { e.stopPropagation(); onselectcommune?.(commune); }}
                  onkeydown={(e) =>
                    e.key === "Enter" && onselectcommune?.(commune)}
                  role="option"
                  aria-selected={false}
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
        <div class="range-inputs">
          <div class="input-group">
            <label for="minSize">Min (m²)</label>
            <input
              type="number"
              id="minSize"
              bind:value={minThumb}
              onchange={handleMinChange}
              min={MIN_SIZE}
              max={maxThumb}
              class="form-input size-input"
              aria-label="Minimum size in square meters"
            />
          </div>
          <div class="input-separator">–</div>
          <div class="input-group">
            <label for="maxSize">Max (m²)</label>
            <input
              type="number"
              id="maxSize"
              bind:value={maxThumb}
              onchange={handleMaxChange}
              min={minThumb}
              max={MAX_SIZE}
              class="form-input size-input"
              aria-label="Maximum size in square meters"
            />
          </div>
        </div>
      </div>

      <div class="form-actions">
        <button
          onclick={onsearch}
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

<style>
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

  .search-form {
    background: white;
    padding: 15px;
    border-radius: 8px;
    margin-bottom: 20px;
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

  .range-slider-container {
    margin-bottom: 15px;
  }

  .range-inputs {
    display: flex;
    align-items: flex-end;
    gap: 12px;
    width: 100%;
  }

  .input-group {
    display: flex;
    flex-direction: column;
    flex: 1;
  }

  .input-group label {
    font-size: 0.75rem;
    color: #64748b;
    margin-bottom: 4px;
    font-weight: 500;
  }

  .input-separator {
    font-size: 1.25rem;
    color: #64748b;
    padding-bottom: 8px;
    font-weight: 500;
  }

  .size-input {
    width: 100%;
    padding: 0.5rem 0.75rem;
    border: 1px solid #d1d5db;
    border-radius: 0.375rem;
    font-size: 0.875rem;
    line-height: 1.5;
    color: #111827;
    background-color: #ffffff;
    transition:
      border-color 0.15s ease-in-out,
      box-shadow 0.15s ease-in-out;
    box-sizing: border-box;
  }

  .size-input:focus {
    outline: none;
    border-color: #3b82f6;
    box-shadow: 0 0 0 1px #3b82f6;
  }

  .size-input::-webkit-outer-spin-button,
  .size-input::-webkit-inner-spin-button {
    -webkit-appearance: none;
    margin: 0;
  }

  .size-input[type="number"] {
    -moz-appearance: textfield;
  }

  @media (max-width: 768px) {
    .search-form-overlay {
      top: 10px;
      left: 10px;
      right: 10px;
      max-width: none;
    }
  }

  @media (max-width: 900px) {
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

    .input-with-button {
      width: 100%;
    }
  }
</style>
