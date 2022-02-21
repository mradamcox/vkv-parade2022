<script>
	import {onMount} from 'svelte';
	import qrcode from 'qrcode-generator';
	
  import { closeModal } from 'svelte-modals'

  // provided by <Modals />
  export let isOpen

  export let title
  export let message

	import 'ol/ol.css';
	import Map from 'ol/Map';
	import View from 'ol/View';
	import Feature from 'ol/Feature';
	
	import Point from 'ol/geom/Point';
	import Overlay from 'ol/Overlay';

	import { boundingExtent } from 'ol/extent';
	import { transformExtent } from 'ol/proj';
	import { get } from 'ol/proj';
	
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
	let showInfo = false;
	let track = true;
	let showQRs = true;
	let positions = [];
	let devices = [];
	let carList = [];
	
	const mbToken = MAPBOX_TOKEN;
	const tcToken = TC_TOKEN;
	const tcUser = TC_USER;
	const tcPassword = TC_PASSWORD;
	const baseUrl = SERVER_BASEURL;
	
	//if (baseUrl.startsWith("https")) {
	 // const socketUrl = baseUrl.replace("https","wss");
	//} else {
	//  const socketUrl = baseUrl.replace("http","ws");
	//}
	
	const socketUrl = baseUrl.startsWith("https") ? baseUrl.replace("https","wss") : baseUrl.replace("http","ws");
	
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
			this.position = null;
			this.color = rainbow[carList.length];

			this.addOverlay();
	
		}
	
		updatePosition(lngLat) {
			// console.log(this.id + ": setting new position: " + newLngLat);
			this.position = lngLat;
			const newLngLat = fromLonLat(lngLat);
			if (this.overlay != null) {
				this.overlay.setPosition(newLngLat);
			}
		}
	
		addOverlay() {
		// create a new Overlay, fill it with a QR code that is the
		// car's name, and add it to the map
			let div = document.createElement("div");
			div.style.background = this.color;
			div.style.padding = "4px 4px 0px 4px";
			div.id = this.divId;
			const qr = qrcode(2, 'L');
			let qrData = this.name
			if (qrData.length > 25) { qrData = qrData.slice(0,25)}
			qr.addData(qrData);
			qr.make();
			let img = qr.createImgTag();
			img = img.replace('width="66"','width="50"')
			img = img.replace('height="66"','height="50"')
			div.innerHTML = img;

			// div.style.width = "30px";
			document.body.appendChild(div);
			this.overlay = new Overlay({
				element: div,
				className: 'ol-overlay-container ol-selectable animation_move',
//				autoPan: true,
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
				cars[position.deviceId].updatePosition([
					position.longitude,
					position.latitude
				]);
			}
		});

		if (track && map) {
			let currentLocations = []
			for (const [key, value] of Object.entries(cars)) {
                                if (value.position != null ) {
					currentLocations.push(value.position);
				}
			}
			if (currentLocations.length == 0) { return }
			const ext = boundingExtent(currentLocations);
			const ext3857 = transformExtent(ext, get('EPSG:4326'), get('EPSG:3857'));
			map.getView().fit(ext3857, {
				duration: 1000,
				maxZoom: 19,
				padding: [100, 100, 100, 100],
			});
		}

		// if (map && markerSource.getFeatures().length > 0 && track) {
		// 	map.getView().fit(markerSource.getExtent(), {
		// 		duration: 1000,
		// 		maxZoom: 19,
		// 		padding: [100, 100, 100, 100],
		// 	});
		// }

	}
	
	$: setPositions(positions);
	
	function setCars(devices) {
		// update the position of all cars
		devices.forEach( function (device) {
			if (!cars[device.id]) {
				cars[device.id] = new Car(device.id, device.name)
			}
		});
	}

	$: setCars(devices);

	function setCarList(carRegistry) {
                carList = [];
		for (const [key, value] of Object.entries(carRegistry)) {
			carList.push(value);
		}
	}
	$: setCarList(cars)

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
		// fetch(baseUrl+"/api/session?token="+tcToken, {
		// fetch("http://parade.fbacchus.com:8082/api/session?token="+tcToken, {
		  fetch(baseUrl+"/api/session", {
			// dataType: 'json',
			// method: "POST",
			// headers: {
			// //   Authorization: `Token ${REVUE_API_KEY}`,
			// 	// Authorization: "Basic " + btoa(tcUser+":"+tcPassword),
			// 	// Accept: 'application/json',
			// 	  'Content-Type': 'application/x-www-form-urlencoded',
			// },
			// body: JSON.stringify({
			// // 	// contentType:'application/json',
			// // 	// accept:'application/json',
			// 	email: tcUser,
			// 	password: tcPassword
			// })
			// body: "email=admin&password=admin"
			// data: {
			// 	email: tcUser,
			// 	password: tcPassword,
			// 	// 'Accept': 'application/x-www-form-urlencoded',
			// 	// 'Content-Type': 'application/x-www-form-urlencoded',
			// 	// "Authorization": "Basic " + btoa(tcUser+":"+tcPassword)
			// }
		})
		.then((response) => { 
			// for (var pair of response.headers.entries()) {
			// 	console.log(pair[0]+ ': '+ pair[1]);
			// 	}
			// console.log(response.headers.get("set-cookie"))
			openWebSocket(response)
		 })
		.catch((error) => {
			console.log(error)
		});
	}
	
	function openWebSocket(response) {
			const socket = new WebSocket(socketUrl+"/api/socket", [],
			);
			socket.onmessage = function (event) {
					data = JSON.parse(event.data);
					if (data.positions) { positions = data.positions }
					if (data.devices) { devices = data.devices }
			}
			return socket
	}
	
	async function getWeird() {
		let fadeIn;
		let opacity = 0;
		let increment = .1
		while (true) {
			await sleep(40);
			if (opacity >= 1) { fadeIn = false; }
			if (opacity <= 0) { fadeIn = true; }
			opacity = fadeIn ? opacity + increment : opacity - increment;
			goldBasemap.setOpacity(opacity);
		}
	}

	async function promptLogin() {
		await sleep(5000)
		if (cars.length == 0) { isOpen = true};
	}
	
	onMount(() => {
		const mapView = new TrackMap("map");
		getWeird();
		startSession();
		// const res = openWebSocket();
		// console.log(res)
		promptLogin();
	});

	const rainbow = [
'Red',
'Cyan',
//'Silver',
//'Blue',
//'Gray',
//'DarkBlue',
//'Black',
'LightBlue',
'Orange',
//'Purple',
//'Brown',
'Yellow',
//'Maroon',
'Lime',
'Green',
'Magenta',
'Olive',
'Pink',
'Aquamarine',
'Red',
'White',
'Cyan',
'Silver',
//'Blue',
'Gray',
//'DarkBlue',
//'Black',
'LightBlue',
'Orange',
//'Purple',
//'Brown',
'Yellow',
'//Maroon',
'Lime',
'Green',
'Magenta',
'Olive',
'Pink',
'Aquamarine']
	
	</script>
	
	<main>
		<div id="map"></div>
		<nav>
			<label style="">track <input type="checkbox" bind:checked={track}></label>
			<!-- <label style="">give me qr <input type="checkbox" bind:checked={showQRs}></label> -->
			<button on:click={() => { showInfo = !showInfo}}>info</button>
		</nav>
		{#if showInfo}
		<div style="position:absolute; right:0; top: 25px;">
			{#if carList.length == 0}
			<div style="background:cyan; padding: 10px;">
				<h3>You may need to fake login:</h3>
				<p><a href="/enter">parade.fbacchus.com/enter</a></p>
                                <p>If things still aren't showing up (esp. on Safari),use:<p/>
				<p><a href="http://parade.fbacchus.com:8082" target="_blank">http://parade.fbacchus.com:8082</a></p>
                                <p>(email: fake, password: bacchus) then make your way back to this page.</p>
			</div>
			{:else}
			<ul style="margin:0px; list-style:none;">
				{#each carList as car, index}
				<li style="background: {car.color}">{car.name}: {car.position}</li>
				{/each}
				<li style="background:black; text-align:right;"><a style="color:white;" href="http://parade.fbacchus.com:8082">fake login/logout</a></li>
			</ul>
			{/if}
		</div>
		{/if}
		<div id="placeholder" style="position:absolute; right:0; top:100px;"></div>
<!-- 
		{#if isOpen}
		<div role="dialog" class="modal">
			<div class="contents">
			<h2>You may need to fake login:</h2>
			<p><a href="/enter">parade.fbacchus.com/enter</a></p>
			<div class="actions">
				<button on:click={() => {isOpen = false}}>OK</button>
			</div>
			</div>
		</div>
		{/if}
			 -->
	</main>
	
	<style>
	
	main {
		font-family: sans-serif;
	}
	
nav {
	background: lightgrey;
	display: flex;
	position: absolute;
	top: 0;
	width: 100%;
	height: 25px;
	justify-content: space-between;
}

button {
	color: #333;
	background-color: #f4f4f4;
	height: 100%;
	margin: 0px;
	padding: 5px;
	font-size: .75em;
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

	.modal {
    position: fixed;
    top: 0;
    bottom: 0;
    right: 0;
    left: 0;
    display: flex;
    justify-content: center;
    align-items: center;

    /* allow click-through to backdrop */
    pointer-events: none;
  }

  .contents {
    min-width: 240px;
    border-radius: 6px;
    padding: 16px;
    background: white;
    display: flex;
    flex-direction: column;
    justify-content: space-between;
    pointer-events: auto;
  }

  h2 {
    text-align: center;
    font-size: 24px;
  }

  p {
    text-align: center;
    margin-top: 16px;
  }

  .actions {
    margin-top: 32px;
    display: flex;
    justify-content: flex-end;
  }

  

</style>
