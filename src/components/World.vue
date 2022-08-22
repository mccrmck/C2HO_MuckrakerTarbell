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
let mouseX = 0;
let mouseY = 0;
let windowHalfX = window.innerWidth / 2;
let windowHalfY = window.innerHeight / 2;
let cameraMoveSpeed = 0.1;
let parallaxCoeff = 0.05;
export default {
  name: "World",
  data() {
    return {
      parameters: undefined,
      materials: [],
    };
  },
  mounted() {
    this.init();
    this.animate();
  },
  methods: {
    counter: function () {
      this.count++;
    },
    init: function () {
      console.log("init");

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

      const textureLoader = new THREE.TextureLoader();

      const sprite1 = textureLoader.load("snowflake1.png");
      const sprite2 = textureLoader.load("snowflake1.png");
      const sprite3 = textureLoader.load("snowflake1.png");
      const sprite4 = textureLoader.load("snowflake1.png");
      const sprite5 = textureLoader.load("snowflake1.png");

      for (let i = 0; i < 10000; i++) {
        const x = Math.random() * 2000 - 1000;
        const y = Math.random() * 2000 - 1000;
        const z = Math.random() * 2000 - 1000;

        vertices.push(x, y, z);
      }

      geometry.setAttribute(
        "position",
        new THREE.Float32BufferAttribute(vertices, 3)
      );

      this.parameters = [
        [[1.0, 0.2, 0.5], sprite2, 20],
        [[0.95, 0.1, 0.5], sprite3, 15],
        [[0.9, 0.05, 0.5], sprite1, 10],
        [[0.85, 0, 0.5], sprite5, 8],
        [[0.8, 0, 0.5], sprite4, 5],
      ];

      for (let i = 0; i < this.parameters.length; i++) {
        const color = this.parameters[i][0];
        const sprite = this.parameters[i][1];
        const size = this.parameters[i][2];

        this.materials[i] = new THREE.PointsMaterial({
          size: size,
          map: sprite,
          blending: THREE.AdditiveBlending,
          depthTest: false,
          transparent: true,
        });
        this.materials[i].color.setHSL(color[0], color[1], color[2]);

        const particles = new THREE.Points(geometry, this.materials[i]);

        particles.rotation.x = Math.random() * 6;
        particles.rotation.y = Math.random() * 6;
        particles.rotation.z = Math.random() * 6;

        scene.add(particles);
      }

      //

      renderer = new THREE.WebGLRenderer();
      renderer.setPixelRatio(window.devicePixelRatio);
      renderer.setSize(window.innerWidth, window.innerHeight);
      document.body.appendChild(renderer.domElement);

      //

      //

      document.body.style.touchAction = "none";
      document.body.addEventListener("pointermove", this.onPointerMove);

      //

      window.addEventListener("resize", this.onWindowResize);
      console.log("init finished successfully");
    },
    animate: function () {
      //   console.log("animate");
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
        }
      }

      for (let i = 0; i < this.materials.length; i++) {
        const color = this.parameters[i][0];

        const h = ((360 * (color[0] + time)) % 360) / 360;
        this.materials[i].color.setHSL(h, color[1], color[2]);
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