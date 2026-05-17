<script>
  let {
    results = [],
    selectedParcel = $bindable(null),
    selectedParcelIndex = $bindable(-1),
    onselectparcel,
  } = $props();

  let carouselOpen = $state(true);
  let currentSlide = $state(0);

  function selectNextParcel() {
    if (results.length === 0) return;
    selectedParcelIndex = (selectedParcelIndex + 1) % results.length;
    selectedParcel = results[selectedParcelIndex];
    scrollToSelectedParcel();
  }

  function selectPreviousParcel() {
    if (results.length === 0) return;
    selectedParcelIndex = selectedParcelIndex <= 0 ? results.length - 1 : selectedParcelIndex - 1;
    selectedParcel = results[selectedParcelIndex];
    scrollToSelectedParcel();
  }

  function scrollToSelectedParcel() {
    const carouselTrack = document.querySelector('.carousel-track');
    const selectedSlide = document.querySelectorAll('.carousel-slide')[selectedParcelIndex];
    if (carouselTrack && selectedSlide) {
      selectedSlide.scrollIntoView({ behavior: 'smooth', block: 'nearest', inline: 'center' });
    }
  }

  function handleKeyDown(event) {
    if (results.length === 0) return;

    if (event.key === 'ArrowRight') {
      event.preventDefault();
      selectNextParcel();
    } else if (event.key === 'ArrowLeft') {
      event.preventDefault();
      selectPreviousParcel();
    }
  }

  function toggleCarousel() {
    carouselOpen = !carouselOpen;
  }

  $effect(() => {
    window.addEventListener("keydown", handleKeyDown);
    return () => {
      window.removeEventListener("keydown", handleKeyDown);
    };
  });
</script>

{#if results.length > 0}
  <div class="carousel-container {!carouselOpen ? 'collapsed' : ''}">
    <button
      class="carousel-nav carousel-nav-left"
      onclick={selectPreviousParcel}
      disabled={results.length === 0}
      title="Previous parcel (Left Arrow)"
      aria-label="Previous parcel"
    >
      <svg
        xmlns="http://www.w3.org/2000/svg"
        width="20"
        height="20"
        viewBox="0 0 24 24"
        fill="none"
        stroke="currentColor"
        stroke-width="2"
        stroke-linecap="round"
        stroke-linejoin="round"
      >
        <polyline points="15 18 9 12 15 6"></polyline>
      </svg>
    </button>

    <div
      class="carousel-track"
      style="transform: translateX(-{currentSlide * 100}%)"
    >
      {#each results as parcel, i (parcel.id)}
        <div
          class="carousel-slide {selectedParcel?.id === parcel.id
            ? 'active'
            : ''}"
          role="button"
          onclick={() => onselectparcel?.(parcel, i)}
          onkeydown={(e) => e.key === "Enter" && onselectparcel?.(parcel, i)}
          tabindex="0"
        >
          <div class="parcel-card">
            <div class="parcel-area {parcel.properties?.isCombined ? 'combined' : ''}">
              <span class="area-value"
                >{Math.round(parcel.properties?.surface_parcelle) || "N/A"}</span
              >
              <span class="area-unit">m²</span>
            </div>
            <div class="parcel-details">
              {#if parcel.properties?.isCombined}
                <span class="parcel-type combined-badge">Combined</span>
                <span class="parcel-count">{parcel.properties.combinedParcelCount} parcels</span>
              {:else if parcel.properties?.type !== undefined}
                <span class="parcel-type">
                  {parcel.properties.type === "B" ? "En dur" : parcel.properties.type === "L" ? "Léger" : parcel.properties.type}
                  {#if parcel.properties.nom}
                    <span class="building-name">— {parcel.properties.nom}</span>
                  {/if}
                </span>
              {:else}
                <span class="parcel-id">#{parcel.properties?.id || "N/A"}</span>
              {/if}
              {#if parcel.properties?.address}
                <a
                  href="https://www.google.com/maps/search/?api=1&query={parcel.properties.coordinates.lat},{parcel.properties.coordinates.lon}"
                  target="_blank"
                  rel="noopener noreferrer"
                  class="parcel-address"
                  onclick={(e) => e.stopPropagation()}
                  title="Open in Google Maps"
                >
                  <svg
                    xmlns="http://www.w3.org/2000/svg"
                    width="12"
                    height="12"
                    viewBox="0 0 24 24"
                    fill="none"
                    stroke="currentColor"
                    stroke-width="2"
                    stroke-linecap="round"
                    stroke-linejoin="round"
                    class="address-icon"
                  >
                    <path d="M21 10c0 7-9 13-9 13s-9-6-9-13a9 9 0 0 1 18 0z"></path>
                    <circle cx="12" cy="10" r="3"></circle>
                  </svg>
                  {parcel.properties.address}
                </a>
              {/if}
            </div>
          </div>
        </div>
      {/each}
    </div>

    <button
      class="carousel-nav carousel-nav-right"
      onclick={selectNextParcel}
      disabled={results.length === 0}
      title="Next parcel (Right Arrow)"
      aria-label="Next parcel"
    >
      <svg
        xmlns="http://www.w3.org/2000/svg"
        width="20"
        height="20"
        viewBox="0 0 24 24"
        fill="none"
        stroke="currentColor"
        stroke-width="2"
        stroke-linecap="round"
        stroke-linejoin="round"
      >
        <polyline points="9 18 15 12 9 6"></polyline>
      </svg>
    </button>

    <div class="keyboard-hint">
      <span>Use ← → arrow keys to navigate</span>
    </div>
  </div>
{/if}

<style>
  .carousel-container {
    position: fixed;
    bottom: 0;
    left: 0;
    right: 0;
    height: 160px;
    z-index: 900;
    display: flex;
    align-items: center;
    padding: 10px 20px;
    pointer-events: none;
  }

  .carousel-nav {
    position: absolute;
    top: 50%;
    transform: translateY(-50%);
    background: rgba(255, 255, 255, 0.95);
    border: 1px solid #e2e8f0;
    border-radius: 50%;
    width: 40px;
    height: 40px;
    display: flex;
    align-items: center;
    justify-content: center;
    cursor: pointer;
    pointer-events: auto;
    z-index: 10;
    transition: all 0.2s ease;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  }

  .carousel-nav:hover:not(:disabled) {
    background: white;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
    border-color: #3b82f6;
  }

  .carousel-nav:active:not(:disabled) {
    transform: translateY(-50%) scale(0.95);
  }

  .carousel-nav:disabled {
    opacity: 0.3;
    cursor: not-allowed;
  }

  .carousel-nav-left {
    left: 10px;
  }

  .carousel-nav-right {
    right: 10px;
  }

  .carousel-nav svg {
    color: #374151;
  }

  .carousel-nav:hover:not(:disabled) svg {
    color: #3b82f6;
  }

  .keyboard-hint {
    position: absolute;
    top: -30px;
    left: 50%;
    transform: translateX(-50%);
    background: rgba(0, 0, 0, 0.75);
    color: white;
    padding: 6px 12px;
    border-radius: 6px;
    font-size: 0.75rem;
    pointer-events: none;
    white-space: nowrap;
    opacity: 0;
    animation: fadeInOut 3s ease-in-out 2s;
  }

  @keyframes fadeInOut {
    0%, 100% { opacity: 0; }
    10%, 90% { opacity: 1; }
  }

  .carousel-track {
    display: flex;
    gap: 12px;
    overflow-x: auto;
    scroll-snap-type: x mandatory;
    height: 100%;
    scrollbar-width: none;
    -ms-overflow-style: none;
    width: 100%;
    padding: 10px 0;
    box-sizing: border-box;
    pointer-events: auto;
  }

  .carousel-track::-webkit-scrollbar {
    display: none;
  }

  .carousel-slide {
    flex: 0 0 250px;
    scroll-snap-align: start;
    transition: all 0.3s ease;
    height: 90px;
    display: flex;
    flex-direction: column;
    background: #ffffff;
    border-radius: 8px;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
    margin: 4px 0;
    color: #1a202c;
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
    color: inherit;
  }

  .parcel-card:hover,
  .carousel-slide.active .parcel-card {
    transform: translateY(-1px);
  }

  .carousel-slide.active {
    background: #ffffff;
    box-shadow: 0 4px 16px rgba(59, 130, 246, 0.3);
    border: 2px solid #3b82f6;
    transform: scale(1.05);
    color: #1a202c;
    animation: pulseActive 0.3s ease-out;
  }

  @keyframes pulseActive {
    0% {
      transform: scale(1);
    }
    50% {
      transform: scale(1.08);
    }
    100% {
      transform: scale(1.05);
    }
  }

  .carousel-slide.active .parcel-area {
    background: #dbeafe;
    border-color: #3b82f6;
  }

  .parcel-area {
    background: #f0f9ff;
    border-radius: 6px;
    padding: 6px 10px;
    min-width: 70px;
    text-align: center;
    border: 1px solid #e0f2fe;
    color: #0369a1;
  }

  .parcel-area.combined {
    background: #f3e8ff;
    border-color: #d8b4fe;
    color: #7c3aed;
  }

  .parcel-count {
    font-weight: 500;
    color: #7c3aed;
    font-size: 0.8rem;
  }

  .combined-badge {
    color: #7c3aed !important;
  }

  .parcel-id {
    font-weight: 500;
    color: #1e293b !important;
    font-size: 0.85rem;
  }

  .parcel-type {
    font-weight: 500;
    color: #1e293b;
    font-size: 0.85rem;
  }

  .building-name {
    font-weight: 400;
    color: #6b7280;
    font-size: 0.8rem;
  }

  .parcel-details {
    display: flex;
    flex-direction: column;
    gap: 0.1rem;
    color: #1a202c;
  }

  .parcel-address {
    display: flex;
    align-items: center;
    gap: 4px;
    font-size: 0.75rem;
    color: #3b82f6;
    text-decoration: none;
    margin-top: 2px;
    transition: color 0.2s ease;
    max-width: 100%;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  .parcel-address:hover {
    color: #2563eb;
    text-decoration: underline;
  }

  .address-icon {
    flex-shrink: 0;
  }

  @media (max-width: 768px) {
    .carousel-slide {
      flex: 0 0 200px;
    }

    .parcel-card {
      padding: 10px;
    }
  }
</style>
