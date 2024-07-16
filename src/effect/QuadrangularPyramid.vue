<script setup>
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { nextTick, onBeforeUnmount, onMounted, ref } from 'vue'
import { gsap } from 'gsap'
import { GUI } from 'lil-gui'

let scene
let camera
let renderer
let orbitControls
const params = {
  flyColor: 0xffff00,
  routeColor: 0x00ffff
}
let gui = new GUI()
const container = ref(null)

const initializeScene = () => {
  scene = new THREE.Scene()
  scene.add(new THREE.GridHelper(10, 10))
}

const initializeCamera = () => {
  camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 4500)
  camera.position.set(10, 10, 10)
  camera.lookAt(0, 0, 0)
}

const initializeOrbitControls = () => {
  orbitControls = new OrbitControls(camera, renderer.domElement)
  orbitControls.enableDamping = false
  orbitControls.minPolarAngle = 0 // 默认值
}

const initializeRenderer = () => {
  renderer = new THREE.WebGLRenderer({
    antialias: true,
    alpha: true,
    precision: 'highp',
    premultipliedAlpha: true
  })
  renderer.clearDepth()
  renderer.toneMapping = THREE.ACESFilmicToneMapping
  renderer.toneMappingExposure = 1.0
  renderer.shadowMap.enabled = true
  renderer.shadowMap.type = THREE.PCFSoftShadowMap
  renderer.outputColorSpace = THREE.SRGBColorSpace
  renderer.gammaOutput = true // 将gammaOutput属性设置为true
  renderer.gammaFactor = 2.2 // Gamma默认值
}

const addTetrahedron = () => {
  const geometry = new THREE.ConeGeometry(1, 4, 32)

  const material = new THREE.MeshBasicMaterial({ color: 0x00ff00, side: THREE.DoubleSide })
  let tetrahedron = new THREE.Mesh(geometry, material)

  // Here you position, transform and animate your tetrahedron
  tetrahedron.position.set(0, 5, 0)
  tetrahedron.rotation.x = Math.PI
  gsap.to(tetrahedron.position, {
    y: 2,
    duration: 1,
    repeat: -1,
    yoyo: true
  })
  gsap.to(tetrahedron.rotation, {
    y: THREE.MathUtils.degToRad(-360),
    duration: 2,
    repeat: -1,
    ease: 'none'
  })
  scene.add(tetrahedron)
}

const createRadarAnimation = (position, scale = 5, duration = 1, delay = 0) => {
  gsap.delayedCall(delay, () => {
    // 创建一个作为“扩散”的环形
    let ringGeometry = new THREE.RingGeometry(0.5, 0.55, 100, 100)
    let ringMaterial = new THREE.MeshBasicMaterial({
      color: 0x00ff00,
      side: THREE.DoubleSide,
      transparent: true
    })
    let ring = new THREE.Mesh(ringGeometry, ringMaterial)

    ring.position.copy(position)
    ring.rotation.x = Math.PI / 2 // 将环旋转90度，使其平放
    scene.add(ring)

    // 创建 gsap 动画
    gsap.to(ring.scale, {
      x: scale,
      y: scale,
      z: scale,
      duration: duration,
      ease: 'none',
      repeat: -1 // 无限重复
    })

    gsap.to(ring.material, {
      opacity: 0,
      duration: duration,
      repeat: -1, // 无限重复
      onComplete: () => {
        // 动画完成后，我们移除环形
        scene.remove(ring)
        ring.geometry.dispose()
        ring.material.dispose()
      }
    })
  })
}

const addBox = () => {
  const geometry = new THREE.SphereGeometry(0.5, 32, 16)
  const material = new THREE.MeshBasicMaterial({ color: 0x00ff00 })
  const cube = new THREE.Mesh(geometry, material)
  cube.position.set(0, 8, 0)
  cube.rotation.x = Math.PI / 4
  cube.rotation.z = Math.PI / 4
  gsap.to(cube.position, {
    y: 5,
    duration: 1,
    repeat: -1,
    yoyo: true
  })
  // gsap.to(cube.rotation, {
  //   x: THREE.MathUtils.degToRad(-360),
  //   duration: 2,
  //   repeat: -1,
  //   ease: 'none'
  // })
  scene.add(cube)
}

const initialize = () => {
  initializeScene()
  initializeCamera()
  initializeRenderer()
  initializeOrbitControls()
  addTetrahedron()
  addBox()
  createRadarAnimation(new THREE.Vector3(0, 0, 0), 10, 2, 0) // 马上开始该动画
  createRadarAnimation(new THREE.Vector3(0, 0, 0), 10, 2, 1) // 0.5秒后开始该动画
  createRadarAnimation(new THREE.Vector3(0, 0, 0), 10, 2, 2) // 1秒后开始该动画

  animate()
}

const updateThree = () => {
  orbitControls && orbitControls.update()
  if (container.value) {
    camera.aspect = container.value.clientWidth / container.value.clientHeight
    camera.updateProjectionMatrix()
    renderer.setSize(container.value.clientWidth, container.value.clientHeight)
    renderer.setPixelRatio(window.devicePixelRatio * 2)
  }
}

const renderThree = () => {
  if (scene && camera && container.value) {
    renderer.render(scene, camera)
    container.value.appendChild(renderer.domElement)
  }
}

const animate = () => {
  requestAnimationFrame(animate)
  updateThree()
  renderThree()
}

const fitView = () => {
  window.addEventListener('resize', updateThree)
}

const cleanup = () => {
  renderer.forceContextLoss()
  renderer.dispose()
  scene.clear()
  window.removeEventListener('resize', updateThree)
}

onMounted(() => {
  nextTick(() => {
    if (container.value) {
      initialize()
      fitView()
    }
  })
})

onBeforeUnmount(() => {
  cleanup()
})
</script>

<template>
  <div class="three" ref="container"></div>
</template>

<style scoped>
.three {
  width: 100vw;
  height: 100vh;
  overflow: hidden;
  margin: 0;
  padding: 0;
  border: 0;
}
</style>
