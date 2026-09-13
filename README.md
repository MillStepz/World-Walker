 <!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>World Walker</title>
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <link
    rel="stylesheet"
    href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css"
  />
  <style>
    body {
      margin: 0;
      font-family: system-ui, -apple-system, BlinkMacSystemFont, sans-serif;
      display: flex;
      flex-direction: column;
      height: 100vh;
      background: #f5f5f5;
    }
    #header {
      padding: 12px 16px;
      background: #222;
      color: white;
      font-size: 1.4rem;
      font-weight: 600;
      letter-spacing: 0.5px;
    }
    #controls {
      padding: 10px;
      background: #fff;
      border-bottom: 1px solid #ddd;
      display: flex;
      flex-wrap: wrap;
      gap: 10px;
      align-items: center;
    }
    #map {
      flex: 1;
    }
    input[type="number"] {
      width: 120px;
      padding: 4px 6px;
    }
    button {
      padding: 6px 10px;
      border: none;
      background: #0077ff;
      color: white;
      border-radius: 4px;
      cursor: pointer;
    }
    button:disabled {
      background: #aaa;
      cursor: default;
    }
    .info {
      font-size: 0.9rem;
    }
  </style>
</head>
<body>
  <div id="header">🌍 World Walker</div>

  <div id="controls">
    <button id="locateBtn">Use my GPS</button>
    <span class="info">Click anywhere on the map to set Point A and Point B.</span>
    <label>
      Daily steps:
      <input type="number" id="stepsInput" min="0" value="8000" />
    </label>
    <button id="updateProgressBtn" disabled>Update progress</button>
    <span id="status" class="info"></span>
  </div>

  <div id="map"></div>

  <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
  <script>
    const map = L.map('map').setView([20, 0], 2);

    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
      maxZoom: 19,
      attribution: '&copy; OpenStreetMap contributors'
    }).addTo(map);

    const locateBtn = document.getElementById('locateBtn');
    const stepsInput = document.getElementById('stepsInput');
    const updateProgressBtn = document.getElementById('updateProgressBtn');
    const statusEl = document.getElementById('status');

    let pointA = null;
    let pointB = null;
    let markerA = null;
    let markerB = null;
    let routeLine = null;
    let progressMarker = null;

    const STEP_LENGTH_METERS = 0.78;

    locateBtn.addEventListener('click', () => {
      if (!navigator.geolocation) {
        statusEl.textContent = 'Geolocation not supported.';
        return;
      }
      navigator.geolocation.getCurrentPosition(
        (pos) => {
          const { latitude, longitude } = pos.coords;
          map.setView([latitude, longitude], 12);
          L.circleMarker([latitude, longitude], {
            radius: 6,
            color: '#0077ff',
            fillColor: '#0077ff',
            fillOpacity: 0.8
          }).addTo(map).bindPopup('You are here').openPopup();
        },
        (err) => {
          statusEl.textContent = 'Location error: ' + err.message;
        }
      );
    });

    map.on('click', (e) => {
      const { lat, lng } = e.latlng;

      if (!pointA) {
        pointA = L.latLng(lat, lng);
        if (markerA) map.removeLayer(markerA);
        markerA = L.marker(pointA, { draggable: true }).addTo(map)
          .bindPopup('Point A').openPopup();
        markerA.on('dragend', () => {
          pointA = markerA.getLatLng();
          drawRoute();
        });
      } else if (!pointB) {
        pointB = L.latLng(lat, lng);
        if (markerB) map.removeLayer(markerB);
        markerB = L.marker(pointB, { draggable: true }).addTo(map)
          .bindPopup('Point B').openPopup();
        markerB.on('dragend', () => {
          pointB = markerB.getLatLng();
          drawRoute();
        });
      } else {
        resetRoute();
        statusEl.textContent = 'Route reset. Choose new A and B.';
      }

      drawRoute();
    });

    function resetRoute() {
      pointA = null;
      pointB = null;
      if (markerA) map.removeLayer(markerA);
      if (markerB) map.removeLayer(markerB);
      if (routeLine) map.removeLayer(routeLine);
      if (progressMarker) map.removeLayer(progressMarker);
      markerA = markerB = routeLine = progressMarker = null;
      updateProgressBtn.disabled = true;
    }

    function drawRoute() {
      if (routeLine) map.removeLayer(routeLine);
      if (progressMarker) map.removeLayer(progressMarker);

      if (pointA && pointB) {
        routeLine = L.polyline([pointA, pointB], {
          color: '#ff5500',
          weight: 4
        }).addTo(map);
        map.fitBounds(routeLine.getBounds());
        updateProgressBtn.disabled = false;

        const distanceKm = pointA.distanceTo(pointB) / 1000;
        statusEl.textContent =
          `Route distance: ${distanceKm.toFixed(2)} km`;
      }
    }

    updateProgressBtn.addEventListener('click', () => {
      if (!pointA || !pointB) {
        statusEl.textContent = 'Set both points first.';
        return;
      }

      const steps = Number(stepsInput.value);
      if (isNaN(steps) || steps <= 0) {
        statusEl.textContent = 'Enter valid steps.';
        return;
      }

      const totalDistanceMeters = pointA.distanceTo(pointB);
      const walkedMeters = steps * STEP_LENGTH_METERS;
      const ratio = Math.min(walkedMeters / totalDistanceMeters, 1);

      const lat = pointA.lat + (pointB.lat - pointA.lat) * ratio;
      const lng = pointA.lng + (pointB.lng - pointA.lng) * ratio;
      const progressLatLng = L.latLng(lat, lng);

      if (progressMarker) map.removeLayer(progressMarker);
      progressMarker = L.circleMarker(progressLatLng, {
        radius: 7,
        color: '#00c853',
        fillColor: '#00c853',
        fillOpacity: 0.9
      }).addTo(map).bindPopup('Your World Walker progress').openPopup();

      const walkedKm = walkedMeters / 1000;
      const totalKm = totalDistanceMeters / 1000;
      statusEl.textContent =
        `You’ve walked ${walkedKm.toFixed(2)} km of ${totalKm.toFixed(2)} km (${(ratio * 100).toFixed(1)}%).`;
    });
  </script>
</body>
</html>
