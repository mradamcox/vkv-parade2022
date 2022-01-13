<script>
import {onMount} from 'svelte';

import 'ol/ol.css';
import Map from 'ol/Map';
import View from 'ol/View';
import Feature from 'ol/Feature';

import Point from 'ol/geom/Point';

import VectorSource from 'ol/source/Vector';
import OSM from 'ol/source/OSM';
import XYZ from 'ol/source/XYZ';
import TileWMS from 'ol/source/TileWMS';

import TileLayer from 'ol/layer/Tile';
import ImageLayer from 'ol/layer/Image';
import VectorLayer from 'ol/layer/Vector';

import {fromLonLat} from 'ol/proj';

let map;
let data;
let track = true;
let positions = [];
let devices = [];
let carList = {};

const mbToken = MAPBOX_TOKEN;
const tcToken = TC_TOKEN;
const server = SERVER;

const mgFountain = fromLonLat([-90.0947302, 30.0255355]);

const osmLayer = new TileLayer({
  source: new OSM(),
})

const markerSource = new VectorSource();
const carLayer = new VectorLayer({
	source: markerSource,
});

class Car {

	constructor(id, name) {
		this.id = id;
		this.name = name;
	}

	updatePosition(newLngLat) {
		this.position = newLngLat;
	}
	
}

$: {
	devices.forEach( function (device) {
		// uncomment this line to create a new point for every socket message
		// device.id = Math.floor(Math.random() * 1000);
		let feature = markerSource.getFeatureById(device.id)
		if (feature == null) {
			feature = new Feature ();
			feature.setId(device.id)
			feature.setProperties({"name": device.name})
			markerSource.addFeature(feature);
		}
		positions.forEach( function (position) {
			if (position.deviceId == feature.getId()) {
				let p = fromLonLat([position.longitude, position.latitude])
				feature.setGeometry(new Point(p))
			}
		});
	})
	if (map && markerSource.getFeatures().length > 0 && track) {
		map.getView().fit(markerSource.getExtent(), {
			duration: 1000,
			maxZoom: 19,
			padding: [100, 100, 100, 100],
		});
	}
}

const bgBasemap = new TileLayer({
	source: new XYZ({
		// url: 'https://api.mapbox.com/styles/v1/mapbox/satellite-streets-v10/tiles/{z}/{x}/{y}?access_token='+mbToken,
		url: 'https://api.mapbox.com/styles/v1/legiongis/ckg13im88155l19pdgv294lu0/tiles/{z}/{x}/{y}?access_token='+mbToken,
		tileSize: 512,
	})	
});

const goldBasemap = new TileLayer({
	source: new XYZ({
		url: 'https://api.mapbox.com/styles/v1/legiongis/cky94td2i231f15ol21yudv39/tiles/{z}/{x}/{y}?access_token='+mbToken,
		tileSize: 512,
	}),
	opacity: 0,	
});

function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

function TrackMap (elementId) {
	const el = document.getElementById(elementId);
	map = new Map({
		target: el,
		view: new View({
			center: mgFountain,
			zoom: 16,
		}),
		controls: [],
	});

	map.addLayer(bgBasemap);
	map.addLayer(goldBasemap);
	map.addLayer(carLayer);

}

function startSession () {
	fetch("https://"+server+"/api/session?token="+tcToken)
	.then((response) => { openWebSocket() } )
	.catch((error) => {
		console.log(error)
	});
}

function openWebSocket() {

        const socket = new WebSocket("wss://"+server+"/gps/api/socket");
        socket.onmessage = function (event) {
                data = JSON.parse(event.data);
                if (data.positions) { positions = data.positions }
                if (data.devices) { devices = data.devices }
        }

}

async function getWeird() {
        let fadeIn;
	let opacity = 0;
	let increment = .1
	while (true) {
		await sleep(100);
		if (opacity >= 1) { fadeIn = false }
		if (opacity <= 0) { fadeIn = true}
		opacity = fadeIn ? opacity + increment : opacity - increment;
		goldBasemap.setOpacity(opacity);
	}
}

onMount(() => {
	const mapView = new TrackMap("map");
	startSession();
	getWeird();
});

</script>

<main>
	<div id="map"></div>
	<label style="position:absolute; right:0; top:0;">track <input type="checkbox" bind:checked={track}></label>
	<div style="position:absolute; right:0; top:50px; background:red;">
		{#each devices as device}
		{device.id}: {device.name}
			{#each positions as position}
				{#if position.deviceId == device.id}
					{position.latitude}, {position.longitude}
				{/if}
			{/each}
		{/each}
	</div>
</main>

<style>

main {
	font-family: sans-serif;
}

#map {
	width: 100%;
	height: 100vh;
}

@media (min-width: 640px) {
	main {
		max-width: none;
	}
}
</style>
