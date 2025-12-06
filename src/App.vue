<script setup>
import { reactive, ref, onMounted } from 'vue'

let crime_url = ref('');
let location = ref('');
let dialog_err = ref(false);
let location_err = ref(false);
let map = reactive(
    {
        leaflet: null,
        center: {
            lat: 44.955139,
            lng: -93.102222,
            address: ''
        },
        zoom: 12,
        bounds: {
            nw: {lat: 45.008206, lng: -93.217977},
            se: {lat: 44.883658, lng: -92.993787}
        },
        neighborhood_markers: [
            {location: [44.942068, -93.020521], marker: null},
            {location: [44.977413, -93.025156], marker: null},
            {location: [44.931244, -93.079578], marker: null},
            {location: [44.956192, -93.060189], marker: null},
            {location: [44.978883, -93.068163], marker: null},
            {location: [44.975766, -93.113887], marker: null},
            {location: [44.959639, -93.121271], marker: null},
            {location: [44.947700, -93.128505], marker: null},
            {location: [44.930276, -93.119911], marker: null},
            {location: [44.982752, -93.147910], marker: null},
            {location: [44.963631, -93.167548], marker: null},
            {location: [44.973971, -93.197965], marker: null},
            {location: [44.949043, -93.178261], marker: null},
            {location: [44.934848, -93.176736], marker: null},
            {location: [44.913106, -93.170779], marker: null},
            {location: [44.937705, -93.136997], marker: null},
            {location: [44.949203, -93.093739], marker: null}
        ]
    }
);

// Vue callback for once <template> HTML has been added to web page
onMounted(() => {
    // Create Leaflet map (set bounds and valied zoom levels)
    map.leaflet = L.map('leafletmap').setView([map.center.lat, map.center.lng], map.zoom);
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
        attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
        minZoom: 11,
        maxZoom: 18
    }).addTo(map.leaflet);
    map.leaflet.setMaxBounds([[44.883658, -93.217977], [45.008206, -92.993787]]);

    // Get boundaries for St. Paul neighborhoods
    let district_boundary = new L.geoJson();
    district_boundary.addTo(map.leaflet);
    fetch('data/StPaulDistrictCouncil.geojson')
    .then((response) => {
        return response.json();
    })
    .then((result) => {
        result.features.forEach((value) => {
            district_boundary.addData(value);
        });
    })
    .catch((error) => {
        console.log('Error:', error);
    });

    // Update location input on map movement
    map.leaflet.on('moveend', () => {
        let center = map.leaflet.getCenter();
        let lat = center.lat;
        let lng = center.lng;

        // Use Nominatim reverse geocoding to get address
        fetch(`https://nominatim.openstreetmap.org/reverse?lat=${lat}&lon=${lng}&format=json`)
        .then(response => response.json())
        .then(data => {
            if (data && data.display_name) {
                location.value = data.display_name;
            } else {
                location.value = `${lat.toFixed(6)}, ${lng.toFixed(6)}`;
            }
        })
        .catch(error => {
            console.error('Error fetching address:', error);
            location.value = `${lat.toFixed(6)}, ${lng.toFixed(6)}`;
        });
    });
});


// FUNCTIONS
// Function called once user has entered REST API URL
function initializeCrimes() {
    // TODO: get code and neighborhood data
    //       get initial 1000 crimes
}

// Function called when user presses 'OK' on dialog box
function closeDialog() {
    let dialog = document.getElementById('rest-dialog');
    let url_input = document.getElementById('dialog-url');
    if (crime_url.value !== '' && url_input.checkValidity()) {
        dialog_err.value = false;
        dialog.close();
        initializeCrimes();
    }
    else {
        dialog_err.value = true;
    }
}

// Function called when user pressed 'GO' on location dialog box
function findLocation() {
    let loc_input = location.value;
    if (loc_input === '') {
        location_err.value = true;
        return;
    }
    location_err.value = false;

    // Check if input is lat/long coordinates
    let latLongRegex = /^(-?\d+(\.\d+)?),\s*(-?\d+(\.\d+)?)$/;
    let match = loc_input.match(latLongRegex);

    if (match) {
        let lat = parseFloat(match[1]);
        let lng = parseFloat(match[3]);

        // Clamp values if outside St. Paul's bounding box
        if (lat > map.bounds.nw.lat) lat = map.bounds.nw.lat;
        if (lat < map.bounds.se.lat) lat = map.bounds.se.lat;
        if (lng < map.bounds.nw.lng) lng = map.bounds.nw.lng;
        if (lng > map.bounds.se.lng) lng = map.bounds.se.lng;

        map.leaflet.setView([lat, lng], 16);
    } else {
        // Use Nominatim API to convert address to lat/long
        fetch(`https://nominatim.openstreetmap.org/search?q=${loc_input}&format=json&viewbox=${map.bounds.nw.lng},${map.bounds.nw.lat},${map.bounds.se.lng},${map.bounds.se.lat}&bounded=1`)
        .then(response => response.json())
        .then(data => {
            if (data.length > 0) {
                let lat = parseFloat(data[0].lat);
                let lng = parseFloat(data[0].lon);
                map.leaflet.setView([lat, lng], 16);
            } else {
                location_err.value = true;
            }
        })
        .catch(error => {
            console.error('Error fetching location:', error);
            location_err.value = true;
        });
    }
}
</script>

<template>
    <dialog id="rest-dialog" open>
        <h1 class="dialog-header">St. Paul Crime REST API</h1>
        <label class="dialog-label">URL: </label>
        <input id="dialog-url" class="dialog-input" type="url" v-model="crime_url" placeholder="http://localhost:8000" />
        <p class="dialog-error" v-if="dialog_err">Error: must enter valid URL</p>
        <br/>
        <button class="button" type="button" @click="closeDialog">OK</button>
    </dialog>
    <div class="grid-container ">
        <div class="grid-x grid-padding-x">
            <div id="leafletmap" class="cell auto"></div>
        </div>
    </div>
    <h1 class="dialog-header">Find Location</h1>
    <label class="dialog-label">Coordinates or Address: </label>
    <input id="dialog-loc" class="dialog-input" type="location" v-model="location" placeholder="893 Aldine St." />
    <p class="dialog-error" v-if="location_err">Error: Location not found. Must enter a valid coordinates or address.</p>
    <br/>
    <button class="button" type="button" @click="findLocation">GO</button>
</template>

<style scoped>
#rest-dialog {
    width: 20rem;
    margin-top: 1rem;
    z-index: 1000;
}

#leafletmap {
    height: 500px;
}

.dialog-header {
    font-size: 1.2rem;
    font-weight: bold;
}

.dialog-label {
    font-size: 1rem;
}

.dialog-input {
    font-size: 1rem;
    width: 100%;
}

.dialog-error {
    font-size: 1rem;
    color: #D32323;
}
</style>
