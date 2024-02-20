<script>
  import { onMount } from 'svelte';
  import mapboxgl from 'mapbox-gl';

  mapboxgl.accessToken = 'pk.eyJ1IjoibmF0ZG9zYW4iLCJhIjoiY2xza3huODg4MDh1ZzJpcDVoOTJ1eWFqayJ9.hvQnPf9rwTCV4aok1j7xJA';

  let mapTokyo;
  let isRailwayVisible = false;
  let areStationsVisible = false; // Initial state
  let stationMarkers = [];

  onMount(() => {
    // Tokyo map
    mapTokyo = new mapboxgl.Map({
      container: 'map-tokyo',
      style: 'mapbox://styles/mapbox/navigation-night-v1',
      center: [139.7528, 35.6852], // 35.6852° N, 139.7528° E
      zoom: 12
    });

    const popup = new mapboxgl.Popup({
      closeButton: false,
      closeOnClick: false
    });

    // Add railways
    mapTokyo.on('load', async() => {
      mapTokyo.addSource('tokyo-railways', {
        type: 'geojson',
        data: '/N02-19_RailroadSection.geojson'
      });

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

      mapTokyo.on('mousemove', 'tokyo-railways-layer', (e) => {
        if (e.features.length > 0) {
          const feature = e.features[0];
          popup.setLngLat(e.lngLat)
               .setText(feature.properties.路線名)
               .addTo(mapTokyo);
        }
      });

      mapTokyo.on('mouseleave', 'tokyo-railways-layer', () => {
        popup.remove();
      });

      // Calculate opacity based on ridership to population ratio
      function calculateOpacity(ridership, population) {
        const ratio = ridership / population;
        const minOpacity = 0.4; // Ensure markers are always somewhat visible
        const maxOpacity = 1.0;
        let opacity = ratio * (maxOpacity - minOpacity) + minOpacity;
        opacity = Math.min(Math.max(opacity, minOpacity), maxOpacity); // Clamp between min and max
        console.log(`Ridership: ${ridership}, Population: ${population}, Opacity: ${opacity}`);
        return opacity;
      }

      const response = await fetch('/stations.json'); 
      const stationsData = await response.json(); 
      console.log(stationsData[0]); // Log the first station object to verify structure


      Object.values(stationsData).flat().forEach(station => {
        const opacity = calculateOpacity(Number(station.ridership), Number(station.population));
        console.log(opacity); // Add this inside the forEach loop after calculating opacity
        const el = document.createElement('div');
        el.className = 'station-marker'; // Add CSS for this class as needed
        el.style.opacity = .5;
        el.style.backgroundColor = `rgba(255, 0, 0, ${opacity})`; // Dynamic red color based on opacity
        el.style.opacity = opacity.toString(); // Explicitly converting to string

        const marker = new mapboxgl.Marker(el)
          .setLngLat(station.coordinates)
          .setPopup(new mapboxgl.Popup().setText(`${station.name}: Daily Ridership - ${station.ridership} \n ${station.name}: Ward Population - ${station.population}`))
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

  .map-title {
    top: 20px; /* Adjust as needed */
    left: 20px; /* Adjust as needed */
    background-color: rgba(255, 255, 255, 0.8); /* Semi-transparent background */
    padding: 0.5rem 1rem;
    border-radius: 5px;
    box-shadow: 0 2px 4px rgba(0,0,0,0.2);
  }

  h1 {
    text-align: center;
    position: absolute;
    top: 10px;
    left: 50%;
    transform: translateX(-50%);
    z-index:2; /* Ensure titile is above map and buttion */
    padding: 8px 16px;
    border-radius: 4px;
  }
</style>

<h1 class='h1'> Live Traffic Congestion and the 10 Most Active Metro Lines in Tokyo </h1>

<div id="map-tokyo">
  <div class="h1">Tokyo Live Roads with Metro Lines</div>
  <button class="toggle-button" on:click={toggleRailwayVisibility}>
    {isRailwayVisible ? 'Hide Railway Lines' : 'Show Railway Lines'}
  </button>
  <button class="toggle-button" on:click={toggleStationVisibility} style="top: 80px;">
    {areStationsVisible ? 'Hide Stations' : 'Show Stations'}
  </button>
</div>