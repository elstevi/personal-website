<!--
.. title: Cross Slide Nut
.. slug: cross-slide-nut
.. date: 2025-10-26 13:10:25 UTC-07:00
.. tags: Utilathe 9" Series 1000, B-65668, lathe
.. category: machining, lathe
.. link: 
.. description: 
.. type: text
-->
<style>
body { margin: 0; overflow: hidden; }
canvas { display: block; }
</style>
<body>

Here are models and drawings for the Cross Slide Nut (part B-65749) for the Standard Modern 9" Utilathe (Series 1000).

<!-- TEASER_END -->
<a href="/lathe/Cross Slide Nut (B-65749).pdf">Drawing</a>
<br/>
<a href="/lathe/Cross Slide Nut (B-65749).obj">Model</a>

<embed src="/lathe/Cross Slide Nut (B-65749).pdf" type="application/pdf" width="100%" height="600px" />

<div id="viewer-container" style="width: 100%; height: 80vh;"></div>

<script type="module"> // this is all chatgpt
  import * as THREE from 'https://esm.run/three@0.160.0';
  import { OBJLoader } from 'https://esm.run/three@0.160.0/examples/jsm/loaders/OBJLoader.js';
  import { OrbitControls } from 'https://esm.run/three@0.160.0/examples/jsm/controls/OrbitControls.js';

  const container = document.getElementById('viewer-container');

  const scene = new THREE.Scene();
  scene.background = new THREE.Color(0x303030); // lighter gray

  // Camera — zoomed out and elevated
  const camera = new THREE.PerspectiveCamera(
    150,
    container.clientWidth / container.clientHeight,
    0.1,
    1000
  );
  camera.position.set(6,6,6);

  // Renderer
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  renderer.outputColorSpace = THREE.SRGBColorSpace;
  container.appendChild(renderer.domElement);

  // Controls
  const controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.dampingFactor = 0.05;

  // Lights — brighter setup
  scene.add(new THREE.AmbientLight(0xffffff, 0.8));
  const light = new THREE.DirectionalLight(0xffffff, 1.5);
  light.position.set(10, 15, 10);
  scene.add(light);

  // OBJ Loader — use encoded URL for file name with spaces and parentheses
  const loader = new OBJLoader();
  loader.load(
    '/lathe/Cross Slide Nut (B-65749).obj',
    (object) => {
      object.scale.set(2,2,2);
      scene.add(object);
    },
    (xhr) => console.log(`${(xhr.loaded / xhr.total * 100).toFixed(0)}% loaded`),
    (err) => console.error('Error loading OBJ:', err)
  );

  // Handle window resize
  window.addEventListener('resize', () => {
    camera.aspect = container.clientWidth / container.clientHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(container.clientWidth, container.clientHeight);
  });

  // Animation loop
  function animate() {
    requestAnimationFrame(animate);
    controls.update();
    renderer.render(scene, camera);
  }
  animate();
</script>
