<template>
  <div ref="container" class="canvas-container">
    <div id="ui">
      <button @click="toggleMode">Modo: {{ mode === 'add' ? 'Agregar' : 'Eliminar' }}</button>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue'
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'

const container = ref(null)
let scene, camera, renderer, controls, raycaster, mouse, previewCube
const cubes = []
const voxelSize = 1
let mode = ref('add')

const toggleMode = () => {
  mode.value = mode.value === 'add' ? 'remove' : 'add'
}

function addVoxel(pos) {
  const cube = new THREE.Mesh(
    new THREE.BoxGeometry(voxelSize, voxelSize, voxelSize),
    new THREE.MeshStandardMaterial({ color: 0x44aa88 })
  )
  cube.position.copy(pos)
  cube.userData.isVoxel = true
  scene.add(cube)
  cubes.push(cube)
}

onMounted(() => {
  scene = new THREE.Scene()
  scene.background = new THREE.Color(0x202020)

  camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000)
  camera.position.set(10, 10, 10)

  renderer = new THREE.WebGLRenderer({ antialias: true })
  renderer.setSize(window.innerWidth, window.innerHeight)
  container.value.appendChild(renderer.domElement)

  controls = new OrbitControls(camera, renderer.domElement)
  controls.enableDamping = true

  raycaster = new THREE.Raycaster()
  mouse = new THREE.Vector2()

  const light = new THREE.DirectionalLight(0xffffff, 1)
  light.position.set(5, 10, 5)
  scene.add(light)

  const gridHelper = new THREE.GridHelper(20, 20)
  scene.add(gridHelper)

  addVoxel(new THREE.Vector3(0, 0, 0))

  const previewMat = new THREE.MeshStandardMaterial({
    color: 0xffffff,
    opacity: 0.5,
    transparent: true
  })
  previewCube = new THREE.Mesh(new THREE.BoxGeometry(voxelSize, voxelSize, voxelSize), previewMat)
  previewCube.visible = false
  scene.add(previewCube)

  window.addEventListener('mousemove', onMouseMove)
  window.addEventListener('click', onClick)
  window.addEventListener('resize', onResize)

  animate()
})

onBeforeUnmount(() => {
  window.removeEventListener('mousemove', onMouseMove)
  window.removeEventListener('click', onClick)
  window.removeEventListener('resize', onResize)
})

function onMouseMove(event) {
  mouse.x = (event.clientX / window.innerWidth) * 2 - 1
  mouse.y = -(event.clientY / window.innerHeight) * 2 + 1
}

function onClick() {
  raycaster.setFromCamera(mouse, camera)
  const intersects = raycaster.intersectObjects(cubes)

  if (intersects.length > 0) {
    const intersect = intersects[0]
    const clickedCube = intersect.object

    if (mode.value === 'add') {
      const normal = intersect.face.normal.clone()
      const newPos = clickedCube.position.clone().add(normal.multiplyScalar(voxelSize))
      addVoxel(newPos)
    } else {
      scene.remove(clickedCube)
      const i = cubes.indexOf(clickedCube)
      if (i > -1) cubes.splice(i, 1)
    }
  }
}

function animate() {
  requestAnimationFrame(animate)
  controls.update()

  if (mode.value === 'add') {
    raycaster.setFromCamera(mouse, camera)
    const intersects = raycaster.intersectObjects(cubes)
    if (intersects.length > 0) {
      const intersect = intersects[0]
      const normal = intersect.face.normal.clone()
      const pos = intersect.object.position.clone().add(normal.multiplyScalar(voxelSize))
      previewCube.position.copy(pos)
      previewCube.visible = true
    } else {
      previewCube.visible = false
    }
  } else {
    previewCube.visible = false
  }

  renderer.render(scene, camera)
}

function onResize() {
  camera.aspect = window.innerWidth / window.innerHeight
  camera.updateProjectionMatrix()
  renderer.setSize(window.innerWidth, window.innerHeight)
}
</script>

<style scoped>
.canvas-container {
  margin: 0;
  overflow: hidden;
  width: 100vw;
  height: 100vh;
  position: relative;
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
</style>
