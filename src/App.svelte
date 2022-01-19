<script>
import {onMount} from 'svelte';
import qrcode from 'qrcode-generator';

import 'ol/ol.css';
import Map from 'ol/Map';
import View from 'ol/View';
import Feature from 'ol/Feature';

import Point from 'ol/geom/Point';
import Overlay from 'ol/Overlay';

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
const baseUrl = SERVER_BASEURL;

//if (baseUrl.startsWith("https")) {
 // const socketUrl = baseUrl.replace("https","wss");
//} else {
//  const socketUrl = baseUrl.replace("http","ws");
//}

const socketUrl = baseUrl.startsWith("https") ? baseUrl.replace("https","wss") : baseUrl.replace("http","ws");
console.log(socketUrl);

const mgFountain = fromLonLat([-90.0361081, 29.9694733]);

const osmLayer = new TileLayer({
	source: new OSM(),
})

const markerSource = new VectorSource();
const carLayer = new VectorLayer({
	source: markerSource,
});

let cars = {};

class Car {

	constructor(id, name) {
		this.id = id;
		this.divId = "car-"+id;
		this.name = name;
		this.overlay = null;
		this.addOverlay();

	}

	updatePosition(newLngLat) {
		console.log(this.id + ": setting new position: " + newLngLat);
		if (this.overlay != null) {
			this.overlay.setPosition(newLngLat);
		}
	}

	addOverlay() {
	// create a new Overlay, fill it with a QR code that is the
	// car's name, and add it to the map
		let div = document.createElement("div");
		div.id = this.divId;
		const qr = qrcode(2, 'L');
		qr.addData(this.name);
		qr.make();
		div.innerHTML = qr.createImgTag();
		document.body.appendChild(div);
		this.overlay = new Overlay({
                        element: div,
			className: 'ol-overlay-container ol-selectable animation_move',
			autoPan: true,
                });
		//document.body.appendChild(div);
		map.addOverlay(this.overlay);
	}

	removeOverlay() {
	// remove the Overlay for this car from the map.
	}		
	
}

function setPositions(positions) {
	// update the position of all cars
	positions.forEach( function (position) {
                if (cars[position.deviceId]) {
			console.log("setting new position for");
			console.log(cars[position.deviceId])
			let p = fromLonLat([position.longitude, position.latitude]);
			cars[position.deviceId].updatePosition(p);
		}
	});
}

$: setPositions(positions);

function setCars(devices) {
        // update the position of all cars
        devices.forEach( function (device) {
                if (!cars[device.id]) {
                        console.log("create car");
			cars[device.id] = new Car(device.id, device.name)

                }
        });
	console.log(cars)
}

$: setCars(devices);


//{
//	devices.forEach( function (device) {
		// uncomment this line to create a new point for every socket message
		// device.id = Math.floor(Math.random() * 1000);
//		let feature = markerSource.getFeatureById(device.id)
//		if (feature == null) {
//			feature = new Feature ();
//			feature.setId(device.id)
//			feature.setProperties({"name": device.name})
//			markerSource.addFeature(feature);
//		}
//		positions.forEach( function (position) {
//			if (position.deviceId == feature.getId()) {
//				let p = fromLonLat([position.longitude, position.latitude])
//				feature.setGeometry(new Point(p))
//			}
//		});
//	})
//	if (map && markerSource.getFeatures().length > 0 && track) {
//		map.getView().fit(markerSource.getExtent(), {
//			duration: 1000,
//			maxZoom: 19,
//			padding: [100, 100, 100, 100],
//		});
//	}
//}

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
			zoom: 13,
		}),
		controls: [],
	});

	map.addLayer(bgBasemap);
	map.addLayer(goldBasemap);
	map.addLayer(carLayer);

}

function startSession () {
	fetch(baseUrl+"/api/session?token="+tcToken)
	.then((response) => { openWebSocket() } )
	.catch((error) => {
		console.log(error)
	});
}

function openWebSocket() {

        const socket = new WebSocket(socketUrl+"/api/socket");
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
	<div id="placeholder" style="position:absolute; right:0; top:100px;"></div>

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

.animation_move {
    -webkit-transition: all 0.7s ease-out;
    -moz-transition: all 0.7s ease-out;
    -ms-transition: all 0.7s ease-out;
    -o-transition: all 0.7s ease-out;
    transition: all 0.7s ease-out;
}

</style>
