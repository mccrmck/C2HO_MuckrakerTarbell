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
let instanceCount = 1000;
let ambientLight, pointLight;
let instanceTargetPosition = [0, 0, 0];
let instancePositions = [];
let instanceVelocities = [];
const dummy = new THREE.Object3D();
const numParticles = 50;
let mouseX = 0;
let mouseY = 0;
let windowHalfX = window.innerWidth / 2;
let windowHalfY = window.innerHeight / 2;
const cameraMoveSpeed = 0.1;
const parallaxCoeff = 0.3;
let parameters;
let materials = [];
let active = [];
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
    scale: function (x, inLow, inHigh, outLow, outHigh) {
      return ((x - inLow) * (outHigh - outLow)) / (inHigh - inLow) + outLow;
    },
    randFloatInRange: function (from, to) {
      return this.scale(Math.random(), 0, 1, from, to);
    },

    fillRandomInstancePositions: function () {
      instancePositions = [];
      for (let i = 0; i < instanceCount; i++) {
        instancePositions.push([
          this.randFloatInRange(-1000, 1000),
          this.randFloatInRange(-1000, 1000),
          this.randFloatInRange(-1000, 1000),
        ]);
      }
    },
    initInstanceVelocities: function () {
      instanceVelocities = [];
      for (let i = 0; i < instanceCount; i++) {
        instanceVelocities.push([0, 0, 0]);
      }
    },
    addLights: function () {
      console.log("addLights");

      ambientLight = new THREE.AmbientLight(0xffffff);
      scene.add(ambientLight);

      const spotLight = new THREE.SpotLight(0xffffff);
      spotLight.position.set(100, 1000, 100);

      spotLight.castShadow = true;

      spotLight.shadow.mapSize.width = 1024;
      spotLight.shadow.mapSize.height = 1024;

      spotLight.shadow.camera.near = 500;
      spotLight.shadow.camera.far = 4000;
      spotLight.shadow.camera.fov = 30;

      scene.add(spotLight);

      pointLight = new THREE.PointLight(0xffffff, 1);
      pointLight.position.y = 500;
      scene.add(pointLight);
    },

    addInstancedMesh: function () {
      console.log("addInstancedMesh");

      this.fillRandomInstancePositions();
      this.initInstanceVelocities();

      let box = new THREE.BoxGeometry(10, 10, 10);

      let phongMaterial = new THREE.MeshPhongMaterial({
        color: 0x00a2ff,
        shininess: 100,
        reflectivity: 1,
      });

      instancedMesh = new THREE.InstancedMesh(
        box,
        phongMaterial,
        instanceCount
      );
      instancedMesh.instanceMatrix.setUsage(THREE.DynamicDrawUsage); // will be updated every frame
      scene.add(instancedMesh);
    },

    updateInstancedMesh: function () {
      const time = Date.now() * 0.0005;
      const uForce = 4;
      const uDamp = 0.9999;
      const uMaxVel = 10;
      for (let i = 0; i < instanceCount; i++) {
        //update positions

        let [ix, iy, iz] = instancePositions[i];
        const iPosition = new THREE.Vector3(ix, iy, iz);
        let [ux, uy, uz] = instanceTargetPosition;
        const uTarget = new THREE.Vector3(ux, uy, uz);

        let dir = uTarget.sub(iPosition);
        let normDir = dir.normalize();
        let dist = normDir.length();
        let acc = normDir.multiplyScalar(uForce);
        acc = acc.divideScalar(dist * 50 * (dist * 50) + 1.0);

        let [vx, vy, vz] = instanceVelocities[i];
        const iVelocity = new THREE.Vector3(vx, vy, vz);
        let oVelocity = iVelocity.add(acc);
        oVelocity = oVelocity.multiplyScalar(uDamp);
        let minVel = new THREE.Vector3(uMaxVel, uMaxVel, uMaxVel);
        minVel = minVel.multiplyScalar(-1);
        let maxVel = new THREE.Vector3(uMaxVel, uMaxVel, uMaxVel);
        oVelocity = oVelocity.clamp(minVel, maxVel);
        let oPosition = iPosition.add(oVelocity);
        // update instance position buffer
        instancePositions[i] = [oPosition.x, oPosition.y, oPosition.z];
        [ix, iy, iz] = instancePositions[i];
        // update instance velocities
        instanceVelocities[i] = [oVelocity.x, oVelocity.y, oVelocity.z];

        // if (i == 0) {
        //   console.log([oPosition.x, oPosition.y, oPosition.z]);
        // }

        dummy.position.set(ix, iy, iz);

        // update rotations
        dummy.rotation.set(Math.cos(i + time), Math.sin(i + time), 0);
        dummy.updateMatrix();
        instancedMesh.setMatrixAt(i, dummy.matrix);
      }
      instancedMesh.instanceMatrix.needsUpdate = true;
    },

    addParticles: function () {
      console.log("addParticles");
      const geometry = new THREE.BufferGeometry();
      const vertices = [];

      for (let i = 0; i < numParticles; i++) {
        const x = Math.random() * 2000 - 1000;
        const y = Math.random() * 2000 - 1000;
        const z = Math.random() * 2000 - 1000;

        vertices.push(x, y, z);
        active.push(false);
      }

      geometry.setAttribute(
        "position",
        new THREE.Float32BufferAttribute(vertices, 3)
      );

      parameters = [[[1.0, 0.2, 0.5], "yo", 20]];

      for (let i = 0; i < parameters.length; i++) {
        const color = parameters[i][0];
        // const sprite = parameters[i][1];
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
    },

    addCCHO: function () {
      console.log("addCCHO");
      this.addSpheres();
      this.addTorus();
      this.addCube();
    },

    addTorus: function () {
      console.log("addTorus");
      const radius = 100;
      const rSegs = 20;
      const tColor = () => Math.floor(Math.random() * 16777215).toString(16);
      const width = window.innerWidth;
      const tXY = [width * -0.3, width * -0.15];

      for (let i = 0; i < 2; i++) {
        const torusGeo = new THREE.TorusGeometry(
          radius,
          20,
          rSegs,
          12,
          Math.PI * 1.2
        );
        torusGeo.rotateZ(Math.PI / 2.5);
        torusGeo.translate(tXY[i], 0, 0);
        const material = new THREE.MeshBasicMaterial({
          color: "#" + tColor(),
          transparent: true,
          opacity: 1,
          shadowSide: THREE.BackSide,
        });

        const torus = new THREE.Mesh(torusGeo, material);
        scene.add(torus);
      }
    },

    addSpheres: function () {
      console.log("addSpheres");
      const space = window.innerWidth * 0.2;
      const sphereX = [-space * 2, -space, space];
      const spherePhi = [Math.PI / 3, Math.PI / 3, Math.PI * 2];

      for (let i = 2; i < 3; i++) {
        let sphereGeo = new THREE.SphereGeometry(130, 50, 50, 0, spherePhi[i]);
        sphereGeo.translate(sphereX[i], 0, 0);
        let randCol = () => Math.floor(Math.random() * 16777215).toString(16);
        let material = new THREE.MeshBasicMaterial({
          color: "#" + randCol(),
          transparent: true,
          opacity: 1,
        });
        let sphere = new THREE.Mesh(sphereGeo, material);
        scene.add(sphere);
      }
    },

    addCube: function () {
      console.log("addCube");
      const boxWidth = window.innerWidth * 0.05;
      const boxheight = window.innerHeight * 0.08;
      const boxColor = "#" + Math.floor(Math.random() * 16777215).toString(16);

      const boxX = [
        -boxWidth,
        -boxWidth,
        -boxWidth,
        0,
        boxWidth,
        boxWidth,
        boxWidth,
      ];
      const boxY = [-boxheight, 0, boxheight, 0, -boxheight, 0, boxheight];

      for (let i = 0; i < 7; i++) {
        const boxGeo = new THREE.BoxGeometry(boxWidth, boxheight, boxheight);
        boxGeo.translate(boxX[i], boxY[i], 0);
        const material = new THREE.MeshBasicMaterial({
          color: boxColor,
          transparent: true,
          opacity: 1,
        });
        const cube = new THREE.Mesh(boxGeo, material);
        scene.add(cube);
      }
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

      // this.addParticles();
      this.addCCHO();
      this.addInstancedMesh();
      this.addLights();

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

    updateCamera: function () {
      camera.position.x += (mouseX - camera.position.x) * cameraMoveSpeed;
      camera.position.y += (-mouseY - camera.position.y) * cameraMoveSpeed;
      camera.lookAt(scene.position);
    },

    updateParticles: function () {
      const time = Date.now() * 0.00005;

      for (let i = 0; i < scene.children.length; i++) {
        const object = scene.children[i];
        if (object instanceof THREE.Points) {
          object.rotation.y = time * (i < 4 ? i + 1 : -(i + 1));
        }
      }
    },

    updateMaterials: function () {
      const time = Date.now() * 0.00005;

      for (let i = 0; i < materials.length; i++) {
        const color = parameters[i][0];
        let h = ((360 * (color[0] + time)) % 360) / 360;
        materials[i].color.setHSL(h, color[1], color[2]);
      }
    },

    renderScene: function () {
      this.updateCamera();
      // this.updateParticles();
      this.updateMaterials();
      this.updateInstancedMesh();

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
