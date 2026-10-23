<!DOCTYPE html>
<html lang="ne">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>लाइभ लोकेसन ट्र्याकर</title>
    
    <!-- Leaflet CSS (नक्साको लागि) -->
    <link rel="stylesheet" href="https://unpkg.com" />
    
    <style>
        * { box-sizing: border-box; margin: 0; padding: 0; font-family: Arial, sans-serif; }
        body { background-color: #f4f4f9; padding: 15px; display: flex; flex-direction: column; height: 100vh; }
        .controls { display: flex; gap: 10px; margin-bottom: 15px; justify-content: center; }
        .btn { padding: 12px 24px; border: none; border-radius: 5px; font-size: 16px; font-weight: bold; cursor: pointer; color: white; transition: 0.3s; }
        .btn-start { background-color: #28a745; }
        .btn-start:disabled { background-color: #a7dfb6; cursor: not-allowed; }
        .btn-stop { background-color: #dc3545; }
        .btn-stop:disabled { background-color: #f5c2c7; cursor: not-allowed; }
        #map { flex-grow: 1; width: 100%; border-radius: 10px; box-shadow: 0 4px 12px rgba(0,0,0,0.15); min-height: 300px; }
        .data-panel { background: white; padding: 15px; border-radius: 8px; margin-top: 15px; box-shadow: 0 4px 12px rgba(0,0,0,0.1); display: grid; grid-template-columns: 1fr 1fr; gap: 10px; font-size: 14px; }
        .data-item { padding: 8px; background: #f8f9fa; border-radius: 4px; border-left: 4px solid #007bff; }
    </style>
</head>
<body>

    <!-- नियन्त्रण बटनहरू -->
    <div class="controls">
        <button id="start-btn" class="btn btn-start" onclick="startTracking()">ट्र्याकिङ सुरु गर्नुहोस्</button>
        <button id="stop-btn" class="btn btn-stop" onclick="stopTracking()" disabled>बन्द गर्नुहोस्</button>
    </div>

    <!-- नक्सा देखिने ठाउँ -->
    <div id="map"></div>

    <!-- डाटा प्यानल -->
    <div class="data-panel">
        <div class="data-item" id="lat">अक्षांश (Latitude): 0.00000</div>
        <div class="data-item" id="lon">देशान्तर (Longitude): 0.00000</div>
        <div class="data-item" id="accuracy">सटीकता (Accuracy): 0 m</div>
        <div class="data-item" id="speed">गति (Speed): 0 km/h</div>
    </div>

    <!-- Leaflet JS (नक्साको लाइब्रेरी) -->
    <script src="https://unpkg.com"></script>
    
    <script>
        var map, marker, circle;
        var watchId = null;

        // नक्सा सुरु गर्ने (Default View: Nepal)
        map = L.map('map').setView([28.3949, 84.1240], 7);

        // नक्साकोBackground Layer सेट गर्ने (OpenStreetMap)
        L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
            maxZoom: 19,
            attribution: '© OpenStreetMap contributors'
        }).addTo(map);

        // ट्र्याकिङ सुरु गर्ने फंक्शन
        function startTracking() {
            if (!navigator.geolocation) {
                alert("तपाईंको ब्राउजरले GPS सपोर्ट गर्दैन।");
                return;
            }

            document.getElementById("start-btn").disabled = true;
            document.getElementById("stop-btn").disabled = false;

            // रियल-टाइम लोकेसन ट्र्याक गर्ने (watchPosition)
            watchId = navigator.geolocation.watchPosition(
                function(position) {
                    var lat = position.coords.latitude;
                    var lon = position.coords.longitude;
                    var accuracy = position.coords.accuracy;
                    var speed = position.coords.speed ? (position.coords.speed * 3.6).toFixed(1) : 0; // m/s लाई km/h मा बदलेको

                    // UI मा डाटा अपडेट गर्ने
                    document.getElementById("lat").innerText = "अक्षांश (Latitude): " + lat.toFixed(5);
                    document.getElementById("lon").innerText = "देशान्तर (Longitude): " + lon.toFixed(5);
                    document.getElementById("accuracy").innerText = "सटीकता (Accuracy): " + accuracy.toFixed(1) + " m";
                    document.getElementById("speed").innerText = "गति (Speed): " + speed + " km/h";

                    var currentPos = [lat, lon];

                    // यदि पहिलो पटक हो भने मार्कर र एक्युरेसी सर्कल बनाउने
                    if (!marker) {
                        marker = L.marker(currentPos).addTo(map);
                        circle = L.circle(currentPos, { radius: accuracy, color: '#007bff', fillColor: '#007bff', fillOpacity: 0.15 }).addTo(map);
                        map.setView(currentPos, 16); // नक्सालाई लोकेसनमा जुम गर्ने
                    } else {
                        // पुराना मार्करलाई नयाँ लोकेसनमा सार्ने
                        marker.setLatLng(currentPos);
                        circle.setLatLng(currentPos);
                        circle.setRadius(accuracy);
                        map.panTo(currentPos); // नक्सालाई बिस्तारै नयाँ लोकेसनमा सार्ने
                    }
                },
                function(error) {
                    console.error("GPS समस्या:", error);
                    if(error.code == 1) {
                        alert("कृपया आफ्नो ब्राउजरमा लोकेसन पर्मिसन (Location Permission) अन गर्नुहोस्।");
                    } else {
                        alert("GPS सिग्नल प्राप्त गर्न सकिएन।");
                    }
                    stopTracking();
                },
                {
                    enableHighAccuracy: true, // मोबाइलको सक्कल GPS चिप प्रयोग गर्ने (Highly Accurate)
                    timeout: 10000,           // १० सेकेन्ड सम्म प्रयास गर्ने
                    maximumAge: 0             // क्यास डाटा प्रयोग नगर्ने, फ्रेस डाटा लिने
                }
            );
        }

        // ट्र्याकिङ बन्द गर्ने फंक्शन
        function stopTracking() {
            if (watchId !== null) {
                navigator.geolocation.clearWatch(watchId);
                watchId = null;
            }
            document.getElementById("start-btn").disabled = false;
            document.getElementById("stop-btn").disabled = true;
        }
    </script>
</body>
</html>
