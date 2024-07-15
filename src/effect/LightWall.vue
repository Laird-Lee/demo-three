<script setup>
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { nextTick, onBeforeUnmount, onMounted, ref } from 'vue'

import { GUI } from 'lil-gui'

let scene
let camera
let renderer
let orbitControls
let gui = new GUI()

let vertexShader = `
    varying vec2 vUv;
    void main(){
        vUv = vec2(uv.x,uv.y);
        vec4 mvPosition = modelViewMatrix * vec4( position, 1.0 );
        gl_Position = projectionMatrix * mvPosition;
    }
`
let fragmentShader = `
    varying vec2 vUv;
    uniform float vTime;
    uniform float vPow;
    uniform vec3 vColor;
    void main(){
        float vy = 1.0 - vUv.y;
        vy = pow(vy,vPow - abs(sin(vTime)));
        gl_FragColor = vec4(vColor,vy);
    }
`

let uniforms = {
  vTime: { value: 0.01 },
  vPow: { value: 2.0 },
  vColor: { value: new THREE.Color('#0678bc') }
}

const addMesh = () => {
  let planeHeight = 5

  let geometry = new THREE.PlaneGeometry(10, planeHeight)
  let material = new THREE.ShaderMaterial({
    uniforms,
    vertexShader,
    fragmentShader,
    transparent: true,
    side: THREE.DoubleSide
  })
  let mesh = new THREE.Mesh(geometry, material)

  let mesh1 = mesh.clone()
  mesh1.position.set(5, planeHeight / 2, 0)
  mesh1.rotation.y = Math.PI / 2
  scene.add(mesh1)

  let mesh2 = mesh.clone()
  mesh2.position.set(-5, planeHeight / 2, 0)
  mesh2.rotation.y = Math.PI / 2
  scene.add(mesh2)

  let mesh3 = mesh.clone()
  mesh3.position.set(0, planeHeight / 2, -5)
  scene.add(mesh3)

  let mesh4 = mesh.clone()
  mesh4.position.set(0, planeHeight / 2, 5)
  scene.add(mesh4)

  gui.add(uniforms.vPow, 'value', 0, 10).name('光栅栏高度')

  let colors = {
    vColor: '#0678bc'
  }
  gui
    .addColor(colors, 'vColor')
    .name('光栅栏颜色')
    .onChange((v) => {
      uniforms.vColor.value = new THREE.Color(v)
    })
}

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

const initialize = () => {
  initializeScene()
  initializeCamera()
  initializeRenderer()
  initializeOrbitControls()
  addMesh()
  animate()
}

const updateThree = () => {
  orbitControls && orbitControls.update()
  if (container.value) {
    camera.aspect = container.value.clientWidth / container.value.clientHeight
    camera.updateProjectionMatrix()
    renderer.setSize(container.value.clientWidth, container.value.clientHeight)
    renderer.setPixelRatio(window.devicePixelRatio * 2)
    uniforms.vTime.value += 0.05
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
