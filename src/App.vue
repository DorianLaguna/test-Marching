<template>
  <div id="ui">
    <button @click="toggleMode">
      Modo: {{ mode === 'add' ? 'Agregar' : 'Eliminar' }}
    </button>
  </div>
  <video
    ref="video"
    autoplay
    muted
    playsinline
    :style="{ display: videoVisible ? 'block' : 'none' }"
    width="320"
    height="240"
    style="position: absolute; top: 10px; left: 10px; pointer-events: none; z-index: 10;"
  ></video>

  <canvas ref="handCanvas" width="640" height="480" style="position:absolute; top:0; left:0; pointer-events:none;"></canvas>
  <canvas ref="threeCanvas" style="display:block; width: 100vw; height: 100vh;"></canvas>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'

import { Hands } from '@mediapipe/hands'
import { Camera } from '@mediapipe/camera_utils'
import { drawConnectors, drawLandmarks, HAND_CONNECTIONS } from '@mediapipe/drawing_utils'

const videoVisible = ref(true)

const video = ref(null)
const handCanvas = ref(null)
const threeCanvas = ref(null)

const mode = ref('add')
const groundPlane = new THREE.Plane(new THREE.Vector3(0, 1, 0), 0)

let scene, camera, renderer, controls
let cubes = []
let raycaster, mouse
let voxelSize = 1
let cubeGeo, cubeMat
let previewCube

let handsDetector, cameraUtils
let cursor, pointerLine
let fingerCursor = null;
function toggleMode() {
  mode.value = mode.value === 'add' ? 'remove' : 'add'
}

function addVoxel(pos) {
  const cube = new THREE.Mesh(cubeGeo, cubeMat.clone())
  cube.position.copy(pos)
  cube.userData.isVoxel = true
  scene.add(cube)
  cubes.push(cube)
}

function isPinching(landmarks) {
  const dx = landmarks[8].x - landmarks[4].x;
  const dy = landmarks[8].y - landmarks[4].y;
  const dz = landmarks[8].z - landmarks[4].z;
  const dist = Math.sqrt(dx * dx + dy * dy + dz * dz);
  return dist < 0.03; // Ajusta el umbral según pruebas
}

function updatePointerLine() {
  if (!fingerCursor.visible) {
    pointerLine.visible = false
    return
  }
  pointerLine.visible = true
  const positions = pointerLine.geometry.attributes.position.array
  // start punto: cámara
  positions[0] = camera.position.x
  positions[1] = camera.position.y
  positions[2] = camera.position.z
  // end punto: cursor
  positions[3] = fingerCursor.position.x
  positions[4] = fingerCursor.position.y
  positions[5] = fingerCursor.position.z
  pointerLine.geometry.attributes.position.needsUpdate = true
}

onMounted(() => {
  scene = new THREE.Scene()
  scene.background = new THREE.Color(0x202020)

  const cursorGeo = new THREE.SphereGeometry(0.05, 12, 12)
  const cursorMat = new THREE.MeshStandardMaterial({ color: 0xff0000, opacity: 0.7, transparent: true })
  fingerCursor = new THREE.Mesh(cursorGeo, cursorMat)
  scene.add(fingerCursor)
  fingerCursor.visible = false

  const material = new THREE.LineBasicMaterial({ color: 0xff0000 })
  const points = [ new THREE.Vector3(), new THREE.Vector3() ]
  const geometry = new THREE.BufferGeometry().setFromPoints(points)
  pointerLine = new THREE.Line(geometry, material)
  scene.add(pointerLine)
  pointerLine.visible = false

  camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000)
  camera.position.set(10, 10, 10)

  renderer = new THREE.WebGLRenderer({ canvas: threeCanvas.value, antialias: true })
  renderer.setSize(window.innerWidth, window.innerHeight)

  controls = new OrbitControls(camera, renderer.domElement)
  controls.enableDamping = true
  controls.target.set(0,0,0)
  controls.update()

  scene.add(new THREE.GridHelper(20, 20))
  scene.add(new THREE.AmbientLight(0xffffff, 0.3))

  const directionalLight = new THREE.DirectionalLight(0xffffff, 1)
  directionalLight.position.set(5, 10, 5)
  scene.add(directionalLight)

  cubeGeo = new THREE.BoxGeometry(voxelSize, voxelSize, voxelSize)
  cubeMat = new THREE.MeshStandardMaterial({ color: 0x44aa88 })

  addVoxel(new THREE.Vector3(0, 0, 0))

  raycaster = new THREE.Raycaster()
  mouse = new THREE.Vector2()

  const previewMat = new THREE.MeshStandardMaterial({ color: 0xffffff, opacity: 0.5, transparent: true })
  previewCube = new THREE.Mesh(cubeGeo, previewMat)
  previewCube.visible = false
  scene.add(previewCube)

  // --- MediaPipe Hands setup ---
  handsDetector = new Hands({
    locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}`
  })

  handsDetector.setOptions({
    maxNumHands: 2,
    modelComplexity: 1,
    minDetectionConfidence: 0.7,
    minTrackingConfidence: 0.7
  })

  handsDetector.onResults(onHandsResults)

  cameraUtils = new Camera(video.value, {
    onFrame: async () => {
      await handsDetector.send({ image: video.value })
    },
    width: 640,
    height: 480
  })

  cameraUtils.start()

  window.addEventListener('resize', onResize)
  animate()
})

function onResize() {
  camera.aspect = window.innerWidth / window.innerHeight
  camera.updateProjectionMatrix()
  renderer.setSize(window.innerWidth, window.innerHeight)
}

function onHandsResults(results) {
  const ctx = handCanvas.value.getContext('2d')
  ctx.clearRect(0, 0, handCanvas.value.width, handCanvas.value.height)
  ctx.drawImage(results.image, 0, 0, handCanvas.value.width, handCanvas.value.height)

  if (results.multiHandLandmarks && results.multiHandLandmarks.length > 0) {
    videoVisible.value = true
    const indexFinger = results.multiHandLandmarks[0][8]
    if (indexFinger && fingerCursor) {
      // Invertir la X para corregir el espejo horizontal
      mouse.x = 1 - indexFinger.x * 2; // en vez de indexFinger.x * 2 - 1
      mouse.y = -(indexFinger.y * 2 - 1);

      raycaster.setFromCamera(mouse, camera)
      const plane = new THREE.Plane(new THREE.Vector3(0, 1, 0), 0) // plano XZ a Y=0
      const intersectPoint = new THREE.Vector3()
      raycaster.ray.intersectPlane(plane, intersectPoint)

      fingerCursor.position.copy(intersectPoint)
      fingerCursor.visible = true
    }
  } else {
    videoVisible.value = false
    if (fingerCursor) fingerCursor.visible = false
  }

}

function animate() {
  requestAnimationFrame(animate);
  controls.update();

  // Ajustar mouse.x para que mano izquierda no sea espejo:
  // (Solo si lo actualizas desde Mediapipe, invertirlo allí)
  // Aquí asumo que mouse.x ya viene invertido (te pongo el fix en onHandsResults luego)

  raycaster.setFromCamera(mouse, camera);

  // Primero intento intersectar cubos
  const intersects = raycaster.intersectObjects(cubes);

  if (intersects.length > 0) {
    const intersect = intersects[0];
    const normal = intersect.face.normal.clone();
    const position = intersect.object.position.clone().add(normal.multiplyScalar(voxelSize));

    fingerCursor.position.copy(position);
    fingerCursor.visible = true;

    if (mode.value === 'add') {
      previewCube.position.copy(position);
      previewCube.visible = true;
    } else {
      previewCube.visible = false;
    }

  } else {
    // No intersecta cubos, pruebo con el plano suelo
    const intersectPoint = new THREE.Vector3();
    if (raycaster.ray.intersectPlane(groundPlane, intersectPoint)) {
      // Ajustar a grid
      const gridPos = new THREE.Vector3(
        Math.round(intersectPoint.x / voxelSize) * voxelSize,
        0,
        Math.round(intersectPoint.z / voxelSize) * voxelSize
      );
      fingerCursor.position.copy(gridPos);
      fingerCursor.visible = true;

      if (mode.value === 'add') {
        previewCube.position.copy(gridPos);
        previewCube.visible = true;
      } else {
        previewCube.visible = false;
      }
    } else {
      fingerCursor.visible = false;
      previewCube.visible = false;
    }
  }

  renderer.render(scene, camera);
}
</script>

<style>
body {
  margin: 0;
  overflow: hidden;
  background: #202020;
  user-select: none;
}
#ui {
  position: absolute;
  top: 10px;
  left: 10px;
  z-index: 10;
  background: rgba(0, 0, 0, 0.6);
  color: white;
  padding: 8px 12px;
  border-radius: 8px;
  font-family: sans-serif;
}
video {
  position: absolute;
  top: 10px;   /* o donde quieras */
  left: 10px;
  width: 320px;  /* tamaño fijo */
  height: 240px;
  pointer-events: none; /* para que no interfiera con clicks */
  z-index: 10;  /* encima del canvas */
}
canvas {
  display: block; 
  width: 100vw;
  height: 100vh;
  position: fixed;
  top: 0;
  left: 0;
  z-index: 1;
}
</style>
