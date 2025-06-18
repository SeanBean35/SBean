<!DOCTYPE html>
<html>
<head>
    <title>Hollywood Public Restrooms</title>
    <style>
        /* Basic CSS for the map and controls */
        body {
            font-family: Arial, sans-serif;
            margin: 0;
            padding: 0;
            display: flex;
            flex-direction: column;
            align-items: center;
            min-height: 100vh;
        }
        #map {
            height: 600px;
            width: 80%;
            border: 1px solid #ccc;
            margin-top: 20px;
        }
        #info {
            margin-top: 20px;
            padding: 10px;
            border: 1px solid #eee;
            background-color: #f9f9f9;
            width: 80%;
            text-align: center;
        }
        .restroom-list {
            margin-top: 20px;
            width: 80%;
            max-height: 300px;
            overflow-y: auto;
            border: 1px solid #eee;
            padding: 10px;
            list-style: none;
        }
        .restroom-list li {
            padding: 5px 0;
            border-bottom: 1px dashed #eee;
        }
        .restroom-list li:last-child {
            border-bottom: none;
        }
    </style>
</head>
<body>
    <h1>Hollywood Public Restrooms</h1>
    <div id="info">
        <p>Getting your location and finding restrooms...</p>
    </div>
    <div id="map"></div>
    <ul class="restroom-list" id="restroom-details-list">
        <h2>Restrooms Near Hollywood</h2>
        </ul>

    <script>
        let map;
        let userLocation;
        const infoWindow = new google.maps.InfoWindow();

        // Hollywood approximate center coordinates
        const hollywoodCenter = { lat: 34.0928, lng: -118.3587 };
        const hollywoodRadiusKm = 5; // Radius to consider restrooms "in Hollywood"

        // Haversine formula to calculate distance between two lat/lng points
        function haversineDistance(coords1, coords2) {
            const R = 6371; // Radius of Earth in kilometers
            const lat1 = coords1.lat * Math.PI / 180;
            const lon1 = coords1.lng * Math.PI / 180;
            const lat2 = coords2.lat * Math.PI / 180;
            const lon2 = coords2.lng * Math.PI / 180;

            const dLat = lat2 - lat1;
            const dLon = lon2 - lon1;

            const a = Math.sin(dLat / 2) * Math.sin(dLat / 2) +
                      Math.cos(lat1) * Math.cos(lat2) *
                      Math.sin(dLon / 2) * Math.sin(dLon / 2);
            const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
            const d = R * c; // Distance in km
            return d;
        }

        async function initMap() {
            const { Map } = await google.maps.importLibrary("maps");
            const { AdvancedMarkerElement } = await google.maps.importLibrary("marker");

            map = new Map(document.getElementById("map"), {
                zoom: 13,
                center: hollywoodCenter, // Initial center, will update with user location
                mapId: "YOUR_MAP_ID" // You can create a Map ID in Google Cloud Console for custom styling
            });

            // Try HTML5 geolocation to get user's current location
            if (navigator.geolocation) {
                navigator.geolocation.getCurrentPosition(
                    (position) => {
                        userLocation = {
                            lat: position.coords.latitude,
                            lng: position.coords.longitude,
                        };
                        map.setCenter(userLocation);
                        map.setZoom(14); // Zoom in closer to user
                        document.getElementById("info").innerText = "Your location found. Finding restrooms...";

                        // Add a marker for the user's location
                        new AdvancedMarkerElement({
                            map: map,
                            position: userLocation,
                            title: "Your Location",
                            content: new PinElement({
                                background: "#4CAF50",
                                borderColor: "#2E7D32",
                                glyphColor: "#fff",
                                scale: 1.2
                            }).element,
                        });

                        fetchRestroomsAndDisplay(AdvancedMarkerElement);
                    },
                    () => {
                        handleLocationError(true);
                        fetchRestroomsAndDisplay(AdvancedMarkerElement); // Fetch anyway, but without user location
                    }
                );
            } else {
                // Browser doesn't support Geolocation
                handleLocationError(false);
                fetchRestroomsAndDisplay(AdvancedMarkerElement); // Fetch anyway, but without user location
            }
        }

        function handleLocationError(browserHasGeolocation) {
            const infoDiv = document.getElementById("info");
            infoDiv.innerText = browserHasGeolocation
                ? "Error: The Geolocation service failed or you denied permission. Showing restrooms in Hollywood."
                : "Error: Your browser doesn't support Geolocation. Showing restrooms in Hollywood.";
        }

        async function fetchRestroomsAndDisplay(AdvancedMarkerElement) {
            const restroomsListElement = document.getElementById("restroom-details-list");
            restroomsListElement.innerHTML = '<h2>Restrooms Near Hollywood</h2>'; // Clear previous list

            // Fetch restrooms from Refuge Restrooms API
            // The API supports searching by latitude/longitude, but not directly by "Hollywood"
            // We'll search broadly and then filter by distance to Hollywood.
            // For a production app, you might want a more sophisticated approach or a local dataset.
            const response = await fetch('https://www.refugerestrooms.org/api/v1/restrooms/search?lat=34.0928&lon=-118.3587&per_page=100'); // Search around Hollywood
            const restrooms = await response.json();

            const hollywoodRestrooms = restrooms.filter(restroom => {
                const restroomCoords = { lat: restroom.latitude, lng: restroom.longitude };
                const distanceToHollywoodCenter = haversineDistance(hollywoodCenter, restroomCoords);
                return distanceToHollywoodCenter <= hollywoodRadiusKm;
            });

            if (hollywoodRestrooms.length === 0) {
                document.getElementById("info").innerText = "No public restrooms found in the Hollywood area.";
            } else {
                document.getElementById("info").innerText = `Found ${hollywoodRestrooms.length} public restrooms in the Hollywood area.`;
            }

            hollywoodRestrooms.forEach(restroom => {
                const restroomCoords = { lat: restroom.latitude, lng: restroom.longitude };

                // Create a custom star icon
                const starIcon = document.createElement('div');
                starIcon.innerHTML = `<svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="#FFD700" stroke="#DAA520" stroke-width="1" stroke-linecap="round" stroke-linejoin="round" class="feather feather-star"><polygon points="12 2 15.09 8.26 22 9.27 17 14.14 18.18 21.02 12 17.77 5.82 21.02 7 14.14 2 9.27 8.91 8.26 12 2"></polygon></svg>`;
               
                const marker = new AdvancedMarkerElement({
                    map: map,
                    position: restroomCoords,
                    title: restroom.name,
                    content: starIcon
                });

                let distanceText = "N/A";
                if (userLocation) {
                    const dist = haversineDistance(userLocation, restroomCoords);
                    distanceText = `${dist.toFixed(2)} km`;
                }

                const restroomAddress = restroom.street + (restroom.city ? `, ${restroom.city}` : '') + (restroom.state ? `, ${restroom.state}` : '') + (restroom.unisex ? ' (Unisex)' : '');

                const contentString = `
                    <div>
                        <strong>${restroom.name || 'Public Restroom'}</strong><br>
                        Address: ${restroomAddress}<br>
                        Distance: ${distanceText}
                    </div>
                `;

                // Add click listener to show info window
                marker.addListener("click", () => {
                    infoWindow.setContent(contentString);
                    infoWindow.open(map, marker);
                });

                // Add to the list below the map
                const listItem = document.createElement('li');
                listItem.innerHTML = `<strong>${restroom.name || 'Public Restroom'}</strong>: ${restroomAddress} (Distance: ${distanceText})`;
                restroomsListElement.appendChild(listItem);
            });
        }
    </script>
    <script async src="https://maps.googleapis.com/maps/api/js?key=YOUR_API_KEY&callback=initMap&libraries=marker"></script>
</body>
</html>