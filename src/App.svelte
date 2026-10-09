<script>
    // COMPONENTS
    import Papa from 'papaparse';
    import { onMount } from 'svelte';
    import * as turf from '@turf/helpers';
    import MaplibreGeocoder from '@maplibre/maplibre-gl-geocoder';
    import '@maplibre/maplibre-gl-geocoder/dist/maplibre-gl-geocoder.css';
    import PointsWithinPolygon from '@turf/points-within-polygon';
    import RidingDetails from '$components/RidingDetails.svelte';

    // DATA
    import ridingsPrev from '$data/riding-boundaries-2021.js';
    import ridingsCurrent from '$data/riding-boundaries-2025.js';
    const resultsUrl = 'https://docs.google.com/spreadsheets/d/e/2PACX-1vR_QVHNdI3f4m1klbM1yPoyFx4eZ1l69DPJUazYAnAYr6q73talOa2AuY03EH_dwixMBzZaiPg1xxwk/pub?gid=1366195801&single=true&output=csv';
    const candidatesUrl = 'https://docs.google.com/spreadsheets/d/e/2PACX-1vR_QVHNdI3f4m1klbM1yPoyFx4eZ1l69DPJUazYAnAYr6q73talOa2AuY03EH_dwixMBzZaiPg1xxwk/pub?gid=715203723&single=true&output=csv';



    // VARIABLES
    const geocodeEl = 'geocoder';
    let candidateData, resultsData, geocoder;
    const bbox = [-139.595032,48.302552,-113.887024,60.199227];
    // Create mock map object with required methods
    const fakeMap = {
        getZoom: () => 2,
        on: () => {},
        flyTo: () => {}
    };


    // REACTIVE VARIABLES
    $: ridingResults = [];
    $: ridingPrev = '';
    $: ridingCurrent = '';
    $: ridingCandidates = [];


    // custom nominatim geocoder API wrapper    
    const nominatimGeocoderApi = {
        forwardGeocode: async ({ query }) => {
            const params = new URLSearchParams({
                q: query,
                addressdetails: '1',
                countrycodes: 'ca',
                format: 'json',
                limit: '5',
                bounded: '1',
                viewbox: `${bbox[0]}, ${bbox[3]},${bbox[2]},${bbox[1]}`// top-left, bottom-right
            });

            const url = `https://nominatim.openstreetmap.org/search?${params.toString()}`;

            try {
                const response = await fetch(url, {
                    headers: {
                        'Accept': 'application/json',
                        'Accept-Language': 'en',
                        'User-Agent': 'Candidate lookup (ngriffiths@postmedia.com)'
                    }
                });

                if (!response.ok) {
                    throw new Error(`HTTP error! status: ${response.status}`);
                }

                const data = await response.json();
                // console.log('✅ Geocoder results:', data);
                
                return {
                    features: data.map(result => {
                        const { house_number, road, city, town, village, suburb, postcode } = result.address;

                        const addressParts = [
                            house_number,
                            road,
                            suburb,
                            city || town || village,
                            postcode
                        ].filter(Boolean)

                        const address = addressParts.join(', ');

                        return {
                            center: [parseFloat(result.lon), parseFloat(result.lat)],
                            geometry: {
                                type: 'Point',
                                coordinates: [parseFloat(result.lon), parseFloat(result.lat)],
                            },
                            place_name: address || result.display_name,
                            properties: {},
                            text: address || result.display_name.split(',')[0],
                            type: 'Feature'
                        };
                    })
                };
            } catch (err) {
                console.error(`❌ Nominatim geocoding error: ${err}`);
                return { features: [] };
            }
        }
    };

    // nominatim geocoder
    geocoder = new MaplibreGeocoder(nominatimGeocoderApi, {
        // maplibregl: { Map: function() {} }, // Required, even if no map
        marker: false,
        placeholder: "Lookup a B.C. address...",
    });

    // /FUNCTIONS
    function addGeocoderGL(geocodeEl) {
        // console.log(geocoder)
        // add event listeners
        geocoder
            .on('result', handleGeocodeResults)
            .on('error', handleGeocodeError);

        // add to document
        document.getElementById(geocodeEl).appendChild(geocoder.onAdd(fakeMap));
    }

    async function fetchData(url) {
        return new Promise((resolve, reject) => {
            Papa.parse(url, {
                download: true,
                header: true,
                complete: results => {
                    resolve(results.data);
                },
                error: (err, file) => {
                    reject(err);
                }
            })
        });
    }

    function getCandidates(ridingName, data) {
        return data.filter(d => d.electoral_district === ridingName)
            .sort((a,b) => a.candidate.split(' ')[1].localeCompare(b.candidate.split(' ')[1]));
    }

    function getRidingResults(ridingName, data) {
        return data.filter(d => d.ed_name === ridingName)
            .sort((a,b) => parseFloat(b.pct_votes) - parseFloat(a.pct_votes));
    }

    function getRiding(latlon, ridings) {
        let ridingName;
        const point = turf.point(latlon);

        // find out which riding the point is inside
	    ridings.features.forEach(d => {
		    // find which polygon the point is within
		    const withinPoly = PointsWithinPolygon(point, d);

		    if (withinPoly.features.length > 0) {
                ridingName = d.properties.ED_NAME;
                return;
            }
	    });

        return ridingName;
    }

    function handleGeocodeError(e) {
        console.log(e);
    }

    function handleGeocodeResults(e) {
        // console.log(e)
        if (e.detail !== null) {
            const latlon = e.result.center;
            ridingCurrent = getRiding(latlon, ridingsCurrent); // ridingsCurrent
            ridingPrev = getRiding(latlon, ridingsPrev); // ridingsPrev

            // console.log(ridingCurrent, ridingPrev)
        
            // get results from the previousriding 
            ridingResults = getRidingResults(ridingPrev, resultsData);
            // current candidate list from riding
            ridingCandidates = getCandidates(ridingCurrent, candidateData);
        }
    }

    function handleGeocodeEarthEvents() {
        const el = document.querySelector('ge-autocomplete');

        el.addEventListener('select', handleGeocodeResults);

        el.addEventListener('error', e => {
            console.log(e.detail, e)
        });
    }

    async function init() {
        // create the geocode element
        addGeocoderGL(geocodeEl);

        // fetch candidate data
        candidateData = await fetchData(candidatesUrl);

        // fetch vote result data
        resultsData = await fetchData(resultsUrl);
    }

    onMount(init);
</script>

<header>
    <h1>Who’s running in my riding?</h1>
</header>

<main>
    
    <div class="geocoding">
        <span class="voting-emoji">🗳️</span>
        <div id="geocoder"></div>
        <span class="voting-emoji">🗳️</span>
    </div>

    <RidingDetails 
        riding={ridingCurrent}
        ridingPrev={ridingPrev}
        results={ridingResults}
        candidates={ridingCandidates}
    />
    
</main>

<footer>
    <p class="note">NOTE: Parties are not required to run candidates in every riding.</p>
    <p class="source">Source: Elections Canada, Political parties</p>
</footer>
  
<style>
    @import '$css/normalize.css';
    @import '$css/fonts.css';
    @import '$css/colors.css';
    @import '$css/app.css';

    header {
		margin-bottom: 2rem;
	}
	header > h1 {
		text-align: center;
	}
    #geocoder {
        z-index: 1;
        margin: 0 10px;
    }
    .geocoding {
        display: flex;
        justify-content: center;
        margin: 0 auto;
        min-width: 320px;
    }
    .geocoding .voting-emoji {
        font-size: 2rem;
        padding: 4px 8px 0 8px;
    }
  	:global(input:focus) {
		outline: none;
  	}   
	:global(
		.svelte-select .selected-item,
		.svelte-select .item,
		.svelte-select input
	) {
		font-family: 'BentonSansCond-Regular', sans;
	}
</style>
