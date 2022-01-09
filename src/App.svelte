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

let mb_api = "pk.eyJ1IjoibGVnaW9uZ2lzIiwiYSI6ImNreTdmMXR6cjE0dHQyc29ieHUxcG54bnYifQ.8We5T2uz0xnEHO5gYzUIUw";

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

function getLocations() {
	fetch("", {
      method: 'POST',
    //   headers: {
    //     'Content-Type': 'application/json;charset=utf-8',
    //   },
    //   body: data,
    })
    .then(response => response.json())
    .then(result => {
      console.log(result)
    });
}

onMount(() => {

	mapView = new TrackMap("map");
});

</script>

<main>
	<div id="map"></div>
	<button on:click={getLocations} style="position:absolute; right:0; top:0;">refresh</button>
</main>

<style>
	#map {
		width: 100%;
		max-width: 800px;
		height: 100vh;
	}

	@media (min-width: 640px) {
		main {
			max-width: none;
		}
	}
</style>