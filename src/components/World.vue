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
let phongMaterial;
let cchoColors = ["#00A833", "#295F98", "#FE0000", "#FF7F00"];
let instanceTargetPosition = [0, 0, 0];
let instancePositions = [];
let instanceVelocities = [];
const dummy = new THREE.Object3D();
let mouseX = 0;
let mouseY = 0;
let windowHalfX = window.innerWidth / 2;
let windowHalfY = window.innerHeight / 2;
const cameraMoveSpeed = 0.1;
const parallaxCoeff = 1.5;

let origins = {
  c0: [window.innerWidth * -0.18, 0, -15],
  c1: [window.innerWidth * -0.105, 0, -5],
  h: [0, 0, 0],
  o: [window.innerWidth * 0.13, 0, 18],
};

export default {
  name: "World",
  data() {
    return {};
  },
  mounted() {
    this.moveInstanceTarget();
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

    moveInstanceTarget: function () {
      const mult = 50;
      const space = 300;
      const clamp = (num, min, max) => Math.min(Math.max(num, min), max);
      let targetX = clamp(
        this.randFloatInRange(-mult, mult) + instanceTargetPosition[0],
        -space,
        space
      );
      let targetY = clamp(
        this.randFloatInRange(-mult, mult) + instanceTargetPosition[1],
        -space,
        space
      );
      let targetZ = clamp(
        this.randFloatInRange(-mult, mult) + instanceTargetPosition[2],
        -space,
        space
      );
      instanceTargetPosition = [targetX, targetY, targetZ];
      // console.log(instanceTargetPosition);
    },

    addInstancedMesh: function () {
      this.fillRandomInstancePositions();
      this.initInstanceVelocities();

      let box = new THREE.BoxGeometry(10, 10, 10);

      phongMaterial = new THREE.MeshPhongMaterial({
        color: 0xffffff,
        shininess: 50,
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
      const uForce = 100;
      const uDamp = 0.999;
      const uMaxVel = 100;
      const baseColor0 = [0, 168, 51];
      const baseColor1 = [41, 95, 152];
      const baseColor2 = [254, 0, 0];
      const baseColor3 = [255, 127, 0];
      let colorPalette = [baseColor0, baseColor1, baseColor2, baseColor3];
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
        // set position
        dummy.position.set(ix, iy, iz);
        // dummy.position.set(ux, uy, uz);

        // update rotations
        dummy.rotation.set(Math.cos(i + time), Math.sin(i + time), 0);
        dummy.updateMatrix();
        instancedMesh.setMatrixAt(i, dummy.matrix);

        // update colors
        let baseColor = colorPalette[i % 4];
        let [cx, cy, cz] = baseColor;
        let baseColorVec = new THREE.Vector3(cx / 255, cy / 255, cz / 255);
        let colorMult = this.scale(oVelocity.length(), 0, 6, 0, 1);

        let oColor = baseColorVec.multiplyScalar(colorMult ** 4);
        oColor = oColor.clamp(
          new THREE.Vector3(0, 0, 0),
          new THREE.Vector3(1, 1, 1)
        );

        let instanceColor = new THREE.Color(
          Math.round(oColor.x),
          Math.round(oColor.y),
          Math.round(oColor.z)
        );

        instancedMesh.setColorAt(i, instanceColor);
      }
      // mark dirty
      instancedMesh.instanceMatrix.needsUpdate = true;
      instancedMesh.instanceColor.needsUpdate = true;
    },

    addCCHO: function () {
      this.addSphere();
      this.addTorus();
      this.addCube();
    },

    randColor: function () {
      return "#" + Math.floor(Math.random() * 16777215).toString(16);
    },

    addTorus: function () {
      for (let i = 0; i < 2; i++) {
        const oKey = ["c1", "c2"][i];
        const torusGeo = new THREE.TorusGeometry(
          100,
          30,
          30,
          20,
          Math.PI * 1.2
        );
        torusGeo.rotateZ(Math.PI / 2.5);
        torusGeo.rotateY(Math.PI * 0.2);
        if (i == 0) {
          torusGeo.translate(
            origins["c0"][0],
            origins["c0"][1],
            origins["c0"][2]
          );
        } else {
          torusGeo.translate(
            origins["c1"][0],
            origins["c1"][1],
            origins["c1"][2]
          );
        }
        const material = new THREE.MeshPhongMaterial({
          // color: this.randColor(),
          color: cchoColors[i],
          transparent: true,
          opacity: 1,
          shadowSide: THREE.BackSide,
        });

        const torus = new THREE.Mesh(torusGeo, material);
        scene.add(torus);
      }
    },

    addSphere: function () {
      let sphereGeo = new THREE.SphereGeometry(130, 50, 50, 0);
      sphereGeo.translate(origins["o"][0], origins["o"][1], origins["o"][2]);
      let material = new THREE.MeshPhongMaterial({
        // color: this.randColor(),
        color: cchoColors[3],
        transparent: true,
        opacity: 1,
      });
      let sphere = new THREE.Mesh(sphereGeo, material);
      scene.add(sphere);
    },

    addCube: function () {
      const boxWidth = window.innerWidth * 0.05;
      const boxheight = window.innerHeight * 0.08;

      const boxX = [
        -boxWidth,
        -boxWidth,
        -boxWidth,
        origins["h"][0],
        boxWidth,
        boxWidth,
        boxWidth,
      ];
      const boxY = [
        -boxheight,
        0,
        boxheight,
        origins["h"][1],
        -boxheight,
        0,
        boxheight,
      ];
      // const color = this.randColor();
      const color = cchoColors[2];

      for (let i = 0; i < 7; i++) {
        const boxGeo = new THREE.BoxGeometry(boxWidth, boxheight, boxheight);
        boxGeo.translate(boxX[i], boxY[i], 0);
        boxGeo.rotateY(Math.PI * 0.1);
        const material = new THREE.MeshPhongMaterial({
          color: color,
          reflectivity: 1,
          specular: 0xffffff,
          transparent: true,
          opacity: 1,
        });
        const cube = new THREE.Mesh(boxGeo, material);
        scene.add(cube);
      }
    },

    checkRadius: function (origin, radius = 150) {
      const center = new THREE.Vector3(
        origins["o"][0],
        origins["o"][1],
        Math.abs(origins["o"][2])
      );
      let activeColor = new THREE.Color(0x295f98);
      origin = center;

      for (let i = 0; i < instanceCount; i++) {
        const m = new THREE.Matrix4();
        const v = new THREE.Vector3();
        instancedMesh.getMatrixAt(i, m);
        v.setFromMatrixPosition(m);
        const distance = v.distanceTo(origin);

        if (distance < radius) {
          instancedMesh.setColorAt(i, activeColor);
        }
      }
      instancedMesh.instanceColor.needsUpdate = true;
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

      this.addCCHO();
      this.addInstancedMesh();
      this.addLights();

      renderer = new THREE.WebGLRenderer({ antialias: true });
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

    renderScene: function () {
      this.updateCamera();
      this.updateInstancedMesh();
      this.checkRadius();
      this.moveInstanceTarget();

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
