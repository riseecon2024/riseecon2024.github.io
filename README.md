# RBC_Financedwells-facilities

<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Mapbox Map with CSV and GeoJSON Layers</title>
  <link rel="stylesheet" href="https://api.mapbox.com/mapbox-gl-js/v3.1.2/mapbox-gl.css" />
  <script src="https://api.mapbox.com/mapbox-gl-js/v3.1.2/mapbox-gl.js"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/PapaParse/5.3.0/papaparse.min.js"></script>
  <script src="https://api.mapbox.com/mapbox-gl-js/plugins/mapbox-gl-geocoder/v4.7.0/mapbox-gl-geocoder.min.js"></script>
  <link href="https://api.mapbox.com/mapbox-gl-js/plugins/mapbox-gl-geocoder/v4.7.0/mapbox-gl-geocoder.css" rel="stylesheet" />
  <link href="https://fonts.googleapis.com/css2?family=Roboto:wght@400;700&display=swap" rel="stylesheet">

  <style>
    #map {
      height: 100vh;
      width: 100vw;
      position: relative;
    }

    #legend {
      position: absolute;
      background: white;
      padding: 10px;
      border-radius: 5px;
      box-shadow: 0 0 5px rgba(0, 0, 0, 0.2);
      bottom: 10px;
      left: 10px;
      font-family: 'Roboto', sans-serif;
      font-size: 12px;
      line-height: 1.5;
    }
 #legend H4 {
      margin: 0;
      padding-bottom: 5px;
      font-size: 14px; /* Adjust size if needed */
    }

    #legend div {
      margin-bottom: 5px; /* Adjust spacing between items */
    }
    .mapboxgl-popup {
      max-width: 400px;
      font: 12px/20px, 'Roboto', sans-serif;
    }

  </style>
</head>

<body>
  <div id="map"></div>
  <div id="legend">
    <h4>Legend</h4>
    <div id="legend-content"></div>
  </div>

  <script>
    var map;

    function initializeMap() {
      mapboxgl.accessToken = 'pk.eyJ1IjoiamJ1ZWxsMjAyNCIsImEiOiJjbHpsdXJoMnMwNnRyMnZwd3hseXBkZnJyIn0.LhvNTpWqA389pQTriiNB7A'; // Replace with your Mapbox access token

      map = new mapboxgl.Map({
        container: 'map',
        style: 'mapbox://styles/jbuell2024/clzlwunn0003701r83gbo30zp',
        center: [-119.0187, 37],
        zoom: 6
      });

      // Add controls
      map.addControl(new MapboxGeocoder({
        accessToken: mapboxgl.accessToken,
        mapboxgl: mapboxgl
      }));
      map.addControl(new mapboxgl.FullscreenControl());
      map.addControl(new mapboxgl.GeolocateControl({
        positionOptions: {
          enableHighAccuracy: true
        },
        trackUserLocation: true
      }));
      map.addControl(new mapboxgl.NavigationControl());

      map.on('load', function () {
        // Load and parse GeoJSON
        fetch('https://raw.githubusercontent.com/riseecon2024/riseecon2024/7bcbefa6c09bda7f69b5c672524e7292a983589f/allwells_reordered.geojson')
          .then(response => response.json())
          .then(geojsonData => {
            console.log('GeoJSON data loaded:', geojsonData); // Debugging line
            addGeoJsonLayer(geojsonData);
          })
          .catch(error => {
            console.error('Error loading GeoJSON:', error); // Debugging line
          });

        // Load and parse CSV
        Papa.parse('https://raw.githubusercontent.com/riseecon2024/riseecon2024/8c9e2abbad3476b4b2a1b3c1e82ef3ad6c7d5fdf/RBC-Funded-Facilities_tractjoined_06-25-2024.csv', {
          download: true,
          header: true,
          complete: function (results) {
            // Convert CSV to GeoJSON
            const csvGeojson = {
              type: 'FeatureCollection',
              features: results.data.map(row => ({
                type: 'Feature',
                geometry: {
                  type: 'Point',
                  coordinates: [parseFloat(row.LONGITUDE), parseFloat(row.LATITUDE)]
                },
                properties: {
                  'GHG QUANTITY': parseFloat(row['GHG QUANTITY']),
                  'FACILITY NAME': row['FACILITY NAME'],
                  'PARENT COMPANIES': row['PARENT COMPANIES'],
                  'NORMALIZED_EMISSIONS2': parseFloat(row['NORMALIZED_EMISSIONS2'])
                }
              }))
            };

            addCsvLayer(csvGeojson);
          }
        });
      });
    }

    function addGeoJsonLayer(geojsonData) {
      map.addSource('geojson-points', {
        type: 'geojson',
        data: geojsonData
      });

      map.addLayer({
        id: 'geojson-points-layer',
        type: 'circle',
        source: 'geojson-points',
        paint: {
          'circle-radius': [
            'case',
            ['==', ['get', 'WellStatus'], 'Active'], 8,
            ['==', ['get', 'WellStatus'], 'Idle'], 6,
            ['==', ['get', 'WellStatus'], 'New'], 4,
            ['==', ['get', 'WellStatus'], 'Plugged'], 2,
            3
          ],
          'circle-stroke-width': 0.2,
          'circle-stroke-color': 'black',
          'circle-stroke-opacity': 0.5,
          'circle-color': [
            'case',
            ['==', ['get', 'WellStatus'], 'Active'], '#571b60',
            ['==', ['get', 'WellStatus'], 'Idle'], '#8d6593',
            ['==', ['get', 'WellStatus'], 'New'], '#c4b0c7',
            ['==', ['get', 'WellStatus'], 'Plugged'], '#fafafa',
            'black'
          ],
          'circle-opacity': 1
        }
      });

      // Initialize legend
      updateLegend();
    }

    function addCsvLayer(geojsonData) {
      map.addSource('csv-points', {
        type: 'geojson',
        data: geojsonData
      });

      map.addLayer({
        id: 'csv-points-layer',
        type: 'circle',
        source: 'csv-points',
        paint: {
          'circle-radius': [
            'interpolate',
            ['linear'],
            ['get', 'GHG QUANTITY'],
            0.001, 2, // Min value and min radius
            2596721, 15,
            6038424, 20 // Max value and max radius
          ],
          'circle-stroke-width': 0.5,
          'circle-stroke-color': 'black',
          'circle-stroke-opacity': 0.5,
          'circle-color': '#ff0000',
          'circle-opacity': 1
        }
      });

 // Initialize the popup for hover (transient popup)
const hoverPopup = new mapboxgl.Popup({
  closeButton: false,
  closeOnClick: false
});

// Add hover events for popups
// Add hover events for popups
map.on('mousemove', 'csv-points-layer', function (e) {
  if (e.features.length > 0) {
    const feature = e.features[0];
    const ghgQuantityFormatted = feature.properties['GHG QUANTITY'].toLocaleString();
    // Set the hover popup content
    hoverPopup.setLngLat(e.lngLat)
      .setHTML(`<strong>Facility Name:</strong> ${feature.properties['FACILITY NAME']}<br> <strong>Owner: </strong>${feature.properties['PARENT COMPANIES']}<br><br> <i> This facility emitted <strong>${ghgQuantityFormatted}</strong> metric tons of CO2 greenhouse gas emissions in 2021.`)
      .addTo(map);
  }
});

map.on('mouseleave', 'csv-points-layer', function () {
  hoverPopup.remove();
});

// Add click event for popups (sticky popup)
let clickPopup = null;

map.on('click', 'csv-points-layer', function (e) {
  if (e.features.length > 0) {
    const feature = e.features[0];
    const ghgQuantityFormatted = feature.properties['GHG QUANTITY'].toLocaleString();

    // Remove the existing click popup if any
    if (clickPopup) {
      clickPopup.remove();
    }

    // Create a new click popup
    clickPopup = new mapboxgl.Popup({
      closeButton: true,
      closeOnClick: true
    });

    // Set the click popup content
    clickPopup.setLngLat(e.lngLat)
      .setHTML(`<strong>Facility Name:</strong> ${feature.properties['FACILITY NAME']}<br> <strong>Owner: </strong>${feature.properties['PARENT COMPANIES']}<br><br> <i> This facility emitted <strong>${ghgQuantityFormatted}</strong> metric tons of CO2 greenhouse gas emissions in 2021.`)
      .addTo(map);
  }
});



      map.on('mouseleave', 'csv-points-layer', function () {
        popup.remove();
      });

      // Update 
      updateLegend();

      // Update the legend whenever the data changes
      map.on('data', function (e) {
        if (e.dataType === 'source' && e.sourceId === 'csv-points') {
          updateLegend();
        }
      });
    }

    function updateLegend() {
      const statusCounts = {
        'Active': 0,
        'Idle': 0,
        'New': 0,
        'Plugged': 0 
      };

      // Get feature count by status for wells layer
      const wellFeatures = map.querySourceFeatures('geojson-points');
      wellFeatures.forEach(feature => {
        const status = feature.properties.WellStatus;
        if (statusCounts[status] !== undefined) {
          statusCounts[status]++;
        }
      });

      // Get feature count for facilities layer
      const facilityFeatures = map.querySourceFeatures('csv-points');
      const facilityCount = facilityFeatures.length;

      document.getElementById('legend-content').innerHTML = `
   <div>
   An<span style="background-color: #fd7777; width: 15px; height: 10px; display: inline-block; margin-left: 5px; margin-right: 5px;"></span>overlay represents a majority-minority census tract.
     </div>
  <div>
    <span style="background-color: #ff0000; border: 0.2px solid #000000; width: 10px; height: 10px; display: inline-block; border-radius: 50%; margin-right: 5px;"></span>
    Oil and Gas Facility Count: ${facilityCount} 
    </div>
<div>
<span style="background-color: #571b60; border: 0.2px solid #000000;width: 10px; height: 10px; display: inline-block; margin-right: 5px;"></span>
    Active well: ${statusCounts['Active']}
  </div>
  <div>
    <span style="background-color: #8d6593; border: 0.2px solid #000000;width: 10px; height: 10px; display: inline-block; margin-right: 5px;"></span>
    Idle well: ${statusCounts['Idle']}
  </div>
  <div>
    <span style="background-color: #c4b0c7; border: 0.2px solid #000000; width: 10px; height: 10px; display: inline-block; margin-right: 5px;"></span>
    New well: ${statusCounts['New']}
  </div>
  <div>
    <span style="background-color: #fafafa; border: 0.2px solid #000000; width: 10px; height: 10px; display: inline-block; margin-right: 5px;"></span>
    Plugged well: ${statusCounts['Plugged']}
  </div>
  
 `;

    }

    // Initialize map
    initializeMap();
  </script>
</body>

</html>
