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
	const map = new Map({
		target: el,
		view: new View({
			center: mgFountain,
			zoom: 16,
		}),
		controls: [],
	});

	map.addLayer(bgBasemap)
	map.addLayer(goldBasemap)

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
	fetch("https://tracker.toulouse.casa:8082/api/session?token="+tcToken, {
		method: "POST",
		dataType: "json",
		contentType: "application/json",
		headers: {
			"Content-Type": "application/json",
		},
	})
	.then( openWebSocket() );
}

async function getWeird() {
	let opacity = 0;
	let fadeIn = true;
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
	mapView = new TrackMap("map");
	startSession();
	getWeird();
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
