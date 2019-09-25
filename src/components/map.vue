<template>
	<div id="map" ref="map">
		<header>
			<span>Coordinates</span>
			<span>X: {{ mouse.x}}</span>
			<span>Y: {{ mouse.y}}</span>
			<span>{{ this.plates.length - 1}} Blocks</span>
			<span>Color</span>
			<ul class="colors">
				<li v-for="(color, index) in colors"
				:key="index"
				:class="{'selected': selectedColor === color}"
				@click="selectedColor = colors[index]"
				v-bind:style="{backgroundColor: color}"
				></li>
			</ul>
			<span>Type</span>
			<ul class="type">
				<li v-for="(type, index) in blockTypes"
				:key="index" :class="{'selected': selectedType === type}"
				@click="selectedType = blockTypes[index]" v-text="type"
				></li>
			</ul>
		</header>
		<footer>
			<strong>controls</strong>
			<span v-for="(control, index) in controls" :key="index">
				{{ Object.keys(control)[0]  }} : {{ Object.values(control)[0] }}
			</span>
		</footer>
	</div>
</template>
<style lang="scss" scoped>
	#map {
		height: calc(100% - 81px - 37px);
		position: relative;
		canvas {
			z-index: 1;
		}
		footer, header {
			text-align: left;
			z-index: 2;
			color: #000;
			position: absolute;
			background-color: rgba(255,255,255,.75);
			border: 5px solid #333;
			padding: 0.5rem;
			bottom: 10px;
			left: 10px;
			span {
				display: block;
				margin-bottom: 0.5rem;
			}
		}

		header {
			top: 10px;
			left: 10px;
			bottom: initial;
			ul {
				list-style-type: none;
				margin: 0;
				padding: 0;
				li {
					display: inline-block;
					margin-right: 5px;
				}
				&.colors li {
					width: 1.5rem;
					height: 1.5rem;
					cursor: pointer;
					&.selected {
						box-shadow: 0px 0px 1px 3px inset #000;
					}
				}
				&.type li {
					cursor: pointer;
					border: 1px solid #555;
					padding: 0.5rem;
					&.selected {
						background-color: #555;
						color: #FFF;
					}
				}
			}
		}
	}
</style>
<script type="module">
import * as THREE from 'three';
import OrbitControls from 'three-orbitcontrols';

export default {
	mounted() {
		this.container = this.$refs.map;
		this.containerSize = this.container.getBoundingClientRect();
		this.mouse = new THREE.Vector3(0, 0, 1);
		this.selectedColor = this.colors[0];
		this.setupClock();
		this.setupScene();
		this.setupRenderer(this.container);
		this.setupCamera();
		this.calculateFOV();
		this.addGrid();
		this.addHelperPosition();
		this.addRaycaster();
		this.addControls();
		this.addStartupFloor();
		this.container.appendChild(this.Renderer.domElement);
		this.startAnimationFrames();
	},
	data() {
		return {
			FOV: {},
			scene: {},
			Renderer: {},
			camera: {},
			GridHelper: {},
			container: {},
			clock: {},
			helperPosition: {},
			containerSize: {},
			perspectiveFar: 101,
			perspectiveNear: 0,
			rollOverMaterial: {},
			rollOverMesh: {},
			mouse: {},
			gridFloor: {},
			plates: [],
			raycaster: {},
			controls: [],
			colors: [
				'#50822D',
				'#4B94CC',
				'#6B4F41',
				'#EDDDB4',
				'#7A7A7A',
				'#FAA381',
			],
			selectedColor: '',
			blockTypes: [
				'block',
				'slab',
				'column',
				'torch',
			],
			selectedType: 'block',
		};
	},
	methods: {
		addStartupFloor() {
			const material = new THREE.MeshToonMaterial({
				color: this.colors[0],
			});
			material.receiveShadow = true;
			const geometry = new THREE.BoxGeometry(16, 16, 16);
			const floorCoordinates = [
				new THREE.Vector3(-8, -8, 8),
				new THREE.Vector3(8, -8, 8),
			];

			floorCoordinates.forEach((coordinates) => {
				const floorMesh = new THREE.Mesh(geometry, material);
				floorMesh.position.copy(coordinates);
				this.scene.add(floorMesh);
			});
		},
		addControls() {
			this.controls.push(
				{ G: 'Toggle grid' },
				{ S: 'save game' },
				{ L: 'load latest saved game' },
			);
		},
		calculateFOV() {
			this.FOV = Math.tan(((Math.PI / 180) * this.camera.fov / 2));
		},
		setupClock() {
			this.clock = new THREE.Clock();
		},
		setupScene() {
			this.scene = new THREE.Scene();
			this.scene.background = new THREE.Color(0xffffff);
			this.scene.position.z = 0;
			this.scene.autoUpdate = true;
		},
		setupRenderer() {
			this.Renderer = new THREE.WebGLRenderer({ alpha: true });
			this.Renderer.setSize(this.containerSize.width, this.containerSize.height);
			this.Renderer.setPixelRatio(window.devicePixelRatio);
		},
		setupCamera() {
			/* this.camera = new THREE.OrthographicCamera(this.containerSize.width / -2, this.containerSize.width / 2, this.containerSize.height / 2, this.containerSize.height / -2, this.perspectiveNear, this.perspectiveFar), */
			this.camera = new THREE.PerspectiveCamera(45, window.innerWidth / window.innerHeight, 1, 10000);
			this.camera.position.set(0, 0, 10);
			this.camera.zoom = this.maxZoom / 2;
			this.scene.add(this.camera);
			this.camera.updateProjectionMatrix();
			this.mouse = new THREE.Vector2();
			const controls = new OrbitControls(this.camera, this.Renderer.domElement);
			controls.maxPolarAngle = Math.PI * 0.5;
			controls.minDistance = 1000;
			controls.maxDistance = 5000;

			const light = new THREE.HemisphereLight(0xffffbb, 0x080820, 0.5);
			light.position.set(0, 50, 0);
			this.scene.add(light);
			const directionalLight = new THREE.DirectionalLight(0xffffff, 0.75);
			directionalLight.position.set(-10, 20, 40);
			this.scene.add(directionalLight);
			directionalLight.position.multiplyScalar(30);
		},
		addGrid() {
			// GridHelper helper
			this.GridHelper = new THREE.GridHelper(1024, 64 /* 0xd6d6d6, 0xd7d7d7 */);
			this.GridHelper.rotation.x = 1.570795;
			this.GridHelper.z = 1;
			this.scene.add(this.GridHelper);

			const geometry = new THREE.PlaneBufferGeometry(1024, 1024);
			this.gridFloor = new THREE.Mesh(geometry, new THREE.MeshBasicMaterial({ visible: false }));
			this.scene.add(this.gridFloor);
			this.plates.push(this.gridFloor);
		},
		addHelperPosition() {
			const rollOverGeo = new THREE.BoxBufferGeometry(16, 16, 16);
			this.rollOverMaterial = new THREE.MeshToonMaterial({ color: 'blue', transparent: true, opacity: 0.5 });
			this.rollOverMesh = new THREE.Mesh(rollOverGeo, this.rollOverMaterial);
			this.scene.add(this.rollOverMesh);
		},
		addRaycaster() {
			this.raycaster = new THREE.Raycaster();
		},
		startAnimationFrames() {
			const animate = () => {
				requestAnimationFrame(animate);
				const self = this;
				const deltaTime = self.clock.getDelta();
				this.plates.forEach((plate) => {
					if (plate.hasOwnProperty('mixer')) {
						plate.mixer.update(deltaTime);
					}
				});
				this.render();
			};

			this.render();
			animate();
		},
		render() {
			this.Renderer.render(this.scene, this.camera);
		},
	},
};
</script>
