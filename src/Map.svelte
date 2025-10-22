<script>
  import { onMount, onDestroy } from 'svelte';
  import 'leaflet/dist/leaflet.css';
  
  export let results = [];
  export let selectedParcel = null;
  
  let map = null;
  let geoJsonLayer = null;
  let selectedParcelLayer = null;
  let mapContainer;
  let isClickingFeature = false;
  let L; // Store Leaflet instance
  let baseLayers = {};
  
  // Initialize the map when the component mounts
  onMount(() => {
    if (typeof window === 'undefined') return;
    
    // Use an IIFE to handle async operations
    (async () => {
      // Dynamically import Leaflet to avoid SSR issues
      L = (await import('leaflet')).default;
      
      // Fix for default marker icons in Leaflet
      delete L.Icon.Default.prototype._getIconUrl;
      L.Icon.Default.mergeOptions({
        iconRetinaUrl: 'https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.7.1/images/marker-icon-2x.png',
        iconUrl: 'https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.7.1/images/marker-icon.png',
        shadowUrl: 'https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.7.1/images/marker-shadow.png',
      });
      
      // Initialize the map with proper options
      const center = L.latLng(46.603354, 1.888334);
      
      const options = {
        center: center,
        zoom: 13,
        zoomControl: true,
        preferCanvas: true // Better performance for large numbers of markers
      };
      
      // Add mobile-specific options
      if (L.Browser.mobile) {
        options.dragging = false;
        options.tap = false;
        options.tapTolerance = 15;
      }
      
      map = L.map(mapContainer, options);
      
      // Store map instance globally for access from App.svelte
      window.leafletMap = map;
      
      // Create base layers
      baseLayers = {
        'OpenStreetMap': L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
          attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
          maxZoom: 19,
        }),
        'Satellite': L.tileLayer('https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}', {
          attribution: 'Tiles &copy; Esri &mdash; Source: Esri, i-cubed, USDA, USGS, AEX, GeoEye, Getmapping, Aerogrid, IGN, IGP, UPR-EGP, and the GIS User Community',
          maxZoom: 19,
        })
      };
      
      // Add the default base layer
      baseLayers['Satellite'].addTo(map);
      
      // Add layer control
      L.control.layers(baseLayers, null, { position: 'topright' }).addTo(map);
      
      // Handle map click to clear selection when clicking outside features
      map.on('click', (e) => {
        if (!isClickingFeature) {
          selectedParcel = null;
          const event = new CustomEvent('parcel-select', { 
            detail: null,
            bubbles: true,
            cancelable: true,
            composed: true
          });
          window.dispatchEvent(event);
        }
        isClickingFeature = false;
      });
      
      // Update the map with initial results if any
      updateMapFeatures();
    })();
    
    // Clean up function
    return () => {
      if (map) {
        map.off();
        map.remove();
        map = null;
delete window.leafletMap;
      }
    };
  });
  
  // Watch for changes to selectedParcel and update the map
  $: if (map && selectedParcel) {
    updateSelectedParcel();
  }
  
  // Function to update map features when results change
  async function updateMapFeatures() {
    if (!map) return;
    
    // Dynamically import Leaflet
    const L = await import('leaflet');
    
    // Remove existing GeoJSON layer if it exists
    if (geoJsonLayer) {
      geoJsonLayer.remove();
    }
    
    if (results && results.length > 0) {
      // Create a feature collection from the results
      const featureCollection = {
        type: 'FeatureCollection',
        features: results
      };
      
      // Add the GeoJSON layer to the map
      geoJsonLayer = L.geoJSON(featureCollection, {
        style: styleFeature,
        onEachFeature: onEachFeature
      }).addTo(map);
      
      // Fit the map to the bounds of all features
      map.fitBounds(geoJsonLayer.getBounds());
    }
  }
  
  // Function to handle map click events
  function handleMapClick(e) {
    const feature = e.layer.feature;
    if (feature) {
      const event = new CustomEvent('parcel-select', { detail: feature });
      window.dispatchEvent(event);
    }
  }
  
  // Function to style the GeoJSON features
  function styleFeature(feature) {
    const isSelected = selectedParcel && selectedParcel.id === feature.id;
    
    return {
      fillColor: isSelected ? '#4CAF50' : '#3388ff',
      weight: isSelected ? 3 : 1,
      opacity: 1,
      color: isSelected ? '#fff' : '#3388ff',
      fillOpacity: isSelected ? 0.7 : 0.3
    };
  }
  
  // Function to get address from coordinates using Nominatim
  async function getAddressFromCoords(lat, lon) {
    try {
      const response = await fetch(`https://nominatim.openstreetmap.org/reverse?format=json&lat=${lat}&lon=${lon}&addressdetails=1`);
      const data = await response.json();
      
      if (data.address) {
        const { road, house_number, postcode, city, village, town } = data.address;
        const addressParts = [];
        
        if (road) {
          addressParts.push(house_number ? `${road} ${house_number}` : road);
        }
        if (postcode) addressParts.push(postcode);
        if (city || town || village) addressParts.push(city || town || village);
        
        return addressParts.join(', ');
      }
      return 'Address not available';
    } catch (error) {
      console.error('Error fetching address:', error);
      return 'Address not available';
    }
  }

  // Function to handle feature click
  function onEachFeature(feature, layer) {
    if (feature.properties) {
      const { surface_parcelle, section, number } = feature.properties;
      
      // Create a basic popup first (will be updated with address)
      const initialPopupContent = `
        <div style="min-width: 180px; max-width: 250px;">
          <div>Loading address...</div>
        </div>
      `;
      
      // Bind the initial popup
      layer.bindPopup(initialPopupContent);
      
      // Fetch address and update popup content asynchronously
      (async () => {
        try {
          const center = layer.getBounds().getCenter();
          const address = await getAddressFromCoords(center.lat, center.lng);
          
          const popupContent = `
            <div style="min-width: 180px; max-width: 250px;">
              <div style="margin-bottom: 6px; font-weight: 600; font-size: 1.05em;">
                ${address}
              </div>
              <div style="border-top: 1px solid #eee; padding-top: 4px; margin-bottom: 4px;">
                <div><strong>Parcel ID:</strong> ${feature.id || 'N/A'}</div>
                <div><strong>Area:</strong> ${surface_parcelle ? `${surface_parcelle} m²` : 'N/A'}</div>
                <div><strong>Section:</strong> ${section || 'N/A'}</div>
                ${number ? `<div><strong>Number:</strong> ${number}</div>` : ''}
              </div>
            </div>
          `;
          
          // Update the popup content
          layer.setPopupContent(popupContent);
          
          // If the popup is currently open, update it
          if (layer.isPopupOpen()) {
            layer.togglePopup();
            setTimeout(() => layer.togglePopup(), 10);
          }
        } catch (error) {
          console.error('Error updating popup:', error);
        }
      })();
      
      // Handle click on the feature
      layer.on('click', (e) => {
        // Prevent the map click handler from clearing the selection
        isClickingFeature = true;
        
        // Store the selected feature
        selectedParcel = feature;
        
        // Dispatch the event to parent components
        const event = new CustomEvent('parcel-select', { 
          detail: feature,
          bubbles: true,
          cancelable: true,
          composed: true
        });
        window.dispatchEvent(event);
        
        // Stop event propagation
        const leafletEvent = e.originalEvent?.view?.L?.DomEvent || e;
        if (leafletEvent.stopPropagation) {
          leafletEvent.stopPropagation();
        }
        
        // Open popup at the clicked location
        if (e.latlng) {
          layer.openPopup(e.latlng);
        } else {
          layer.openPopup();
        }
      });
    }
  }
  
  // Function to update the selected parcel layer
  async function updateSelectedParcel() {
    if (!map || !selectedParcel) {
      if (selectedParcelLayer) {
        selectedParcelLayer.remove();
        selectedParcelLayer = null;
      }
      return;
    }
    
    // Make sure L is available
    if (!L) L = (await import('leaflet')).default;
    
    // Remove existing selected parcel layer if it exists
    if (selectedParcelLayer) {
      selectedParcelLayer.remove();
      selectedParcelLayer = null;
    }
    
    try {
      // Create a style for the selected parcel
      const selectedParcelStyle = {
        color: '#FF0000',
        weight: 4,
        opacity: 1,
        fillColor: '#FF0000',
        fillOpacity: 0.2,
        dashArray: '10, 10'
      };
      
      // Create a feature collection with the selected parcel
      const featureCollection = {
        type: 'FeatureCollection',
        features: [selectedParcel]
      };
      
      // Add the selected parcel to the map with the custom style
      selectedParcelLayer = L.geoJSON(featureCollection, {
        style: selectedParcelStyle
      }).addTo(map);
      
      // Fit the map to the selected feature with some padding
      const bounds = selectedParcelLayer.getBounds();
      if (bounds.isValid()) {
        map.fitBounds(bounds, { padding: [50, 50] });
      }
      
      // Open the popup for the selected feature
      selectedParcelLayer.eachLayer((layer) => {
        if (layer.openPopup) {
          layer.openPopup();
        }
      });
      
    } catch (error) {
      console.error('Error updating selected parcel:', error);
    }
  }
  
  // Update the map when selectedParcel changes
  $: if (map && selectedParcel) {
    updateSelectedParcel();
  }
  
  // Clear selection when selectedParcel becomes null
  $: if (map && !selectedParcel && selectedParcelLayer) {
    selectedParcelLayer.remove();
    selectedParcelLayer = null;
  }
  
  // Update the map when results change
  $: if (map) {
    updateMapFeatures();
  }
  
  // Clean up when the component is destroyed
  onDestroy(() => {
    if (selectedParcelLayer) {
      selectedParcelLayer.remove();
      selectedParcelLayer = null;
    }
    
    if (map) {
      map.remove();
      map = null;
    }
  });
</script>

<div class="map" bind:this={mapContainer}>
  <!-- Map will be rendered here -->
</div>

<style>
  .map {
    width: 100%;
    height: 100%;
    min-height: 500px;
  }
  
  :global(.leaflet-popup-content) {
    margin: 10px 15px;
    font-size: 14px;
  }
  
  :global(.leaflet-popup-content h3) {
    margin-top: 0;
    margin-bottom: 5px;
    color: #2c3e50;
  }
  
  :global(.leaflet-popup-content p) {
    margin: 5px 0;
  }
</style>
