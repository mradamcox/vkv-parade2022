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

let mapView;
let data;
let positions = [];
let devices = [];

const mbToken = MAPBOX_TOKEN;
const tcToken = TC_TOKEN;

const mgFountain = fromLonLat([-90.0947302, 30.0255355]);

const osmLayer = new TileLayer({
  source: new OSM(),
})

const imageryLayer = new TileLayer({
  source: new XYZ({
    url: 'https://api.mapbox.com/styles/v1/mapbox/satellite-streets-v10/tiles/{z}/{x}/{y}?access_token='+mb_api,
    tileSize: 512,
  })
});

const basemaps = [
	{ id: "osm", layer: osmLayer, label: "Streets" },
	{ id: "satellite", layer: imageryLayer, label: "Streets+Satellite" },
]
let currentBasemap = basemaps[0].id;

function TrackMap (elementId) {
	const el = document.getElementById(elementId);
	const map = new Map({
		target: el,
		// layers: [basemaps[0]],
		view: new View({
		center: [-90, 30],
		zoom: 1,
		maxZoom: 8,
		})
	});

	map.addLayer(imageryLayer)

	self.map = map;
}

function openWebSocket() {

	const socket = new WebSocket("ws://tracker.toulouse.casa:8082/api/socket");

	socket.onclose = function (event) {
		console.log("closed socket");
	}

	socket.onmessage = function (event) {
		data = JSON.parse(event.data);
		if (data.positions) { positions = data.positions }
		if (data.devices) { devices = data.devices }
	}

}

function startSession () {
	fetch("http://tracker.toulouse.casa:8082/api/session?token="+tcToken, {
		method: "POST",
		dataType: "json",
		contentType: "application/json",
		headers: {
			"Content-Type": "application/json",
		},
	})
	.then( openWebSocket() );
}

onMount(() => {

	mapView = new TrackMap("map");
	startSession();
});

</script>

<main>
	<div id="map"></div>
	<button disabled style="position:absolute; right:0; top:0;">refresh</button>
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
