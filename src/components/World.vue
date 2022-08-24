<script setup>
import * as THREE from "three";
</script>

<template>
  <div class="wrapper">
    <div id="canvas"></div>
  </div>
</template>

<script>
let renderer, scene, camera;
let instancedMesh;
const numParticles = 100;
let mouseX = 0;
let mouseY = 0;
let windowHalfX = window.innerWidth / 2;
let windowHalfY = window.innerHeight / 2;
const cameraMoveSpeed = 0.1;
const parallaxCoeff = 0.3;
let parameters;
let materials = [];
export default {
  name: "World",
  data() {
    return {};
  },
  mounted() {
    this.init();
    this.animate();
  },
  methods: {
    addCCHO: function () {
      console.log("addCCHO");
      this.addSpheres();
      this.addCube();
    },
    addSpheres: function () {
      console.log("addSpheres");
      const sphereX = [-400, -200, 300];
      const spherePhi = [Math.PI / 3, Math.PI / 3, Math.PI * 2];

      for (let i = 0; i < 3; i++) {
        let sphereGeo = new THREE.SphereGeometry(150, 50, 50, 0, spherePhi[i]);
        sphereGeo.translate(sphereX[i], 0, 0);
        let randCol = () => Math.floor(Math.random() * 16777215).toString(16);
        let material = new THREE.MeshBasicMaterial({ color: "#" + randCol() });
        let sphere = new THREE.Mesh(sphereGeo, material);
        scene.add(sphere);
      }
    },
    addCube: function () {
      console.log("addCube");
      const boxGeo = new THREE.BoxGeometry(200, 200, 50);
      boxGeo.translate(0, 0, 0);
      const material = new THREE.MeshBasicMaterial({
        color: Math.floor(Math.random() * 16777215).toString(16),
      });
      const cube = new THREE.Mesh(boxGeo, material);
      scene.add(cube);
    },
    init: function () {
      camera = new THREE.PerspectiveCamera(
        75,
        window.innerWidth / window.innerHeight,
        1,
        2000
      );
      camera.position.z = 1000;

      scene = new THREE.Scene();
      scene.fog = new THREE.FogExp2(0x000000, 0.0008);

      const geometry = new THREE.BufferGeometry();
      const vertices = [];

      for (let i = 0; i < numParticles; i++) {
        const x = Math.random() * 2000 - 1000;
        const y = Math.random() * 2000 - 1000;
        const z = Math.random() * 2000 - 1000;

        vertices.push(x, y, z);
      }

      geometry.setAttribute(
        "position",
        new THREE.Float32BufferAttribute(vertices, 3)
      );

      parameters = [[[1.0, 0.2, 0.5], "yo", 20]];

      for (let i = 0; i < parameters.length; i++) {
        const color = parameters[i][0];
        const sprite = parameters[i][1];
        const size = parameters[i][2];

        materials[i] = new THREE.PointsMaterial({
          size: size,
          //   map: sprite,
          blending: THREE.AdditiveBlending,
          depthTest: false,
          transparent: true,
        });
        materials[i].color.setHSL(color[0], color[1], color[2]);

        const particles = new THREE.Points(geometry, materials[i]);

        particles.rotation.x = Math.random() * 6;
        particles.rotation.y = Math.random() * 6;
        particles.rotation.z = Math.random() * 6;

        scene.add(particles);
      }

      this.addCCHO();

      renderer = new THREE.WebGLRenderer();
      renderer.setPixelRatio(window.devicePixelRatio);
      renderer.setSize(window.innerWidth, window.innerHeight);
      document.body.appendChild(renderer.domElement);

      document.body.style.touchAction = "none";
      document.body.addEventListener("pointermove", this.onPointerMove);

      //
      window.addEventListener("resize", this.onWindowResize);
    },
    animate: function () {
      requestAnimationFrame(this.animate);
      this.renderScene();
    },
    renderScene: function () {
      const time = Date.now() * 0.00005;

      camera.position.x += (mouseX - camera.position.x) * cameraMoveSpeed;
      camera.position.y += (-mouseY - camera.position.y) * cameraMoveSpeed;

      camera.lookAt(scene.position);

      for (let i = 0; i < scene.children.length; i++) {
        const object = scene.children[i];

        if (object instanceof THREE.Points) {
          object.rotation.y = time * (i < 4 ? i + 1 : -(i + 1));
          //   object.rotation.x =
          //   object.rotation.y = time;
          //   object.position.x += (Math.random() * 2 - 1) * 5;
          //   object.position.y += (Math.random() * 2 - 1) * 5;
          //   object.position.z += (Math.random() * 2 - 1) * 5;
          //TODO: Points is a mesh that contains the vertices. We need to move the vertices and update the mesh.
        }
      }

      for (let i = 0; i < materials.length; i++) {
        const color = parameters[i][0];

        const h = ((360 * (color[0] + time)) % 360) / 360;
        materials[i].color.setHSL(h, color[1], color[2]);
      }

      renderer.render(scene, camera);
    },
    onPointerMove: function (event) {
      if (event.isPrimary === false) return;

      mouseX = event.clientX - windowHalfX;
      mouseY = event.clientY - windowHalfY;
      mouseX *= parallaxCoeff;
      mouseY *= parallaxCoeff;
    },
    onWindowResize: function () {
      windowHalfX = window.innerWidth / 2;
      windowHalfY = window.innerHeight / 2;

      camera.zoom = window.innerWidth / screen.availWidth;
      camera.aspect = window.innerWidth / window.innerHeight;
      camera.updateProjectionMatrix();

      renderer.setSize(window.innerWidth, window.innerHeight);
    },
  },
};
</script>

<style scoped>
.wrapper {
  display: flex;
  flex-direction: column;
  justify-content: space-around;
  align-items: center;
}
</style>
