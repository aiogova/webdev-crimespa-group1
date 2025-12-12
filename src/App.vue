<script setup>
import { reactive, ref, computed, onMounted } from 'vue'

let crime_url = ref('');
let urlSubmitted = ref(false);
let location = ref('');
let dialog_err = ref(false);
let location_err = ref(false);
let crimes = ref([]);                
let visible_crimes = ref([]);        
let neighborhoods = ref([]);
let codes = ref([]);
let map = reactive(
    {
        leaflet: null,
        selected_marker: null,
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
            {location: [44.942068, -93.020521], marker: null, neighborhood_id: null, neighborhood_name: null, number_of_crimes: null},
            {location: [44.977413, -93.025156], marker: null, neighborhood_id: null, neighborhood_name: null, number_of_crimes: null},
            {location: [44.931244, -93.079578], marker: null, neighborhood_id: null, neighborhood_name: null, number_of_crimes: null},
            {location: [44.956192, -93.060189], marker: null, neighborhood_id: null, neighborhood_name: null, number_of_crimes: null},
            {location: [44.978883, -93.068163], marker: null, neighborhood_id: null, neighborhood_name: null, number_of_crimes: null},
            {location: [44.975766, -93.113887], marker: null, neighborhood_id: null, neighborhood_name: null, number_of_crimes: null},
            {location: [44.959639, -93.121271], marker: null, neighborhood_id: null, neighborhood_name: null, number_of_crimes: null},
            {location: [44.947700, -93.128505], marker: null, neighborhood_id: null, neighborhood_name: null, number_of_crimes: null},
            {location: [44.930276, -93.119911], marker: null, neighborhood_id: null, neighborhood_name: null, number_of_crimes: null},
            {location: [44.982752, -93.147910], marker: null, neighborhood_id: null, neighborhood_name: null, number_of_crimes: null},
            {location: [44.963631, -93.167548], marker: null, neighborhood_id: null, neighborhood_name: null, number_of_crimes: null},
            {location: [44.973971, -93.197965], marker: null, neighborhood_id: null, neighborhood_name: null, number_of_crimes: null},
            {location: [44.949043, -93.178261], marker: null, neighborhood_id: null, neighborhood_name: null, number_of_crimes: null},
            {location: [44.934848, -93.176736], marker: null, neighborhood_id: null, neighborhood_name: null, number_of_crimes: null},
            {location: [44.913106, -93.170779], marker: null, neighborhood_id: null, neighborhood_name: null, number_of_crimes: null},
            {location: [44.937705, -93.136997], marker: null, neighborhood_id: null, neighborhood_name: null, number_of_crimes: null},
            {location: [44.949203, -93.093739], marker: null, neighborhood_id: null, neighborhood_name: null, number_of_crimes: null}
        ]
    }
);

let filters = reactive({
    neighborhoods: [], // array of selected neighborhood names
    incident_types: [], // array of selected incident type names
    startDate: '',      // YYYY-MM-DD
    endDate: '',        // YYYY-MM-DD
    maxIncidents: 1000, // default max
});

let incidentTypeGroups = computed(() => {
    const groups = {};
    codes.value.forEach(c => {
        const codeNum = parseInt(c.code);
        let typeName = c.type || "Unknown";
       
        // "first string before the first comma"
        if (typeName.includes(',')) {
            typeName = typeName.split(',')[0];
        }
        typeName = typeName.trim();
       
        if (!groups[typeName]) {
            groups[typeName] = [];
        }
        groups[typeName].push(codeNum);
    });
   
    // Return sorted keys alphabetically
    return Object.keys(groups).sort().reduce((obj, key) => {
        obj[key] = groups[key];
        return obj;
    }, {});
});



let placeholders = {
    case_number: "e.g. 12345678",
    date: "YYYY-MM-DD",
    time: "HH:MM:SS (e.g. 04:43:00)",
    code: "Numeric code (e.g. 110)",
    incident: "Incident description (string)",
    police_grid: "Grid number (integer, e.g. 82)",
    neighborhood_number: "Neighborhood ID (integer, e.g. 3)",
    block: "e.g. 123X BLOCK OF SOME ST",
};

let newIncident = ref({
    case_number: '',
    date: '',
    time: '',
    code: '',
    incident: '',
    police_grid: '',
    neighborhood_number: '',
    block: '',
});

let formError = ref('');
let formSuccess = ref('');

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

        // make neighborhood markers visible on map
        map.neighborhood_markers.forEach((n) => {
            n.marker = L.marker(n.location).addTo(map.leaflet);
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

            filterVisibleCrimes();
        })
        .catch(error => {
            console.error('Error fetching address:', error);
            location.value = `${lat.toFixed(6)}, ${lng.toFixed(6)}`;
        });
    });
});

// FUNCTIONS
function validateIncident(incident) {
    // case_number must be digits
    if (!/^\d+$/.test(incident.case_number)) return "Case number must be numeric";

    // date must match YYYY-MM-DD
    if (!/^\d{4}-\d{2}-\d{2}$/.test(incident.date)) return "Date must be YYYY-MM-DD";

    // time must match HH:MM:SS
    if (!/^\d{2}:\d{2}:\d{2}$/.test(incident.time)) return "Time must be HH:MM:SS";

    // code must be numeric
    if (!/^\d+$/.test(incident.code)) return "Code must be numeric";

    // police_grid must be numeric
    if (!/^\d+$/.test(incident.police_grid)) return "Police grid must be numeric";

    // neighborhood_number must be numeric
    if (!/^\d+$/.test(incident.neighborhood_number)) return "Neighborhood number must be numeric";

    // incident and block must not be empty
    if (!incident.incident.trim()) return "Incident description cannot be empty";
    if (!incident.block.trim()) return "Block cannot be empty";

    return null; // valid
}

async function submitNewIncident() {
    formError.value = ''
    formSuccess.value = ''

    // require all fields
    for (let [key, value] of Object.entries(newIncident.value)) {
        if (!value || value.toString().trim() === '') {
        formError.value = `Please fill out the '${key}' field.`
        return
        }
    }

    // validate formats/types
    const validationError = validateIncident(newIncident.value);
    if (validationError) {
        formError.value = validationError;
        return;
    }

    try {
        let response = await fetch(crime_url.value + '/new-incident', {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(newIncident.value),
        })

        if (!response.ok) {
        throw new Error('Request failed')
        }

        formSuccess.value = 'Incident uploaded successfully!'

        // re-fetch incidents so the UI updates
        await applyFilters();

        // clear the form
        Object.keys(newIncident.value).forEach(k => (newIncident.value[k] = ''))

    } catch (err) {
        formError.value = 'Failed to upload incident.'
    }
}

async function deleteIncident(caseNumber) {
    if (!confirm(`Are you sure you want to delete incident #${caseNumber}?`)) {
        return;
    }

    try {
        const response = await fetch(crime_url.value + '/remove-incident', {
            method: 'DELETE',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ case_number: caseNumber })
        });

        if (!response.ok) {
            throw new Error("Failed to delete incident");
        }

        alert(`Incident #${caseNumber} deleted successfully.`);

        // remove from table immediately (without requiring refresh)
        crimes.value = crimes.value.filter(c => c.case_number !== caseNumber);
        visible_crimes.value = visible_crimes.value.filter(c => c.case_number !== caseNumber);

        updateMarkerCounts();

    } catch (err) {
        console.error(err);
        alert("Error deleting incident.");
    }
}

function getCrimeCategory(type) {
    if (!type) return 'other-crime';
    const lowerType = type.toLowerCase();
   
    // Violent crimes: Crimes against another person
    const violentKeywords = ['assault', 'homicide', 'rape', 'robbery', 'murder', 'domestic', 'battery', 'person', 'abduction', 'kidnapping', 'sex', 'trafficking'];
   
    // Property crimes: Crimes against property
    const propertyKeywords = ['theft', 'burglary', 'arson', 'damage', 'vandalism', 'stolen', 'shoplifting', 'property', 'vehicle', 'fraud', 'embezzlement', 'forgery'];
   
    if (violentKeywords.some(k => lowerType.includes(k))) return 'violent-crime';
    if (propertyKeywords.some(k => lowerType.includes(k))) return 'property-crime';
   
    // Other crimes: Anything else
    return 'other-crime';
}

function cleanAddress(address) {
    // Replace 'X' in address numbers with '0'
    // Look for digits followed by X, e.g. "12X " -> "120 "
    return address.replace(/(\d+)X(?=\s)/g, '$10');
}

function selectCrime(crime) {
    if (!map.leaflet) return;

    // Scroll map into view
    const mapElement = document.getElementById('leafletmap');
    if (mapElement) {
        mapElement.scrollIntoView({ behavior: 'smooth' });
    }

    // Clean address
    let address = cleanAddress(crime.block);
   
    // Use Nominatim to geocode
    // Restrict to St Paul area for better results
    const viewbox = `${map.bounds.nw.lng},${map.bounds.nw.lat},${map.bounds.se.lng},${map.bounds.se.lat}`;
   
    fetch(`https://nominatim.openstreetmap.org/search?q=${encodeURIComponent(address + ", St. Paul, MN")}&format=json&viewbox=${viewbox}&bounded=1`)
    .then(response => response.json())
    .then(data => {
        if (data && data.length > 0) {
            const lat = parseFloat(data[0].lat);
            const lng = parseFloat(data[0].lon);
            const latLng = [lat, lng];

            // Remove existing selected marker
            if (map.selected_marker) {
                map.leaflet.removeLayer(map.selected_marker);
            }

            // Create new red marker
            const redIcon = new L.Icon({
                iconUrl: 'https://raw.githubusercontent.com/pointhi/leaflet-color-markers/master/img/marker-icon-2x-red.png',
                shadowUrl: 'https://cdnjs.cloudflare.com/ajax/libs/leaflet/0.7.7/images/marker-shadow.png',
                iconSize: [25, 41],
                iconAnchor: [12, 41],
                popupAnchor: [1, -34],
                shadowSize: [41, 41]
            });

            map.selected_marker = L.marker(latLng, { icon: redIcon }).addTo(map.leaflet);

            // Create popup content with delete button
            const popupDiv = document.createElement('div');
            popupDiv.innerHTML = `
                <p><strong>Date:</strong> ${crime.date}</p>
                <p><strong>Time:</strong> ${crime.time}</p>
                <p><strong>Incident:</strong> ${crime.incident}</p>
                <button class="deleteBtn popup-delete">Remove Marker</button>
            `;

            // Add event listener to the button
            const btn = popupDiv.querySelector('.popup-delete');
            btn.addEventListener('click', () => {
                // remove the marker from the map
                if (map.selected_marker) {
                    map.leaflet.removeLayer(map.selected_marker);
                    map.selected_marker = null;
                }
            });

            map.selected_marker.bindPopup(popupDiv).openPopup();

            // Pan map to location
            map.leaflet.setView(latLng, 16);
        } else {
            console.log("Could not find location for address:", address);
            alert("Could not locate this address on the map.");
        }
    })
    .catch(err => {
        console.error("Geocoding error:", err);
    });
}

function filterVisibleCrimes() {
    if (!map.leaflet) return; // make sure map exists

    let bounds = map.leaflet.getBounds();
    let nw = bounds.getNorthWest();
    let se = bounds.getSouthEast();

    visible_crimes.value = crimes.value.filter(c => {
        // find marker by neighborhood_id (both converted to numbers)
        let markerObj = map.neighborhood_markers.find(
            m => Number(m.neighborhood_id) === Number(c.neighborhood_number)
        );
        if (!markerObj) return false;

        return bounds.contains(L.latLng(markerObj.location[0], markerObj.location[1]));
    });
}

function updateMarkerCounts() {
    // Build counts per neighborhood_number from *current* crimes.value
    const counts = {};
    crimes.value.forEach(c => {
        const id = String(c.neighborhood_number);
        counts[id] = (counts[id] || 0) + 1;
    });

    // Build a lookup for neighborhood id to name
    const nameLookup = {};
    neighborhoods.value.forEach(n => {
        nameLookup[String(n.id)] = n.name;
    });

    // Update each neighborhood marker
    map.neighborhood_markers.forEach(m => {
        const nid = m.neighborhood_id != null ? String(m.neighborhood_id) : null;
        const count = nid && counts[nid] ? counts[nid] : 0;

        // Update the stored count
        m.number_of_crimes = count;

        // Update popup text if popup already exists
        const popupName =
            m.neighborhood_name ||
            (nid && nameLookup[nid]) ||
            "Unknown Neighborhood";

        if (m.marker) {
            m.marker.bindPopup(`${popupName}<br>Crimes: ${count}`);
        }
    });
}


// Function called once user has entered REST API URL
function initializeCrimes() {
    // TODO: get code and neighborhood data
    //       get initial 1000 crimes

     // fetch neighborhoods
    fetch(crime_url.value + '/neighborhoods')
    .then((response) => response.json())
    .then((neigh_data) => {
        neighborhoods.value = neigh_data;

        // once we have neighborhoods.value populated, assign id/name to marker objects
        map.neighborhood_markers.forEach((markerObj, index) => {
            let n = neighborhoods.value[index];
            if (n) {
                // copy id and name to the marker object
                markerObj.neighborhood_id = n.id;
                markerObj.neighborhood_name = n.name;

                // if the Leaflet marker was already created earlier, update its popup
                if (markerObj.marker) {
                    // re-bind popup with a string (safe). This will replace any previous popup
                    markerObj.marker.bindPopup(`Id: ${n.id}<br/>Name: ${n.name}`);
                }
            }
        });

        // fetch codes
        return fetch(crime_url.value + '/codes');
    })
    .then((response) => response.json())
    .then((code_data) => {
        codes.value = code_data;

        // fetch initial 1000 crimes
        return fetch(crime_url.value + '/incidents?limit=1000');
    })
    .then((response) => response.json())
    .then((crime_data) => {

        // update each crime record
        let neighLookup = {};
        let codeLookup = {};

        // build lookup from neighborhoods
        neighborhoods.value.forEach((n) => {
            neighLookup[n.id] = n.name;
        });

        // build lookup from codes
        codes.value.forEach((c) => {
            codeLookup[c.code] = c.type;
        });

        // add readable names to the crime data
        crime_data.forEach((c) => {
            c.neighborhood_name = neighLookup[c.neighborhood_number] || "Unknown";
            c.incident_type = codeLookup[c.code] || "Unknown";
        });

        crimes.value = crime_data;

        updateMarkerCounts();

        // immediately filter to show only visible ones
        filterVisibleCrimes();
    })
    .catch((err) => {
        console.log(err);
    });
}

// Function called when user presses 'OK' on dialog box
function closeDialog() {
    let dialog = document.getElementById('rest-dialog');
    let url_input = document.getElementById('dialog-url');
    if (crime_url.value !== '' && url_input.checkValidity()) {
        // normalize URL: remove trailing slash if present
        if (crime_url.value.endsWith('/')) {
            crime_url.value = crime_url.value.slice(0, -1);
        }

        urlSubmitted.value = true; // mark that URL has been entered
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

async function applyFilters() {
    try {
        const params = new URLSearchParams();

        // neighborhoods (use singular 'neighborhood' and IDs)
        if (filters.neighborhoods.length > 0) {
            params.append('neighborhood', filters.neighborhoods.join(','));
        }

        // incident types
        if (filters.incident_types && filters.incident_types.length > 0) {
            let selectedCodes = [];
            filters.incident_types.forEach(type => {
                if (incidentTypeGroups.value[type]) {
                    selectedCodes.push(...incidentTypeGroups.value[type]);
                }
            });
            if (selectedCodes.length > 0) {
                params.append('code', selectedCodes.join(','));
            }
        }

        // date range
        if (filters.startDate) params.append('start_date', filters.startDate);
        if (filters.endDate) params.append('end_date', filters.endDate);

        // max incidents
        let limit = Number(filters.maxIncidents);
        if (filters.neighborhoods.length === 0) {
            // only add +1 when no neighborhood filter, to fix off-by-one issue
            limit += 1;
        }
        params.append('limit', limit);

        const url = `${crime_url.value}/incidents?${params.toString()}`;

        const response = await fetch(url);
        if (!response.ok) throw new Error('Failed to fetch filtered incidents');

        const data = await response.json();

        // add neighborhood_name & incident_type to each crime
        const neighLookup = {};
        neighborhoods.value.forEach(n => { neighLookup[n.id] = n.name; });

        const codeLookup = {};
        codes.value.forEach(c => { codeLookup[c.code] = c.type; });

        data.forEach(c => {
            c.neighborhood_name = neighLookup[c.neighborhood_number] || 'Unknown';
            c.incident_type = codeLookup[c.code] || 'Unknown';
        });

        crimes.value = data;
        updateMarkerCounts();
        filterVisibleCrimes();
    } catch (err) {
        console.error(err);
    }
}

</script>

<template>
    <!-- Nav bar -->
     
    <div id="appContainer">
        <dialog id="rest-dialog" open>
            <h1 class="dialog-header">St. Paul Crime REST API</h1>
            <label class="dialog-label">URL: </label>
            <input id="dialog-url" class="dialog-input" type="url" v-model="crime_url" placeholder="http://localhost:8000" />
            <p class="dialog-error" v-if="dialog_err">Error: must enter valid URL</p>
            <br/>
            <button class="button" type="button" @click="closeDialog">OK</button>
        </dialog>

        <h1 id="pageTitle">St. Paul Crime SPA</h1>

        <div class="grid-container ">
            <div class="grid-x grid-padding-x">
                <div id="leafletmap" class="cell auto"></div>
            </div>
        </div>
       
        <div id="findLocation">
        <h1 class="dialog-header">Find Location</h1>
        <label class="dialog-label">Coordinates or Address: </label>
        <input id="dialog-loc" class="dialog-input" type="location" v-model="location" placeholder="893 Aldine St." />
        <p class="dialog-error" v-if="location_err">Error: Location not found. Must enter a valid coordinates or address.</p>
        <br/>
        <button class="button" type="button" @click="findLocation">GO</button>
        </div>

        <div class="grid-x grid-margin-x">
            <!-- New Incident Form -->
            <div v-if="urlSubmitted" class="new-incident-form cell small-12 medium-6">
                <h2>Add New Crime Incident</h2>
                <p v-if="formError" style="color: red; font-weight: bold;">
                    {{ formError }}
                </p>
                <p v-if="formSuccess" style="color: green; font-weight: bold;">
                    {{ formSuccess }}
                </p>

                <div v-for="(value, key) in newIncident" :key="key" style="margin-bottom: 8px;">
                    <label :for="key" style="font-weight: bold;">
                    {{ key.replace('_', ' ') }}:
                    </label>
                    <input
                    :id="key"
                    v-model="newIncident[key]"
                    :placeholder="placeholders[key]"
                    type="text"
                    style="display: block; padding: 5px;"
                    />
                </div>

                <button type="button" @click="submitNewIncident" style="margin-top: 10px; padding: 6px 12px;" class="submitBtn">
                    Submit Incident
                </button>
            </div>

            <!-- Filters -->
            <div v-if="urlSubmitted" class="filters cell small-12 medium-6">
                <h2>Filter Crimes</h2>
                <!-- Neighborhoods -->
                <div>
                    <h3>Neighborhoods</h3>
                    <div v-for="n in neighborhoods" :key="n.id">
                    <label>
                        <input
                        type="checkbox"
                        :value="n.id"
                        v-model="filters.neighborhoods"
                        />
                        {{ n.name }}
                    </label>
                    </div>
                </div>

                <!-- Incident Types -->
                <div>
                    <h3>Incident Types</h3>
                    <div style="max-height: 200px; overflow-y: auto; border: 1px solid #ccc; padding: 5px; border-radius: 6px;">
                        <div v-for="(codes, type) in incidentTypeGroups" :key="type">
                            <label>
                                <input
                                type="checkbox"
                                :value="type"
                                v-model="filters.incident_types"
                                />
                                {{ type }}
                            </label>
                        </div>
                    </div>
                </div>

                <!-- Date range -->
                <div>
                    <h3>Date Range</h3>
                    <label>Start Date: <input type="date" v-model="filters.startDate" /></label>
                    <label>End Date: <input type="date" v-model="filters.endDate" /></label>
                </div>

                <!-- Max incidents -->
                <div>
                    <h3>Max Incidents</h3>
                    <input type="number" v-model.number="filters.maxIncidents" min="1" />
                </div>

                <button class="submitBtn" type="button" @click="applyFilters">Update</button>
            </div>
        </div>


        <div v-if="urlSubmitted">
            <div class="legend-container">
                <h3>Crime Category Legend</h3>
                <div class="legend-content">
                    <div class="legend-item">
                        <span class="color-box violent"></span>
                        <span>Violent Crimes (Against Person)</span>
                    </div>
                    <div class="legend-item">
                        <span class="color-box property"></span>
                        <span>Property Crimes (Against Property)</span>
                    </div>
                    <div class="legend-item">
                        <span class="color-box other"></span>
                        <span>Other Crimes</span>
                    </div>
                </div>
            </div>

            <table>
                <thead>
                    <tr>
                        <th>Case Number</th>
                        <th>Date</th>
                        <th>Time</th>
                        <th>Incident Type</th>
                        <th>Incident</th>
                        <th>Police Grid</th>
                        <th>Neighborhood Name</th>
                        <th>Block</th>
                        <th>Delete</th>
                    </tr>
                </thead>

                <tbody>
                    <tr
                        v-for="c in visible_crimes"
                        :key="c.case_number"
                        :class="getCrimeCategory(c.incident_type)"
                        @click="selectCrime(c)"
                        style="cursor: pointer;"
                    >
                        <td>{{ c.case_number }}</td>
                        <td>{{ c.date }}</td>
                        <td>{{ c.time }}</td>
                        <td>{{ c.incident_type }}</td>
                        <td>{{ c.incident }}</td>
                        <td>{{ c.police_grid }}</td>
                        <td>{{ c.neighborhood_name }}</td>
                        <td>{{ c.block }}</td>
                        <td><button @click.stop="deleteIncident(c.case_number)" class="deleteBtn">Delete</button></td>
                    </tr>
                </tbody>
            </table>
        </div>
    </div>
</template>

<style scoped>
#rest-dialog {
    width: 20rem;
    margin-top: 1rem;
    z-index: 1000;
}

#leafletmap {
    width: 100%;
    height: 500px; /* fixed height so map always shows */
    border-radius: 10px;
    box-shadow: 0 4px 12px rgba(0,0,0,0.1);
    margin-top: 2rem;
}


.dialog-header {
    font-size: 2rem;
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

button {
    cursor: pointer;
}

.submitBtn {
    background-color: rgb(106, 106, 255);
}

.submitBtn:hover {
    background-color: blue;
}

.deleteBtn {
    background-color: rgb(106, 106, 255);
}

.deleteBtn:hover {
    background-color: blue;
}

#appContainer {
    background-color: #1e1e1e; /* dark smokey background */
    min-height: 100vh;          /* ensures it fills the viewport */
    padding: 1rem;
}


#findLocation {
    max-width: 1000px;             /* keeps the section a reasonable width */
    margin: 2rem auto;            /* centers horizontally and adds vertical spacing */
    padding: 1.5rem;              /* space inside the box */
    background-color: #fff;       /* white background to pop off the page */
    border-radius: 10px;          /* rounded corners */
    box-shadow: 0 4px 15px rgba(0,0,0,0.1); /* subtle drop shadow */
    text-align: center;           /* center the content inside */
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; /* nicer font */
}

#findLocation label {
    display: block;
    font-weight: 600;
    margin-bottom: 0.5rem;
}

#findLocation input {
    width: 100%;
    padding: 0.5rem;
    border-radius: 6px;
    border: 1px solid #ccc;
    margin-bottom: 0.5rem;
    font-size: 1rem;
}

#findLocation button {
    padding: 0.6rem 1.2rem;
    border-radius: 6px;
    border: none;
    background-color: #6a6aff;
    color: #fff;
    font-weight: 600;
    cursor: pointer;
    transition: background-color 0.2s ease;
}

#findLocation button:hover {
    background-color: #3d3dff;
}

.new-incident-form {
    max-width: 600px;                /* reasonable width */
    margin: 2rem auto;                /* centers horizontally, adds spacing above/below */
    padding: 1.5rem 2rem;             /* space inside */
    background-color: #fff;           /* white background */
    border-radius: 10px;              /* rounded corners */
    box-shadow: 0 4px 15px rgba(0,0,0,0.1); /* subtle drop shadow */
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; /* modern font */
}

.new-incident-form h2 {
    text-align: center;
    margin-bottom: 1rem;
    font-weight: 700;
    color: #1a1a1a;
}

.new-incident-form p {
    text-align: center;
    font-weight: 600;
    margin-bottom: 1rem;
}

.new-incident-form label {
    display: block;
    font-weight: 600;
    margin-bottom: 0.3rem;
}

.new-incident-form input {
    width: 100%;
    padding: 0.5rem;
    border-radius: 6px;
    border: 1px solid #ccc;
    margin-bottom: 0.8rem;
    font-size: 1rem;
}

.new-incident-form input {
    width: 100%;         /* full width of parent container */
    box-sizing: border-box; /* includes padding in width */
    padding: 0.5rem;      /* comfortable padding */
    margin-bottom: 0.8rem; /* spacing between fields */
    font-size: 1rem;
}

.new-incident-form .submitBtn {
    display: block;
    margin: 1rem auto 0 auto; /* centers the button */
    padding: 0.6rem 1.2rem;
    border-radius: 6px;
    border: none;
    background-color: #6a6aff;
    color: #fff;
    font-weight: 600;
    cursor: pointer;
    transition: background-color 0.2s ease;
}

.new-incident-form .submitBtn:hover {
    background-color: #3d3dff;
}

.filters {
    max-width: 600px;                /* keep consistent width */
    margin: 2rem auto;                /* center horizontally with vertical spacing */
    padding: 1.5rem 2rem;             /* inside spacing */
    background-color: #fff;           /* white background */
    border-radius: 10px;              /* rounded corners */
    box-shadow: 0 4px 15px rgba(0,0,0,0.1); /* subtle drop shadow */
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; /* modern font */
}

.filters h2 {
    text-align: center;
    margin-bottom: 1rem;
    font-weight: 700;
    color: #1a1a1a;
}

.filters h3 {
    margin-top: 1rem;
    margin-bottom: 0.5rem;
    font-weight: 600;
}

.filters label {
    display: block;
    font-weight: 600;
    margin-bottom: 0.3rem;
}

.filters input[type="number"],
.filters input[type="date"] {
    width: 100%;
    padding: 0.5rem;
    border-radius: 6px;
    border: 1px solid #ccc;
    margin-bottom: 0.8rem;
    font-size: 1rem;
}

.filters input[type="checkbox"] {
    margin-right: 0.5rem;
}

.filters .submitBtn {
    display: block;
    margin: 1rem auto 0 auto; /* center button */
    padding: 0.6rem 1.2rem;
    border-radius: 6px;
    border: none;
    background-color: #6a6aff;
    color: #fff;
    font-weight: 600;
    cursor: pointer;
    transition: background-color 0.2s ease;
}

.filters .submitBtn:hover {
    background-color: #3d3dff;
}

.deleteBtn {
    padding: 0.4rem 0.8rem;
    border-radius: 6px;
    border: none;
    background-color: #ff5c5c;  /* red for delete */
    color: #fff;
    font-weight: 600;
    cursor: pointer;
    transition: background-color 0.2s ease, transform 0.1s ease;
}

.deleteBtn:hover {
    background-color: #ff0000;
    transform: scale(1.05);  /* slight pop on hover */
}

table {
    width: 100%;
    border-collapse: collapse;
    margin-top: 2rem;
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    background-color: #fff;
    box-shadow: 0 4px 15px rgba(0,0,0,0.05);
    border-radius: 10px;
    overflow: hidden; /* rounds the corners */
}

thead {
    background-color: #6a6aff;
    color: #fff;
    font-weight: 600;
    text-align: left;
}

thead th {
    padding: 0.8rem 1rem;
}

tbody td {
    padding: 0.6rem 1rem;
    border-bottom: 1px solid #eee;
}

tbody tr:nth-child(even) {
    background-color: #f9f9f9; /* subtle zebra striping */
}

tbody tr:hover {
    background-color: #e6e6ff; /* highlight on hover */
}

#pageTitle {
    text-align: center;               /* centers the title */
    font-family: 'Poppins', 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; /* modern, clean font */
    font-size: 2.5rem;                /* big, bold title */
    font-weight: 700;
    color: #f0f0f0;            /* light text */
    margin: 2rem 0 1rem 0;            /* space above and below */
    letter-spacing: 1px;              /* slight spacing between letters */
    text-shadow: 1px 1px 2px rgba(255, 255, 255, 0.1);
}

#findLocation label,
.new-incident-form label,
.filters label {
    font-size: 1.2rem; /* labels bigger too */
    font-weight: 600;  /* bold labels for clarity */
}

#findLocation input,
.new-incident-form input,
.filters input {
    font-size: 1rem; /* input text readable */
}

.new-incident-form, .filters, #findLocation, #leafletmap, table {
    box-shadow: 0 4px 20px rgba(255, 255, 255, 0.201); /* darker shadow for contrast on dark bg */
}

.legend-container {
    max-width: 800px;
    margin: 2rem auto 1rem auto;
    padding: 1rem 1.5rem;
    background-color: #fff;
    border-radius: 10px;
    box-shadow: 0 4px 15px rgba(0,0,0,0.1);
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
}

.legend-container h3 {
    margin-top: 0;
    margin-bottom: 1rem;
    text-align: center;
    font-size: 1.2rem;
    font-weight: 700;
}

.legend-content {
    display: flex;
    justify-content: center;
    gap: 2rem;
    flex-wrap: wrap;
}

.legend-item {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    font-weight: 600;
}

.color-box {
    width: 24px;
    height: 24px;
    border-radius: 4px;
    border: 1px solid #ccc;
    display: inline-block;
}

.color-box.violent { background-color: #f8d7da; }
.color-box.property { background-color: #d4edda; }
.color-box.other { background-color: #fff3cd; }

/* Table Row Coloring */
/* Override the nth-child styling for colored rows, but keep hover effect */
tbody tr.violent-crime { background-color: #f8d7da; }
tbody tr.property-crime { background-color: #d4edda; }
tbody tr.other-crime { background-color: #fff3cd; }

/* Ensure hover state is visible */
tbody tr.violent-crime:hover { background-color: #f5c6cb; }
tbody tr.property-crime:hover { background-color: #c3e6cb; }
tbody tr.other-crime:hover { background-color: #ffeeba; }

</style>

<style>
/* Global styles for Leaflet popups */
.popup-delete {
    display: block;
    width: 100%;
    margin-top: 10px;
    padding: 8px 12px;
    background-color: #ff5c5c;
    color: white;
    border: none;
    border-radius: 6px;
    cursor: pointer;
    font-weight: 600;
    text-align: center;
    box-shadow: 0 2px 4px rgba(0,0,0,0.2);
    transition: background-color 0.2s, transform 0.1s;
}

.popup-delete:hover {
    background-color: #ff0000;
    transform: translateY(-1px);
}

.popup-delete:active {
    transform: translateY(0);
}
</style>