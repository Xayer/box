<script>
import * as THREE from 'three';
import map from './map.vue';

export default {
	extends: map,
	mounted() {
		window.onload = this.addListeners();
		/* this.loadGame(); */
		// auto save every 5minutes
		setInterval(() => {
			this.saveGame();
		}, 300000);
	},
	data() {
		return {
			minZoom: 20,
			maxZoom: 10,
			gridEnabled: true,
		};
	},
	methods: {
		addListeners() {
			this.$refs.map.addEventListener('mousedown', this.mouseDown, false);
			window.addEventListener('mouseup', this.mouseUp, false);
			this.$refs.map.addEventListener('wheel', this.zoom, false);
			this.$refs.map.addEventListener('mousedown', this.click, false);
			this.$refs.map.addEventListener('resize', this.onWindowResize, false);
			document.addEventListener('mousemove', this.mouseMove, false);
			document.addEventListener('keydown', this.keyDown, false);
		},
		mouseUp() {
			window.removeEventListener('mousemove', this.divMove, true);
		},
		mouseDown() {
			window.addEventListener('mousemove', this.divMove, true);
		},
		divMove(event) {
			this.camera.position.x += -event.movementX / this.camera.zoom;
			this.camera.position.y += event.movementY / this.camera.zoom;
		},
		zoom(event) {
			event.preventDefault();
			const navigationAmount = 2;
			if (event.deltaY > 0) {
				if (this.camera.zoom === this.minZoom) {
					return;
				}
				this.camera.zoom -= navigationAmount;
			} else {
				if (this.camera.zoom === this.maxZoom) {
					return;
				}
				this.camera.zoom += navigationAmount;
			}
			this.camera.updateProjectionMatrix();
		},
		onWindowResize() {
			this.containerSize = this.container.getBoundingClientRect();
			this.camera.aspect = this.containerSize.width / this.containerSize.height;
			// adjust the FOV
			this.camera.fov = (360 / Math.PI) * Math.atan(this.FOV * (this.containerSize.width / this.containerSize.height));
			this.camera.lookAt(new THREE.Vector3(0, 0, 0));

			this.Renderer.setSize(this.containerSize.width, this.containerSize.height);
			this.camera.updateProjectionMatrix();
			this.render();
		},
		mouseMove(event) {
			event.preventDefault();
			this.mouse.set((event.clientX / this.containerSize.width) * 2 - 1, -(event.clientY / this.containerSize.height) * 2 + 1);
			this.raycaster.setFromCamera(this.mouse, this.camera);
			const intersects = this.raycaster.intersectObjects(this.plates);
			if (intersects.length > 0) {
				const intersect = intersects[0];
				this.rollOverMesh.position.copy(intersect.point).add(intersect.face.normal);
				this.rollOverMesh.position.divideScalar(16).floor().multiplyScalar(16).addScalar(8);
				this.camera.updateProjectionMatrix();
			}
			this.render();
		},
		click(event) {
			event.preventDefault();

			this.raycaster.setFromCamera(this.mouse, this.camera);
			const intersects = this.raycaster.intersectObjects(this.plates);
			if (intersects.length > 0) {
				// if left clicking, delete
				const intersect = intersects[0];
				if (event.button === 0) {
					if (intersect.object !== this.gridFloor) {
						const currentPlate = this.plates[this.plates.indexOf(intersect.object)];
						this.plates[this.plates.indexOf(intersect.object)].mixer = new THREE.AnimationMixer(intersect.object);
						const fadeAnimation = new THREE.NumberKeyframeTrack('.material.opacity', [0, 1], [1, 0]);
						const animationClip = new THREE.AnimationClip(null, 0.25, [fadeAnimation]);
						const animationAction = currentPlate.mixer.clipAction(animationClip);
						animationAction.setLoop(THREE.LoopOnce);
						animationAction.play();
						setTimeout(() => {
							if (intersect.object !== this.gridFloor) {
								this.scene.remove(intersect.object);
								this.plates.splice(this.plates.indexOf(intersect.object), 1);
							}
						}, 300);
					}
				// right clicking, add block
				} else if (event.button === 2) {
					this.createBlock(intersect);
				}
			}
			this.camera.updateProjectionMatrix();
		},
		createBlock(intersect) {
			let geometry = new THREE.BoxBufferGeometry(16, 16, 16);
			if (this.selectedType === 'slab') {
				geometry = new THREE.BoxBufferGeometry(16, 8, 16);
			}
			if (this.selectedType === 'torch') {
				geometry = new THREE.BoxBufferGeometry(8, 16, 8);
			}
			const color = new THREE.Color(this.selectedColor);
			const material = new THREE.MeshPhongMaterial({ color });
			const voxel = new THREE.Mesh(geometry, material);
			voxel.position.copy(intersect.point).add(intersect.face.normal);
			voxel.position.divideScalar(16).floor().multiplyScalar(16).addScalar(8);
			this.scene.add(voxel);
			this.plates.push(voxel);
			this.render();
		},
		keyDown(event) {
			if (event.key === 'g') {
				this.toggleGrid();
			}
			if (event.key === 's') {
				this.saveGame();
			}
			if (event.key === 'l') {
				this.loadGame();
			}
		},
		toggleGrid() {
			this.gridEnabled = !this.gridEnabled;
			if (this.gridEnabled) {
				this.scene.add(this.GridHelper);
			} else {
				this.scene.remove(this.GridHelper);
			}
		},
		saveGame() {
			const dateNow = Date.now();
			const voxelsToSave = localStorage.voxels ? JSON.parse(localStorage.voxels) : {};
			const allVoxelsInCurrentScene = [];
			this.plates.forEach((item) => {
				allVoxelsInCurrentScene.push(item.toJSON());
			});
			voxelsToSave[dateNow] = this.plates;
			localStorage.voxels = JSON.stringify(voxelsToSave);
		},
		loadGame() {
			const localHostData = JSON.parse(localStorage.voxels);
			const allSaves = Object.keys(localHostData);
			const lastSave = allSaves[allSaves.length - 1];
			const loadedData = localHostData[lastSave];
			console.log(lastSave, allSaves.length, allSaves);
			loadedData.forEach((item, index) => {
				if (index === 0) {
					// first item is always the grid, so skip it
					return;
				}
				const loader = new THREE.ObjectLoader();

				loader.parse(
					item,
					// onLoad callback
					// Here the loaded data is assumed to be an object
					(obj) => {
						// Add the loaded object to the scene#
						this.plates.push(obj);
						this.scene.add(obj);
						console.log();
						/* this.scene.add( obj ); */
					},

					// onProgress callback
					(xhr) => {
						console.log(`${xhr.loaded / xhr.total * 100}% loaded`);
					},

					// onError callback
					(err) => {
						console.error('An error happened', err);
					},
				);
			});
		},
	},
};
</script>
