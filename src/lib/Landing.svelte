/*
  Description: The landing page for my-portfolio
*/

<script lang="ts">
  import { onMount } from "svelte";
  import * as THREE from "three";
  import { ImprovedNoise } from "three/examples/jsm/math/ImprovedNoise.js";
  import { EffectComposer } from "three/examples/jsm/postprocessing/EffectComposer.js";
  import { RenderPass } from 'three/examples/jsm/postprocessing/RenderPass.js';
  import { UnrealBloomPass } from 'three/examples/jsm/postprocessing/UnrealBloomPass.js';

  let canvas: HTMLCanvasElement;
  let scene: THREE.Scene;
  let camera: THREE.PerspectiveCamera;
  let renderer: THREE.WebGLRenderer;
  let cube: THREE.LineSegments;
  let stars: THREE.Points;
  let mouseX = 0;
  let mouseY = 0;
  let targetX = 0;
  let targetY = 0;
  let cubeSpinSpeed = 0.005;
  let targetSpinSpeed = 0.005;
  let isHovered = false;

  onMount(() => {
    // Scene setup
    scene = new THREE.Scene();
    scene.background = new THREE.Color(0x000009);

    // Camera
    camera = new THREE.PerspectiveCamera(100, window.innerWidth / window.innerHeight, 0.1, 1000);
    camera.position.z = 5;

    // Renderer
    renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.setPixelRatio(window.devicePixelRatio);

    // Postprocessing
    const composer = new EffectComposer(renderer);
    composer.addPass(new RenderPass(scene, camera));
    const bloomPass = new UnrealBloomPass( 
      new THREE.Vector2(window.innerWidth, window.innerHeight),
      0.8,  // strength
      0.6,  // radius
      0.1  // threshold
    );
    composer.addPass(bloomPass);

    // Lighting
    const ambientLight = new THREE.AmbientLight(0xffffff, 0.3);
    scene.add(ambientLight);

    // Cube
    const geometry = new THREE.BoxGeometry();
    const edges = new THREE.EdgesGeometry(geometry);
    const lineMaterial = new THREE.LineBasicMaterial({ color: 0xffffff, linewidth: 2 });
    cube = new THREE.LineSegments(edges, lineMaterial);
    scene.add(cube);

    // Star Generator
    const N = 10000; // More stars for galaxy effect
    const starGeometry = new THREE.BufferGeometry();
    const starPositions = new Float32Array(N * 3);
    const starColors = new Float32Array(N * 3);
    const v = new THREE.Vector3();
    const c = new THREE.Color();
    const perlin = new ImprovedNoise();

    let validStars = 0;
    let attempts = 0;

    while (validStars < N && attempts < N * 5) {
      // Distribution of strs
      v.randomDirection().setLength(1 + Math.random() * 15);

      // Equation for Spiral
      const angle = Math.atan2(v.z, v.x);
      const radius = Math.sqrt(v.x * v.x + v.z * v.z);
      const spiralAngle = angle + radius * 0.2;
      
      v.x = radius * Math.cos(spiralAngle);
      v.z = radius * Math.sin(spiralAngle);
      
      // Generates how the stars are flattened having a higher value makes it look like a disk
      v.y *= Math.pow(Math.random(), 1);
      v.x *= Math.pow(Math.random(), 1);

      // Perlin Noise to make it nebula-like
      const scale = 0.05;
      const noiseValue = 
        perlin.noise(v.x * scale, v.y * scale, v.z * scale) +
        0.5 * perlin.noise(v.x * scale * 2, v.y * scale * 2, v.z * scale * 2) +
        0.25 * perlin.noise(v.x * scale * 4, v.y * scale * 4, v.z * scale * 4);
      
      // Density falloff from center
      const centerDistance = Math.sqrt(v.x * v.x + v.y * v.y + v.z * v.z);
      const densityFalloff = Math.exp(-centerDistance * 0.05);
      
      if (noiseValue * densityFalloff > 0.3) {
        starPositions[validStars * 3] = v.x;
        starPositions[validStars * 3 + 1] = v.y;
        starPositions[validStars * 3 + 2] = v.z;

        // Colors of the stars
        const colorChoice = Math.random();
        if (colorChoice < 0.4) {
          // Blue
          c.setHSL(0.6 + 0.1 * Math.random(), 0.8, 0.6 + Math.random() * 0.3);
        } else if (colorChoice < 0.7) {
          // Purple
          c.setHSL(0.8 + 0.1 * Math.random(), 0.6, 0.5 + Math.random() * 0.3);
        } else if (colorChoice < 0.9) {
          // White
          c.setHSL(0, 0, 0.8 + Math.random() * 0.2);
        } else {
          // Red
          c.setHSL(0.05 + 0.1 * Math.random(), 0.8, 0.6 + Math.random() * 0.2);
        }
        
        starColors[validStars * 3] = c.r;
        starColors[validStars * 3 + 1] = c.g;
        starColors[validStars * 3 + 2] = c.b;

        validStars++;
      }
      attempts++;    
    }

    // Trims the array to the correct size
    const finalPositions = starPositions.slice(0, validStars * 3);
    const finalColors = starColors.slice(0, validStars * 3);

    starGeometry.setAttribute('position', new THREE.BufferAttribute(finalPositions, 3));
    starGeometry.setAttribute('color', new THREE.BufferAttribute(finalColors, 3));
    const baseColors = finalColors.slice(); // This line is used for twinkling effect

    const starMaterial = new THREE.PointsMaterial({
      vertexColors: true,
      size: 0.001,
      sizeAttenuation: true,
      transparent: true,
      opacity: 0.6,
      blending: THREE.AdditiveBlending,
      depthWrite: false,
    });

    stars = new THREE.Points(starGeometry, starMaterial);
    scene.add(stars);

    // Mouse hover effect
    const raycaster = new THREE.Raycaster();
    const mouse = new THREE.Vector2();
    
    const onMouseMove = (event: MouseEvent) => {
      // Parallax
      targetX = THREE.MathUtils.clamp((event.clientX / window.innerWidth - 0.5) * 2, -1, 1);
      targetY = THREE.MathUtils.clamp(-(event.clientY / window.innerHeight - 0.5) * 2, -1, 1);

      // Raycasting
      mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
      mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;
      
      // Check if hovering
      raycaster.setFromCamera(mouse, camera);
      const intersects = raycaster.intersectObject(cube);
      
      if (intersects.length > 0) {
        if (!isHovered) {
          isHovered = true;
          targetSpinSpeed = 0.05;
        }
      } else {
        if (isHovered) {
          isHovered = false;
          targetSpinSpeed = 0.005;
        }
      }
    };
    
    window.addEventListener('mousemove', onMouseMove);

    // Resize
    const onResize = () => {
      camera.aspect = window.innerWidth / window.innerHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(window.innerWidth, window.innerHeight);
    };

    window.addEventListener('resize', onResize);

    // Animation
    const animate = () => {
      requestAnimationFrame(animate);

      // Smooth follow (interpolation)
      mouseX += (targetX - mouseX) * 0.0005;
      mouseY += (targetY - mouseY) * 0.0005;

      const MAX_ROTATION = 0.4;
      scene.rotation.y = THREE.MathUtils.clamp(mouseX * 0.2, -MAX_ROTATION, MAX_ROTATION);
      scene.rotation.x = THREE.MathUtils.clamp(mouseY * 0.2, -MAX_ROTATION, MAX_ROTATION);

      // Gentle galaxy rotation, needs a fix because with parallax this fucks it up
      if (stars) {
        stars.rotation.y += 0.0002; // Slow galaxy rotation
        stars.rotation.y += mouseX * 0.001;
        stars.rotation.x += mouseY * 0.001;
      }

      cubeSpinSpeed += (targetSpinSpeed - cubeSpinSpeed) * 0.1;
      cube.rotation.x += cubeSpinSpeed;
      cube.rotation.y += cubeSpinSpeed;

      cube.position.y = Math.sin(Date.now() * 0.001) * 0.05;

      // Twinkling stars
      const t = Date.now() * 0.001;
      const colors = starGeometry.getAttribute('color');

      for (let i = 0; i < colors.count; i++) {
        const twinkle = 0.85 + 0.3 * Math.sin(t * 3.5 + i * 0.3); // Random phase shifts
        colors.setX(i, baseColors[i * 3] * twinkle);
        colors.setY(i, baseColors[i * 3 + 1] * twinkle);
        colors.setZ(i, baseColors[i * 3 + 2] * twinkle);
      }
      colors.needsUpdate = true;

      composer.render();
    };
    
    animate();

    // Cleanup
    return () => {
      window.removeEventListener('mousemove', onMouseMove);
      window.removeEventListener('resize', onResize);
    };
  });
</script>

<canvas bind:this={canvas} class="w-screen h-screen block"></canvas>

<style>
  canvas {
    display: block;
    position: fixed;
    top: 0;
    left: 0;
    z-index: 0;
  }
</style>
