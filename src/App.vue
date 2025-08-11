<template>
  <div id="ui">
    <button id="toggle" @click="toggleMode">
      Modo: {{ mode === 'add' ? 'Agregar' : 'Eliminar' }}
    </button>
  </div>
  <canvas ref="canvas"></canvas>
</template>

<script setup>
import { onMounted, ref } from 'vue'
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'

const canvas = ref(null)
const mode = ref('add')

let scene, camera, renderer, controls
let cubes = []
let raycaster, mouse
let voxelSize = 1
let cubeGeo, cubeMat
let previewCube

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

onMounted(() => {
  scene = new THREE.Scene()
  scene.background = new THREE.Color(0x202020)

  camera = new THREE.PerspectiveCamera(
    75,
    window.innerWidth / window.innerHeight,
    0.1,
    1000
  )
  camera.position.set(10, 10, 10)

  renderer = new THREE.WebGLRenderer({ canvas: canvas.value, antialias: true })
  renderer.setSize(window.innerWidth, window.innerHeight)

  controls = new OrbitControls(camera, renderer.domElement)
  controls.enableDamping = true

  scene.add(new THREE.GridHelper(20, 20))

  const light = new THREE.DirectionalLight(0xffffff, 1)
  light.position.set(5, 10, 5)
  scene.add(light)

  scene.add(new THREE.AmbientLight(0xffffff, 0.3))

  cubeGeo = new THREE.BoxGeometry(voxelSize, voxelSize, voxelSize)
  cubeMat = new THREE.MeshStandardMaterial({ color: 0x44aa88 })

  // Cubo inicial
  addVoxel(new THREE.Vector3(0, 0, 0))

  raycaster = new THREE.Raycaster()
  mouse = new THREE.Vector2()

  // Preview cube
  const previewMat = new THREE.MeshStandardMaterial({
    color: 0xffffff,
    opacity: 0.5,
    transparent: true
  })
  previewCube = new THREE.Mesh(cubeGeo, previewMat)
  previewCube.visible = false
  scene.add(previewCube)

  window.addEventListener('mousemove', onMouseMove)
  window.addEventListener('click', onClick)
  window.addEventListener('resize', onResize)

  animate()
})

function onMouseMove(event) {
  const rect = canvas.value.getBoundingClientRect()
  mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1
  mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1
}

function onClick() {
  raycaster.setFromCamera(mouse, camera)
  const intersects = raycaster.intersectObjects(cubes)

  if (intersects.length > 0) {
    const intersect = intersects[0]
    const clickedCube = intersect.object

    if (mode.value === 'add') {
      const normal = intersect.face.normal.clone()
      const newPos = clickedCube.position
        .clone()
        .add(normal.multiplyScalar(voxelSize))
      addVoxel(newPos)
    } else if (mode.value === 'remove') {
      scene.remove(clickedCube)
      const i = cubes.indexOf(clickedCube)
      if (i > -1) cubes.splice(i, 1)
    }
  }
}

function onResize() {
  camera.aspect = window.innerWidth / window.innerHeight
  camera.updateProjectionMatrix()
  renderer.setSize(window.innerWidth, window.innerHeight)
}

function animate() {
  requestAnimationFrame(animate)
  controls.target.set(0, 0, 0)
controls.update()

  if (mode.value === 'add') {
    raycaster.setFromCamera(mouse, camera)
    const intersects = raycaster.intersectObjects(cubes)
    if (intersects.length > 0) {
      const intersect = intersects[0]
      const normal = intersect.face.normal.clone()
      const position = intersect.object.position
        .clone()
        .add(normal.multiplyScalar(voxelSize))
      previewCube.position.copy(position)
      previewCube.visible = true
    } else {
      previewCube.visible = false
    }
  } else {
    previewCube.visible = false
  }

  renderer.render(scene, camera)
}
</script>

<style>
body {
  margin: 0;
  overflow: hidden;
}
#ui {
  position: absolute;
  top: 10px;
  left: 10px;
  z-index: 1;
  background: rgba(0, 0, 0, 0.6);
  color: white;
  padding: 8px 12px;
  border-radius: 8px;
  font-family: sans-serif;
}
#toggle {
  padding: 5px 10px;
  font-size: 14px;
}
</style>
