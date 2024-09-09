<script>
	import { onMount } from 'svelte';
	import * as THREE from 'three';
	import TouchTexture from '@lib/water-distortion/components/TouchTexture';

	import { EffectComposer, RenderPass, EffectPass /* , OutputPass */ } from 'postprocessing';

	/* import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
	import { RenderPass } from 'three/addons/postprocessing/RenderPass.js'; */
	//import { EffectPass } from 'three/addons/postprocessing/EffectPass.js';
	import { OutputPass } from 'three/addons/postprocessing/OutputPass.js';

	import { WaterEffect } from '@lib/water-distortion/components/WaterEffect';
	import { Planes } from '@lib/water-distortion/components/Planes';

	import image1 from '@lib/water-distortion/images/image-1.jpg';
	import image2 from '@lib/water-distortion/images/image-2.jpg';
	import image3 from '@lib/water-distortion/images/image-3.jpg';

	const images = [image1, image2, image3];

	class Loader {
		constructor() {
			this.items = [];
			this.loaded = [];
		}
		begin(name) {
			this.items.push(name);
		}
		end(name) {
			this.loaded.push(name);
			if (this.loaded.length === this.items.length) {
				this.onComplete();
			}
		}
		onComplete() {}
	}

	let renderer,
		camera,
		scene,
		clock,
		composer,
		raycaster,
		hitObjects,
		touchTexture,
		assets,
		data,
		subjects,
		disposed,
		loader,
		waterEffect,
		mouse,
		video;

	function loadAssets() {
		return new Promise((resolve, reject) => {
			// loadTextAssets(assets, loader);
			console.log('subjects', subjects);

			subjects.forEach((subject) => subject.load(loader));

			loader.onComplete = () => {
				resolve();
			};
		});
	}

	function initComposer() {
		const renderPass = new RenderPass(scene, camera);
		waterEffect = new WaterEffect({ texture: touchTexture.texture });
		const waterPass = new EffectPass(camera, waterEffect);
		//const outputPass = new OutputPass();

		waterPass.renderToScreen = true;
		renderPass.renderToScreen = false;

		composer.addPass(renderPass);
		composer.addPass(waterPass);
		//composer.addPass(outputPass);
	}

	function init() {
		touchTexture.initTexture();

		// const textGeometry2 = createGeometry({
		//   font: assets.font,
		//   align: "center",
		//   width: 600,
		//   text: Array.from({ length: 100 }, () => "water").join(" ")
		// });
		// const textMaterial2 = createTextMaterial(assets.glyphs, {
		//   color: "rgba(20,20,20,1.0)"
		// });
		// const textMesh2 = new THREE.Mesh(textGeometry2, textMaterial2);
		// scale = 0.1;
		// console.log(textGeometry2.layout);
		// textMesh2.scale.x = scale;
		// textMesh2.scale.y = -scale;
		// textMesh2.position.z += -0.1;
		// textMesh2.position.x = (-textGeometry2.layout.width / 2) * scale;
		// textMesh2.position.y =
		//   (-textGeometry2.layout.height / 2) * scale +
		//   (-textGeometry2.layout.lineHeight / 4) * scale;
		// scene.add(textMesh2);

		addHitPlane();
		subjects.forEach((subject) => subject.init());
		initComposer();

		// addImagePlane();

		tick();

		window.addEventListener('resize', onResize);
		window.addEventListener('mousemove', onMouseMove);
		window.addEventListener('touchmove', onTouchMove);
	}

	function onTouchMove(ev) {
		const touch = ev.targetTouches[0];
		onMouseMove({ clientX: touch.clientX, clientY: touch.clientY });
	}

	function onMouseMove(ev) {
		mouse = {
			x: ev.clientX / window.innerWidth,
			y: 1 - ev.clientY / window.innerHeight
		};
		touchTexture.addTouch(mouse);

		raycaster.setFromCamera(
			{
				x: (ev.clientX / window.innerWidth) * 2 - 1,
				y: -(ev.clientY / window.innerHeight) * 2 + 1
			},
			camera
		);
		// var intersections = raycaster.intersectObjects(hitObjects);
		// if (intersections.length > 0) {
		//   const intersect = intersections[0];
		//   touchTexture.addTouch(intersect.uv);
		// }
		subjects.forEach((subject) => {
			if (subject.onMouseMove) {
				subject.onMouseMove(ev);
			}
		});
	}

	function addImagePlane() {
		const viewSize = getViewSize();

		let width = viewSize.width / 4.5;

		const geometry = new THREE.PlaneGeometry(width, viewSize.height * 0.8, 1, 1);
		let x = -viewSize.width / 2 + width / 2 + viewSize.width / 5 / 1.5;

		let space = (viewSize.width - (viewSize.width / 5 / 1.5) * 2 - width) / 2;
		for (let i = 0; i < 3; i++) {
			const material = new THREE.MeshBasicMaterial({ color: 0x484848 });
			const mesh = new THREE.Mesh(geometry, material);
			mesh.position.x += x + i * space;
			scene.add(mesh);
		}
	}

	function addHitPlane() {
		const viewSize = getViewSize();
		const geometry = new THREE.PlaneGeometry(viewSize.width, viewSize.height, 1, 1);
		const material = new THREE.MeshBasicMaterial();
		const mesh = new THREE.Mesh(geometry, material);

		hitObjects.push(mesh);
	}
	function getViewSize() {
		const fovInRadians = (camera.fov * Math.PI) / 180;
		const height = Math.abs(camera.position.z * Math.tan(fovInRadians / 2) * 2);

		return { width: height * camera.aspect, height };
	}
	function dispose() {
		disposed = true;
		window.removeEventListener('resize', onResize);
		window.removeEventListener('mousemove', onMouseMove);
		scene.children.forEach((child) => {
			child.material.dispose();
			child.geometry.dispose();
		});
		if (assets.glyphs) assets.glyphs.dispose();

		hitObjects.forEach((child) => {
			if (child) {
				if (child.material) child.material.dispose();
				if (child.geometry) child.geometry.dispose();
				// child.dispose();
			}
		});
		if (touchTexture) touchTexture.texture.dispose();
		scene.dispose();
		renderer.dispose();
		composer.dispose();
	}
	function update() {
		touchTexture.update();
		subjects.forEach((subject) => {
			subject.update();
		});
	}
	function render() {
		// renderer.render(scene, camera);
		composer.render(clock.getDelta());
	}
	function tick() {
		if (disposed) return;
		render();
		update();
		requestAnimationFrame(tick);
	}
	function onResize() {
		camera.aspect = window.innerWidth / window.innerHeight;
		camera.updateProjectionMatrix();

		composer.setSize(window.innerWidth, window.innerHeight);
		subjects.forEach((subject) => {
			subject.onResize(window.innerWidth, window.innerHeight);
		});
	}
	function initialize() {
		renderer = new THREE.WebGLRenderer({
			antialias: false
		});
		renderer.setSize(window.innerWidth, window.innerHeight);
		renderer.setPixelRatio(window.devicePixelRatio);

		composer = new EffectComposer(renderer);

		document.body.append(renderer.domElement);
		renderer.domElement.id = 'webGLApp';

		camera = new THREE.PerspectiveCamera(45, window.innerWidth / window.innerHeight, 0.1, 10000);
		camera.position.z = 50;
		disposed = false;
		scene = new THREE.Scene();

		const light = new THREE.DirectionalLight(0xffffff, 3);
		light.position.set(0.5, 1, 1).normalize();
		scene.add(light);

		scene.background = new THREE.Color(0x161624);

		clock = new THREE.Clock();

		assets = {};
		raycaster = new THREE.Raycaster();
		hitObjects = [];

		touchTexture = new TouchTexture();

		data = {
			text: ["DON'T", 'LOOK', 'BACK'],
			images: images
		};

		subjects = [
			new Planes(
				{
					getViewSize,
					scene,
					raycaster,
					onPlaneHover: (plane) => {
						/* console.log('plane', plane); */
					}
				},
				images,
				video
			)
		];
		// subjects = [];

		tick = tick.bind(this);
		onResize = onResize.bind(this);
		onMouseMove = onMouseMove.bind(this);
		onTouchMove = onTouchMove.bind(this);

		init = init.bind(this);
		loader = new Loader();
		loadAssets().then(init);

		/* video.play();
		video.addEventListener('play', function () {
			this.currentTime = 3;
		}); */

		/* <video bind:this={video} loop crossOrigin="anonymous" style="display:none;" muted>
	<source src="videos/sintel.mp4" type="video/mp4;" />
</video> */
	}

	onMount(() => {
		initialize();
	});
</script>
