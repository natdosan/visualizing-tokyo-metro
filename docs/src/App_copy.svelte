<script>
  import { onMount } from 'svelte';
  import mapboxgl from 'mapbox-gl';
  import { arcgisToGeoJSON } from 'arcgis-to-geojson-utils';

  mapboxgl.accessToken = 'pk.eyJ1IjoibmF0ZG9zYW4iLCJhIjoiY2xza3huODg4MDh1ZzJpcDVoOTJ1eWFqayJ9.hvQnPf9rwTCV4aok1j7xJA';

  let mapTokyo;
  let isRailwayVisible = false;
  let areStationsVisible = false; // Initial state
  let isChoroplethVisible = false; 
  let stationMarkers = [];
  let geojson;

  onMount(() => {
    // define Tokyo map
    mapTokyo = new mapboxgl.Map({
      container: 'map-tokyo',
      style: 'mapbox://styles/mapbox/navigation-night-v1',
      center: [139.7528, 35.6852], // 35.6852° N, 139.7528° E
      zoom: 12
    });

    // Pop up for stations
    let popup = new mapboxgl.Popup({
      closeButton: false,
      closeOnClick: false
    });

    // Load railways
    mapTokyo.on('load', async() => {
      mapTokyo.addSource('tokyo-railways', {
        type: 'geojson',
        data: '/N02-19_RailroadSection.geojson'
      });

      // add the actual layer
      mapTokyo.addLayer({
        id: 'tokyo-railways-layer',
        type: 'line',
        source: 'tokyo-railways',
        layout: {},
        paint: {
          'line-color': '#ADD8E6',
          'line-width': 2,
          'line-opacity': isRailwayVisible ? 1 : 0
        }
      });

      // Hover metric: Line name
      mapTokyo.on('mousemove', 'tokyo-railways-layer', (e) => {
        if (e.features.length > 0) {
          const feature = e.features[0];
          popup.setLngLat(e.lngLat)
                .setText(feature.properties.路線名)
                .addTo(mapTokyo);
        }
      });
      // Remove line name when hover off
      mapTokyo.on('mouseleave', 'tokyo-railways-layer', () => {
        popup.remove();
      });

    // Load administrative boundaries
    mapTokyo.addSource('tokyo-boundaries', {
      type: 'geojson',
      data: '/tokyo23_population.geojson'
    });

    // add administrative boundary layer
    mapTokyo.addLayer({
      id: 'tokyo-administrative-boundaries',
      type: 'line',
      source: 'tokyo-boundaries',
      layout: {},
      paint: {
        'line-color': '#FFFFFF',
        'line-width': 2,
      }
    });

    // Load Population Density Choropleth source
    mapTokyo.addSource('tokyo-population-density', {
        type: 'geojson',
        data: '/tokyo23_population.geojson'
    });
      
    // add the population density layer
    mapTokyo.addLayer({
      id: 'tokyo-population-density',
      type: 'fill',
      source: 'tokyo-population-density',
      paint: {
        'fill-color': [
          'interpolate',
          ['linear'],
          ['get', 'population'],
          0, '#ffffff',
          100000, '#614144',
          300000, '#802A37',
          500000, '#A51C2A',
          700000, '#DE0D0D',
          1000000, '#FF9641'
        ],
        'fill-opacity': 0.5
      }
    });

    // Show/hide population numbers on hover
    mapTokyo.on('mousemove', 'tokyo-population-density', (e) => {
      // console.log(e.features[0].properties) they are populated 
      if (e.features.length > 0) {
        const feature = e.features[0];
        const population = feature.properties['population'];
        // console.log(population) the population numbers are showing in real time
        mapTokyo.getCanvas().style.cursor = 'pointer';
        popup.setLngLat(e.lngLat)
            .setHTML(`<h3>Population: ${population}</h3>`)
            .addTo(mapTokyo);
      }
    });

    // Remove popup on mouseleave
    mapTokyo.on('mouseleave', 'tokyo-population-density', () => {
      mapTokyo.getCanvas().style.cursor = '';
      popup.remove();
    });

    // Calculate opacity based on ridership to population ratio
    function calculateOpacity(ridership, population) {
      const ratio = ridership / population;
      const minOpacity = 0.4; // Ensure markers are always somewhat visible
      const maxOpacity = 1.0;
      let opacity = ratio * (maxOpacity - minOpacity) + minOpacity;
      opacity = Math.min(Math.max(opacity, minOpacity), maxOpacity); // Clamp between min and max
      return opacity;
    }

    // Load stations data 
    const response = await fetch('/stations.json'); 
    const stationsData = await response.json(); 

    Object.values(stationsData).flat().forEach(station => {
      const opacity = calculateOpacity(station.ridership, station.population);
      const el = document.createElement('div');
      el.className = 'station-marker'; // Add CSS for this class as needed
      el.style.opacity = opacity;
      el.style.backgroundColor = `rgba(255, 0, 0, ${opacity})`; // Dynamic red color based on opacity
      el.style.opacity = opacity.toString(); // Explicitly converting to string

      const marker = new mapboxgl.Marker(el)
        .setLngLat(station.coordinates)
        .setPopup(new mapboxgl.Popup().setText(`${station.name}: Daily Ridership - ${station.ridership}`))
        .addTo(mapTokyo);
      });
    });
  });

  // Reactive Elements
  function toggleRailwayVisibility() {
    isRailwayVisible = !isRailwayVisible;
    mapTokyo.setPaintProperty('tokyo-railways-layer', 'line-opacity', isRailwayVisible ? 1 : 0);
  }
  function toggleStationVisibility() {
    areStationsVisible = !areStationsVisible;
    stationMarkers.forEach(marker => {
      areStationsVisible ? marker.addTo(mapTokyo) : marker.remove();
  });
  }
  // Toggle choropleth visibility
  function toggleChoroplethVisibility() {
    isChoroplethVisible = !isChoroplethVisible;
    mapTokyo.setLayoutProperty('tokyo-population-density', 'visibility', isChoroplethVisible ? 'visible' : 'none');
  }
</script>

<style>
  #map-tokyo {
    position: relative; /* Needed for absolute positioning of children */
    height: 100vh;
  }
  .toggle-button {
    position: absolute; /* Position the button over the map */
    top: 20px; /* Distance from the top of the map container */
    right: 20px; /* Distance from the right side of the map container */
    z-index: 10; /* Ensure the button is above map layers and controls */
    padding: 0.5rem 1rem;
    background-color: white; /* Button background for visibility */
    border: none; /* Optional: for style */
    cursor: pointer; /* Optional: change cursor on hover */
    border-radius: 5px; /* Optional: for rounded corners */
    box-shadow: 0 2px 4px rgba(0,0,0,0.2); /* Optional: for a subtle shadow */
  }
  h1 {
    text-align: center;
    position: absolute;
    top: 10px;
    left: 50%;
    transform: translateX(-50%);
    z-index: 2; /* Ensure title is above map and button */
    background-color: rgba(255, 255, 255, 0.8);
    padding: 8px 16px;
    border-radius: 4px;
  }
</style>


<div id="map-tokyo">
  <h1>Live Traffic Congestion and the 10 Most Active Metro Lines in Tokyo</h1>
  <button class="toggle-button" on:click={toggleRailwayVisibility}>
    {isRailwayVisible ? 'Hide Railway Lines' : 'Show Railway Lines'}
  </button>
  <button class="toggle-button" on:click={toggleStationVisibility} style="top: 80px;">
    {areStationsVisible ? 'Hide Stations' : 'Show Stations'}
  </button>
  <button class="toggle-button" on:click={toggleChoroplethVisibility} style="top: 140px;">
    {isChoroplethVisible ? 'Hide Choropleth' : 'Show Choropleth'}
  </button>
</div>
