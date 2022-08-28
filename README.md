![image](https://user-images.githubusercontent.com/87375644/187082846-99a3502c-b75d-4b6e-bca1-292dd46a80ab.png)

<!DOCTYPE html>
<html lang="en">
	<head>
		<title>three.js vr - sculpt</title>
		<meta charset="utf-8">
		<meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
		<link type="text/css" rel="stylesheet" href="main.css">
	</head>
	<body>

		
		<script type="importmap">
			{
				"imports": {
					"three": "../build/three.module.js"
				}
			}
		</script>

		<script type="module">

			import * as THREE from 'three';
			import { OrbitControls } from './jsm/controls/OrbitControls.js';
			import { MarchingCubes } from './jsm/objects/MarchingCubes.js';
			import { VRButton } from './jsm/webxr/VRButton.js';

			let container;
			let camera, scene, renderer;
			let controller1, controller2;

			let controls, blob;

			let points = [];

			init();
			initBlob();
			animate();

			function init() {

				container = document.createElement( 'div' );
				document.body.appendChild( container );

				scene = new THREE.Scene();
				scene.background = new THREE.Color( 0x222222 );

				camera = new THREE.PerspectiveCamera( 50, window.innerWidth / window.innerHeight, 0.01, 50 );
				camera.position.set( 0, 1.6, 3 );

				controls = new OrbitControls( camera, container );
				controls.target.set( 0, 1.6, 0 );
				controls.update();

				const tableGeometry = new THREE.BoxGeometry( 0.5, 0.8, 0.5 );
				const tableMaterial = new THREE.MeshStandardMaterial( {
					color: 0x444444,
					roughness: 1.0,
					metalness: 0.0
				} );
				const table = new THREE.Mesh( tableGeometry, tableMaterial );
				table.position.y = 0.35;
				table.position.z = 0.85;
				scene.add( table );

				const floorGometry = new THREE.PlaneGeometry( 4, 4 );
				const floorMaterial = new THREE.MeshStandardMaterial( {
					color: 0x222222,
					roughness: 1.0,
					metalness: 0.0
				} );
				const floor = new THREE.Mesh( floorGometry, floorMaterial );
				floor.rotation.x = - Math.PI / 2;
				scene.add( floor );

				const grid = new THREE.GridHelper( 10, 20, 0x111111, 0x111111 );
				// grid.material.depthTest = false; // avoid z-fighting
				scene.add( grid );

				scene.add( new THREE.HemisphereLight( 0x888877, 0x777788 ) );

				const light = new THREE.DirectionalLight( 0xffffff );
				light.position.set( 0, 6, 0 );
				scene.add( light );

				//

				renderer = new THREE.WebGLRenderer( { antialias: true } );
				renderer.setPixelRatio( window.devicePixelRatio );
				renderer.setSize( window.innerWidth, window.innerHeight );
				renderer.outputEncoding = THREE.sRGBEncoding;
				renderer.xr.enabled = true;
				container.appendChild( renderer.domElement );

				document.body.appendChild( VRButton.createButton( renderer ) );

				// controllers

				function onSelectStart() {

					this.userData.isSelecting = true;

				}

				function onSelectEnd() {

					this.userData.isSelecting = false;

				}

				controller1 = renderer.xr.getController( 0 );
				controller1.addEventListener( 'selectstart', onSelectStart );
				controller1.addEventListener( 'selectend', onSelectEnd );
				controller1.userData.id = 0;
				scene.add( controller1 );

				controller2 = renderer.xr.getController( 1 );
				controller2.addEventListener( 'selectstart', onSelectStart );
				controller2.addEventListener( 'selectend', onSelectEnd );
				controller2.userData.id = 1;
				scene.add( controller2 );

				//

				const geometry = new THREE.CylinderGeometry( 0.01, 0.02, 0.08, 5 );
				geometry.rotateX( - Math.PI / 2 );
				const material = new THREE.MeshStandardMaterial( { flatShading: true } );
				const mesh = new THREE.Mesh( geometry, material );

				const pivot = new THREE.Mesh( new THREE.IcosahedronGeometry( 0.01, 3 ) );
				pivot.name = 'pivot';
				pivot.position.z = - 0.05;
				mesh.add( pivot );

				controller1.add( mesh.clone() );
				controller2.add( mesh.clone() );

				//

				window.addEventListener( 'resize', onWindowResize );

			}

			function initBlob() {

				/*
				let path = "textures/cube/SwedishRoyalCastle/";
				let format = '.jpg';
				let urls = [
					path + 'px' + format, path + 'nx' + format,
					path + 'py' + format, path + 'ny' + format,
					path + 'pz' + format, path + 'nz' + format
				];

				let reflectionCube = new CubeTextureLoader().load( urls );
				*/

				const material = new THREE.MeshStandardMaterial( {
					color: 0xffffff,
					// envMap: reflectionCube,
					roughness: 0.9,
					metalness: 0.0,
					transparent: true
				} );

				blob = new MarchingCubes( 64, material, false, false, 500000 );
				blob.position.y = 1;
				scene.add( blob );

				initPoints();

			}

			function initPoints() {

				points = [
					{ position: new THREE.Vector3(), strength: 0.04, subtract: 10 },
					{ position: new THREE.Vector3(), strength: - 0.08, subtract: 10 }
				];

			}

			function onWindowResize() {

				camera.aspect = window.innerWidth / window.innerHeight;
				camera.updateProjectionMatrix();

				renderer.setSize( window.innerWidth, window.innerHeight );

			}

			//

			function animate() {

				renderer.setAnimationLoop( render );

			}

			function transformPoint( vector ) {

				vector.x = ( vector.x + 1.0 ) / 2.0;
				vector.y = ( vector.y / 2.0 );
				vector.z = ( vector.z + 1.0 ) / 2.0;

			}

			function handleController( controller ) {

				const pivot = controller.getObjectByName( 'pivot' );

				if ( pivot ) {

					const id = controller.userData.id;
					const matrix = pivot.matrixWorld;

					points[ id ].position.setFromMatrixPosition( matrix );
					transformPoint( points[ id ].position );

					if ( controller.userData.isSelecting ) {

						const strength = points[ id ].strength / 2;

						const vector = new THREE.Vector3().setFromMatrixPosition( matrix );

						transformPoint( vector );

						points.push( { position: vector, strength: strength, subtract: 10 } );

					}

				}

			}

			function updateBlob() {

				blob.reset();

				for ( let i = 0; i < points.length; i ++ ) {

					const point = points[ i ];
					const position = point.position;

					blob.addBall( position.x, position.y, position.z, point.strength, point.subtract );

				}

			}

			function render() {

				handleController( controller1 );
				handleController( controller2 );

				updateBlob();

				renderer.render( scene, camera );

			}

		</script>
	</body>
</html>

  
  
  ---------------------------------------------------------------------------------------------------------------------------------------------------------------------
  
  ![image](https://user-images.githubusercontent.com/87375644/187082907-793ce801-b926-4306-a63a-f09b9f93c951.png)
  
  
<!DOCTYPE html>
<html lang="en">
	<head>
		<title>three.js vr - haptics</title>
		<meta charset="utf-8">
		<meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
		<link type="text/css" rel="stylesheet" href="main.css">
	</head>
	<body>

		<div id="info">
			<a href="https://threejs.org" target="_blank" rel="noopener">three.js</a> vr - haptics
		</div>

		<!-- Import maps polyfill -->
		<!-- Remove this when import maps will be widely supported -->
		<script async src="https://unpkg.com/es-module-shims@1.3.6/dist/es-module-shims.js"></script>

		<script type="importmap">
			{
				"imports": {
					"three": "../build/three.module.js"
				}
			}
		</script>

		<script type="module">

			import * as THREE from 'three';
			import { OrbitControls } from './jsm/controls/OrbitControls.js';
			import { VRButton } from './jsm/webxr/VRButton.js';
			import { XRControllerModelFactory } from './jsm/webxr/XRControllerModelFactory.js';

			let container;
			let camera, scene, renderer;
			let controller1, controller2;
			let controllerGrip1, controllerGrip2;
			const box = new THREE.Box3();

			const controllers = [];
			const oscillators = [];
			let controls, group;
			let audioCtx = null;

			// minor pentatonic scale, so whichever notes is striked would be more pleasant
			const musicScale = [ 0, 3, 5, 7, 10 ];

			init();
			animate();

			function initAudio() {

				if ( audioCtx !== null ) {

					return;

				}

				audioCtx = new ( window.AudioContext || window.webkitAudioContext )();
				function createOscillator() {

					// creates oscillator
					const oscillator = audioCtx.createOscillator();
					oscillator.type = 'sine'; // possible values: sine, triangle, square
					oscillator.start();
					return oscillator;

				}

				oscillators.push( createOscillator() );
				oscillators.push( createOscillator() );
				window.oscillators = oscillators;

			}

			function init() {

				container = document.createElement( 'div' );
				document.body.appendChild( container );

				scene = new THREE.Scene();
				scene.background = new THREE.Color( 0x808080 );

				camera = new THREE.PerspectiveCamera( 50, window.innerWidth / window.innerHeight, 0.1, 10 );
				camera.position.set( 0, 1.6, 3 );

				controls = new OrbitControls( camera, container );
				controls.target.set( 0, 1.6, 0 );
				controls.update();

				const floorGeometry = new THREE.PlaneGeometry( 4, 4 );
				const floorMaterial = new THREE.MeshStandardMaterial( {
					color: 0xeeeeee,
					roughness: 1.0,
					metalness: 0.0
				} );
				const floor = new THREE.Mesh( floorGeometry, floorMaterial );
				floor.rotation.x = - Math.PI / 2;
				floor.receiveShadow = true;
				scene.add( floor );

				scene.add( new THREE.HemisphereLight( 0x808080, 0x606060 ) );

				const light = new THREE.DirectionalLight( 0xffffff );
				light.position.set( 0, 6, 0 );
				light.castShadow = true;
				light.shadow.camera.top = 2;
				light.shadow.camera.bottom = - 2;
				light.shadow.camera.right = 2;
				light.shadow.camera.left = - 2;
				light.shadow.mapSize.set( 4096, 4096 );
				scene.add( light );

				group = new THREE.Group();
				group.position.z = - 0.5;
				scene.add( group );
				const BOXES = 10;

				for ( let i = 0; i < BOXES; i ++ ) {

					const intensity = ( i + 1 ) / BOXES;
					const w = 0.1;
					const h = 0.1;
					const minH = 1;
					const geometry = new THREE.BoxGeometry( w, h * i + minH, w );
					const material = new THREE.MeshStandardMaterial( {
						color: new THREE.Color( intensity, 0.1, 0.1 ),
						roughness: 0.7,
						metalness: 0.0
					} );

					const object = new THREE.Mesh( geometry, material );
					object.position.x = ( i - 5 ) * ( w + 0.05 );
					object.castShadow = true;
					object.receiveShadow = true;
					object.userData = {
						index: i + 1,
						intensity: intensity
					};

					group.add( object );

				}

				//

				renderer = new THREE.WebGLRenderer( { antialias: true } );
				renderer.setPixelRatio( window.devicePixelRatio );
				renderer.setSize( window.innerWidth, window.innerHeight );
				renderer.outputEncoding = THREE.sRGBEncoding;
				renderer.shadowMap.enabled = true;
				renderer.xr.enabled = true;
				container.appendChild( renderer.domElement );

				document.body.appendChild( VRButton.createButton( renderer ) );

				document.getElementById( 'VRButton' ).addEventListener( 'click', () => {

					initAudio();

				} );

				// controllers

				controller1 = renderer.xr.getController( 0 );
				scene.add( controller1 );

				controller2 = renderer.xr.getController( 1 );
				scene.add( controller2 );

				const controllerModelFactory = new XRControllerModelFactory();

				controllerGrip1 = renderer.xr.getControllerGrip( 0 );
				controllerGrip1.addEventListener( 'connected', controllerConnected );
				controllerGrip1.addEventListener( 'disconnected', controllerDisconnected );
				controllerGrip1.add( controllerModelFactory.createControllerModel( controllerGrip1 ) );
				scene.add( controllerGrip1 );

				controllerGrip2 = renderer.xr.getControllerGrip( 1 );
				controllerGrip2.addEventListener( 'connected', controllerConnected );
				controllerGrip2.addEventListener( 'disconnected', controllerDisconnected );
				controllerGrip2.add( controllerModelFactory.createControllerModel( controllerGrip2 ) );
				scene.add( controllerGrip2 );

				//

				window.addEventListener( 'resize', onWindowResize );

			}

			function controllerConnected( evt ) {

				controllers.push( {
					gamepad: evt.data.gamepad,
					grip: evt.target,
					colliding: false,
					playing: false
				} );

			}

			function controllerDisconnected( evt ) {

				const index = controllers.findIndex( o => o.controller === evt.target );
				if ( index !== - 1 ) {

					controllers.splice( index, 1 );

				}

			}

			function onWindowResize() {

				camera.aspect = window.innerWidth / window.innerHeight;
				camera.updateProjectionMatrix();

				renderer.setSize( window.innerWidth, window.innerHeight );

			}

			//

			function animate() {

				renderer.setAnimationLoop( render );

			}

			function handleCollisions() {

				for ( let i = 0; i < group.children.length; i ++ ) {

					group.children[ i ].collided = false;

				}

				for ( let g = 0; g < controllers.length; g ++ ) {

					const controller = controllers[ g ];
					controller.colliding = false;

					const { grip, gamepad } = controller;
					const sphere = {
						radius: 0.03,
						center: grip.position
					};

					const supportHaptic = 'hapticActuators' in gamepad && gamepad.hapticActuators != null && gamepad.hapticActuators.length > 0;

					for ( let i = 0; i < group.children.length; i ++ ) {

						const child = group.children[ i ];
						box.setFromObject( child );
						if ( box.intersectsSphere( sphere ) ) {

							child.material.emissive.b = 1;
							const intensity = child.userData.index / group.children.length;
							child.scale.setScalar( 1 + Math.random() * 0.1 * intensity );

							if ( supportHaptic ) {

								gamepad.hapticActuators[ 0 ].pulse( intensity, 100 );

							}

							const musicInterval = musicScale[ child.userData.index % musicScale.length ] + 12 * Math.floor( child.userData.index / musicScale.length );
							oscillators[ g ].frequency.value = 110 * Math.pow( 2, musicInterval / 12 );
							controller.colliding = true;
							group.children[ i ].collided = true;

						}

					}



					if ( controller.colliding ) {

						if ( ! controller.playing ) {

							controller.playing = true;
							oscillators[ g ].connect( audioCtx.destination );

						}

					} else {

						if ( controller.playing ) {

							controller.playing = false;
							oscillators[ g ].disconnect( audioCtx.destination );

						}

					}

				}

				for ( let i = 0; i < group.children.length; i ++ ) {

					let child = group.children[ i ];
					if ( ! child.collided ) {

						// reset uncollided boxes
						child.material.emissive.b = 0;
						child.scale.setScalar( 1 );

					}

				}

			}

			function render() {

				handleCollisions();

				renderer.render( scene, camera );

			}

		</script>
	</body>
</html>

  
  
  ---------------------------------------------------------------------------------------------------------------------------------------------------------------------
  
  
  ![image](https://user-images.githubusercontent.com/87375644/187082948-d62b7b11-b3a6-4e90-bc3b-c683103d36ed.png)

  
  
  <!DOCTYPE html>
<html lang="en">
	<head>
		<title>three.js vr - handinput</title>
		<meta charset="utf-8">
		<meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
		<link type="text/css" rel="stylesheet" href="main.css">
	</head>
	<body>

		<div id="info">
			<a href="https://threejs.org" target="_blank" rel="noopener">three.js</a> vr - handinput<br/>
			(Oculus Browser 15.1+)
		</div>

		<!-- Import maps polyfill -->
		<!-- Remove this when import maps will be widely supported -->
		<script async src="https://unpkg.com/es-module-shims@1.3.6/dist/es-module-shims.js"></script>

		<script type="importmap">
			{
				"imports": {
					"three": "../build/three.module.js"
				}
			}
		</script>

		<script type="module">

			import * as THREE from 'three';
			import { OrbitControls } from './jsm/controls/OrbitControls.js';
			import { VRButton } from './jsm/webxr/VRButton.js';
			import { XRControllerModelFactory } from './jsm/webxr/XRControllerModelFactory.js';
			import { XRHandModelFactory } from './jsm/webxr/XRHandModelFactory.js';

			let container;
			let camera, scene, renderer;
			let hand1, hand2;
			let controller1, controller2;
			let controllerGrip1, controllerGrip2;

			let controls;

			init();
			animate();

			function init() {

				container = document.createElement( 'div' );
				document.body.appendChild( container );

				scene = new THREE.Scene();
				scene.background = new THREE.Color( 0x444444 );

				camera = new THREE.PerspectiveCamera( 50, window.innerWidth / window.innerHeight, 0.1, 10 );
				camera.position.set( 0, 1.6, 3 );

				controls = new OrbitControls( camera, container );
				controls.target.set( 0, 1.6, 0 );
				controls.update();

				const floorGeometry = new THREE.PlaneGeometry( 4, 4 );
				const floorMaterial = new THREE.MeshStandardMaterial( { color: 0x222222 } );
				const floor = new THREE.Mesh( floorGeometry, floorMaterial );
				floor.rotation.x = - Math.PI / 2;
				floor.receiveShadow = true;
				scene.add( floor );

				scene.add( new THREE.HemisphereLight( 0x808080, 0x606060 ) );

				const light = new THREE.DirectionalLight( 0xffffff );
				light.position.set( 0, 6, 0 );
				light.castShadow = true;
				light.shadow.camera.top = 2;
				light.shadow.camera.bottom = - 2;
				light.shadow.camera.right = 2;
				light.shadow.camera.left = - 2;
				light.shadow.mapSize.set( 4096, 4096 );
				scene.add( light );

				//

				renderer = new THREE.WebGLRenderer( { antialias: true } );
				renderer.setPixelRatio( window.devicePixelRatio );
				renderer.setSize( window.innerWidth, window.innerHeight );
				renderer.outputEncoding = THREE.sRGBEncoding;
				renderer.shadowMap.enabled = true;
				renderer.xr.enabled = true;

				container.appendChild( renderer.domElement );

				document.body.appendChild( VRButton.createButton( renderer ) );

				// controllers

				controller1 = renderer.xr.getController( 0 );
				scene.add( controller1 );

				controller2 = renderer.xr.getController( 1 );
				scene.add( controller2 );

				const controllerModelFactory = new XRControllerModelFactory();
				const handModelFactory = new XRHandModelFactory();

				// Hand 1
				controllerGrip1 = renderer.xr.getControllerGrip( 0 );
				controllerGrip1.add( controllerModelFactory.createControllerModel( controllerGrip1 ) );
				scene.add( controllerGrip1 );

				hand1 = renderer.xr.getHand( 0 );
				hand1.add( handModelFactory.createHandModel( hand1 ) );

				scene.add( hand1 );

				// Hand 2
				controllerGrip2 = renderer.xr.getControllerGrip( 1 );
				controllerGrip2.add( controllerModelFactory.createControllerModel( controllerGrip2 ) );
				scene.add( controllerGrip2 );

				hand2 = renderer.xr.getHand( 1 );
				hand2.add( handModelFactory.createHandModel( hand2 ) );
				scene.add( hand2 );

				//

				const geometry = new THREE.BufferGeometry().setFromPoints( [ new THREE.Vector3( 0, 0, 0 ), new THREE.Vector3( 0, 0, - 1 ) ] );

				const line = new THREE.Line( geometry );
				line.name = 'line';
				line.scale.z = 5;

				controller1.add( line.clone() );
				controller2.add( line.clone() );

				//

				window.addEventListener( 'resize', onWindowResize );

			}

			function onWindowResize() {

				camera.aspect = window.innerWidth / window.innerHeight;
				camera.updateProjectionMatrix();

				renderer.setSize( window.innerWidth, window.innerHeight );

			}
			//

			function animate() {

				renderer.setAnimationLoop( render );

			}

			function render() {

				renderer.render( scene, camera );

			}

		</script>
	</body>
</html>

  
  ---------------------------------------------------------------------------------------------------------------------------------------------------------------------
  
  
  
  ![image](https://user-images.githubusercontent.com/87375644/187083011-6fdb7ea5-2398-45c4-a06e-a6178deb72d5.png)

  
  <!DOCTYPE html>
<html lang="en">
	<head>
		<title>three.js vr - dragging</title>
		<meta charset="utf-8">
		<meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
		<link type="text/css" rel="stylesheet" href="main.css">
	</head>
	<body>

		<div id="info">
			<a href="https://threejs.org" target="_blank" rel="noopener">three.js</a> vr - dragging
		</div>

		<!-- Import maps polyfill -->
		<!-- Remove this when import maps will be widely supported -->
		<script async src="https://unpkg.com/es-module-shims@1.3.6/dist/es-module-shims.js"></script>

		<script type="importmap">
			{
				"imports": {
					"three": "../build/three.module.js"
				}
			}
		</script>

		<script type="module">

			import * as THREE from 'three';
			import { OrbitControls } from './jsm/controls/OrbitControls.js';
			import { VRButton } from './jsm/webxr/VRButton.js';
			import { XRControllerModelFactory } from './jsm/webxr/XRControllerModelFactory.js';

			let container;
			let camera, scene, renderer;
			let controller1, controller2;
			let controllerGrip1, controllerGrip2;

			let raycaster;

			const intersected = [];
			const tempMatrix = new THREE.Matrix4();

			let controls, group;

			init();
			animate();

			function init() {

				container = document.createElement( 'div' );
				document.body.appendChild( container );

				scene = new THREE.Scene();
				scene.background = new THREE.Color( 0x808080 );

				camera = new THREE.PerspectiveCamera( 50, window.innerWidth / window.innerHeight, 0.1, 10 );
				camera.position.set( 0, 1.6, 3 );

				controls = new OrbitControls( camera, container );
				controls.target.set( 0, 1.6, 0 );
				controls.update();

				const floorGeometry = new THREE.PlaneGeometry( 4, 4 );
				const floorMaterial = new THREE.MeshStandardMaterial( {
					color: 0xeeeeee,
					roughness: 1.0,
					metalness: 0.0
				} );
				const floor = new THREE.Mesh( floorGeometry, floorMaterial );
				floor.rotation.x = - Math.PI / 2;
				floor.receiveShadow = true;
				scene.add( floor );

				scene.add( new THREE.HemisphereLight( 0x808080, 0x606060 ) );

				const light = new THREE.DirectionalLight( 0xffffff );
				light.position.set( 0, 6, 0 );
				light.castShadow = true;
				light.shadow.camera.top = 2;
				light.shadow.camera.bottom = - 2;
				light.shadow.camera.right = 2;
				light.shadow.camera.left = - 2;
				light.shadow.mapSize.set( 4096, 4096 );
				scene.add( light );

				group = new THREE.Group();
				scene.add( group );

				const geometries = [
					new THREE.BoxGeometry( 0.2, 0.2, 0.2 ),
					new THREE.ConeGeometry( 0.2, 0.2, 64 ),
					new THREE.CylinderGeometry( 0.2, 0.2, 0.2, 64 ),
					new THREE.IcosahedronGeometry( 0.2, 8 ),
					new THREE.TorusGeometry( 0.2, 0.04, 64, 32 )
				];

				for ( let i = 0; i < 50; i ++ ) {

					const geometry = geometries[ Math.floor( Math.random() * geometries.length ) ];
					const material = new THREE.MeshStandardMaterial( {
						color: Math.random() * 0xffffff,
						roughness: 0.7,
						metalness: 0.0
					} );

					const object = new THREE.Mesh( geometry, material );

					object.position.x = Math.random() * 4 - 2;
					object.position.y = Math.random() * 2;
					object.position.z = Math.random() * 4 - 2;

					object.rotation.x = Math.random() * 2 * Math.PI;
					object.rotation.y = Math.random() * 2 * Math.PI;
					object.rotation.z = Math.random() * 2 * Math.PI;

					object.scale.setScalar( Math.random() + 0.5 );

					object.castShadow = true;
					object.receiveShadow = true;

					group.add( object );

				}

				//

				renderer = new THREE.WebGLRenderer( { antialias: true } );
				renderer.setPixelRatio( window.devicePixelRatio );
				renderer.setSize( window.innerWidth, window.innerHeight );
				renderer.outputEncoding = THREE.sRGBEncoding;
				renderer.shadowMap.enabled = true;
				renderer.xr.enabled = true;
				container.appendChild( renderer.domElement );

				document.body.appendChild( VRButton.createButton( renderer ) );

				// controllers

				controller1 = renderer.xr.getController( 0 );
				controller1.addEventListener( 'selectstart', onSelectStart );
				controller1.addEventListener( 'selectend', onSelectEnd );
				scene.add( controller1 );

				controller2 = renderer.xr.getController( 1 );
				controller2.addEventListener( 'selectstart', onSelectStart );
				controller2.addEventListener( 'selectend', onSelectEnd );
				scene.add( controller2 );

				const controllerModelFactory = new XRControllerModelFactory();

				controllerGrip1 = renderer.xr.getControllerGrip( 0 );
				controllerGrip1.add( controllerModelFactory.createControllerModel( controllerGrip1 ) );
				scene.add( controllerGrip1 );

				controllerGrip2 = renderer.xr.getControllerGrip( 1 );
				controllerGrip2.add( controllerModelFactory.createControllerModel( controllerGrip2 ) );
				scene.add( controllerGrip2 );

				//

				const geometry = new THREE.BufferGeometry().setFromPoints( [ new THREE.Vector3( 0, 0, 0 ), new THREE.Vector3( 0, 0, - 1 ) ] );

				const line = new THREE.Line( geometry );
				line.name = 'line';
				line.scale.z = 5;

				controller1.add( line.clone() );
				controller2.add( line.clone() );

				raycaster = new THREE.Raycaster();

				//

				window.addEventListener( 'resize', onWindowResize );

			}

			function onWindowResize() {

				camera.aspect = window.innerWidth / window.innerHeight;
				camera.updateProjectionMatrix();

				renderer.setSize( window.innerWidth, window.innerHeight );

			}

			function onSelectStart( event ) {

				const controller = event.target;

				const intersections = getIntersections( controller );

				if ( intersections.length > 0 ) {

					const intersection = intersections[ 0 ];

					const object = intersection.object;
					object.material.emissive.b = 1;
					controller.attach( object );

					controller.userData.selected = object;

				}

			}

			function onSelectEnd( event ) {

				const controller = event.target;

				if ( controller.userData.selected !== undefined ) {

					const object = controller.userData.selected;
					object.material.emissive.b = 0;
					group.attach( object );

					controller.userData.selected = undefined;

				}


			}

			function getIntersections( controller ) {

				tempMatrix.identity().extractRotation( controller.matrixWorld );

				raycaster.ray.origin.setFromMatrixPosition( controller.matrixWorld );
				raycaster.ray.direction.set( 0, 0, - 1 ).applyMatrix4( tempMatrix );

				return raycaster.intersectObjects( group.children, false );

			}

			function intersectObjects( controller ) {

				// Do not highlight when already selected

				if ( controller.userData.selected !== undefined ) return;

				const line = controller.getObjectByName( 'line' );
				const intersections = getIntersections( controller );

				if ( intersections.length > 0 ) {

					const intersection = intersections[ 0 ];

					const object = intersection.object;
					object.material.emissive.r = 1;
					intersected.push( object );

					line.scale.z = intersection.distance;

				} else {

					line.scale.z = 5;

				}

			}

			function cleanIntersected() {

				while ( intersected.length ) {

					const object = intersected.pop();
					object.material.emissive.r = 0;

				}

			}

			//

			function animate() {

				renderer.setAnimationLoop( render );

			}

			function render() {

				cleanIntersected();

				intersectObjects( controller1 );
				intersectObjects( controller2 );

				renderer.render( scene, camera );

			}

		</script>
	</body>
</html>

  
  
  
  --------------------------------------------------------------------------------------------------------------------------------------------------------------------
  
  
  
  
  
  ![image](https://user-images.githubusercontent.com/87375644/187083303-645b93e4-bdce-490a-96bd-df071e8a083d.png)

  
  <!DOCTYPE html>
<html lang="en">
	<head>
		<title>three.js webgl - VSM Shadows example </title>
		<meta charset="utf-8">
		<meta name="viewport" content="width=device-width, user-scalable=no, minimum-scale=1.0, maximum-scale=1.0">
		<link type="text/css" rel="stylesheet" href="main.css">
	</head>
	<body>
		<div id="info">
		<a href="https://threejs.org" target="_blank" rel="noopener">three.js</a> - VSM Shadows example by <a href="https://github.com/supereggbert">Paul Brunt</a>
		</div>

		<!-- Import maps polyfill -->
		<!-- Remove this when import maps will be widely supported -->
		<script async src="https://unpkg.com/es-module-shims@1.3.6/dist/es-module-shims.js"></script>

		<script type="importmap">
			{
				"imports": {
					"three": "../build/three.module.js"
				}
			}
		</script>

		<script type="module">

			import * as THREE from 'three';

			import Stats from './jsm/libs/stats.module.js';
			import { GUI } from './jsm/libs/lil-gui.module.min.js';

			import { OrbitControls } from './jsm/controls/OrbitControls.js';

			let camera, scene, renderer, clock, stats;
			let dirLight, spotLight;
			let torusKnot, dirGroup;

			init();
			animate();

			function init() {

				initScene();
				initMisc();

				// Init gui
				const gui = new GUI();

				const config = {
					spotlightRadius: 4,
					spotlightSamples: 8,
					dirlightRadius: 4,
					dirlightSamples: 8
				};

				const spotlightFolder = gui.addFolder( 'Spotlight' );
				spotlightFolder.add( config, 'spotlightRadius' ).name( 'radius' ).min( 0 ).max( 25 ).onChange( function ( value ) {

					spotLight.shadow.radius = value;

				} );

				spotlightFolder.add( config, 'spotlightSamples', 1, 25, 1 ).name( 'samples' ).onChange( function ( value ) {

					spotLight.shadow.blurSamples = value;

				} );
				spotlightFolder.open();

				const dirlightFolder = gui.addFolder( 'Directional Light' );
				dirlightFolder.add( config, 'dirlightRadius' ).name( 'radius' ).min( 0 ).max( 25 ).onChange( function ( value ) {

					dirLight.shadow.radius = value;

				} );

				dirlightFolder.add( config, 'dirlightSamples', 1, 25, 1 ).name( 'samples' ).onChange( function ( value ) {

					dirLight.shadow.blurSamples = value;

				} );
				dirlightFolder.open();

				document.body.appendChild( renderer.domElement );
				window.addEventListener( 'resize', onWindowResize );

			}

			function initScene() {

				camera = new THREE.PerspectiveCamera( 45, window.innerWidth / window.innerHeight, 1, 1000 );
				camera.position.set( 0, 10, 30 );

				scene = new THREE.Scene();
				scene.background = new THREE.Color( 0x222244 );
				scene.fog = new THREE.Fog( 0x222244, 50, 100 );

				// Lights

				scene.add( new THREE.AmbientLight( 0x444444 ) );

				spotLight = new THREE.SpotLight( 0xff8888 );
				spotLight.angle = Math.PI / 5;
				spotLight.penumbra = 0.3;
				spotLight.position.set( 8, 10, 5 );
				spotLight.castShadow = true;
				spotLight.shadow.camera.near = 8;
				spotLight.shadow.camera.far = 200;
				spotLight.shadow.mapSize.width = 256;
				spotLight.shadow.mapSize.height = 256;
				spotLight.shadow.bias = - 0.002;
				spotLight.shadow.radius = 4;
				scene.add( spotLight );


				dirLight = new THREE.DirectionalLight( 0x8888ff );
				dirLight.position.set( 3, 12, 17 );
				dirLight.castShadow = true;
				dirLight.shadow.camera.near = 0.1;
				dirLight.shadow.camera.far = 500;
				dirLight.shadow.camera.right = 17;
				dirLight.shadow.camera.left = - 17;
				dirLight.shadow.camera.top	= 17;
				dirLight.shadow.camera.bottom = - 17;
				dirLight.shadow.mapSize.width = 512;
				dirLight.shadow.mapSize.height = 512;
				dirLight.shadow.radius = 4;
				dirLight.shadow.bias = - 0.0005;

				dirGroup = new THREE.Group();
				dirGroup.add( dirLight );
				scene.add( dirGroup );

				// Geometry

				const geometry = new THREE.TorusKnotGeometry( 25, 8, 75, 20 );
				const material = new THREE.MeshPhongMaterial( {
					color: 0x999999,
					shininess: 0,
					specular: 0x222222
				} );

				torusKnot = new THREE.Mesh( geometry, material );
				torusKnot.scale.multiplyScalar( 1 / 18 );
				torusKnot.position.y = 3;
				torusKnot.castShadow = true;
				torusKnot.receiveShadow = true;
				scene.add( torusKnot );

				const cylinderGeometry = new THREE.CylinderGeometry( 0.75, 0.75, 7, 32 );

				const pillar1 = new THREE.Mesh( cylinderGeometry, material );
				pillar1.position.set( 8, 3.5, 8 );
				pillar1.castShadow = true;
				pillar1.receiveShadow = true;

				const pillar2 = pillar1.clone();
				pillar2.position.set( 8, 3.5, - 8 );
				const pillar3 = pillar1.clone();
				pillar3.position.set( - 8, 3.5, 8 );
				const pillar4 = pillar1.clone();
				pillar4.position.set( - 8, 3.5, - 8 );

				scene.add( pillar1 );
				scene.add( pillar2 );
				scene.add( pillar3 );
				scene.add( pillar4 );

				const planeGeometry = new THREE.PlaneGeometry( 200, 200 );
				const planeMaterial = new THREE.MeshPhongMaterial( {
					color: 0x999999,
					shininess: 0,
					specular: 0x111111
				} );

				const ground = new THREE.Mesh( planeGeometry, planeMaterial );
				ground.rotation.x = - Math.PI / 2;
				ground.scale.multiplyScalar( 3 );
				ground.castShadow = true;
				ground.receiveShadow = true;
				scene.add( ground );

			}

			function initMisc() {

				renderer = new THREE.WebGLRenderer( { antialias: true } );
				renderer.setPixelRatio( window.devicePixelRatio );
				renderer.setSize( window.innerWidth, window.innerHeight );
				renderer.shadowMap.enabled = true;
				renderer.shadowMap.type = THREE.VSMShadowMap;

				// Mouse control
				const controls = new OrbitControls( camera, renderer.domElement );
				controls.target.set( 0, 2, 0 );
				controls.update();

				clock = new THREE.Clock();

				stats = new Stats();
				document.body.appendChild( stats.dom );

			}

			function onWindowResize() {

				camera.aspect = window.innerWidth / window.innerHeight;
				camera.updateProjectionMatrix();

				renderer.setSize( window.innerWidth, window.innerHeight );

			}

			function animate( time ) {

				requestAnimationFrame( animate );

				const delta = clock.getDelta();

				torusKnot.rotation.x += 0.25 * delta;
				torusKnot.rotation.y += 0.5 * delta;
				torusKnot.rotation.z += 1 * delta;

				dirGroup.rotation.y += 0.7 * delta;
				dirLight.position.z = 17 + Math.sin( time * 0.001 ) * 5;

				renderer.render( scene, camera );

				stats.update();

			}

		</script>
	</body>
</html>

  
  --------------------------------------------------------------------------------------------------------------------------------------------------------------------
  
  
  
  
