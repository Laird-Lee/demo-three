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

let vertexShader = `
    attribute float aIndex;
    uniform float uTime;
    uniform vec3 uColor;
    varying float vSize;
    void main() {
      vec4 viewPosition = viewMatrix * modelMatrix *vec4(position,1);
      gl_Position = projectionMatrix * viewPosition;
      if(aIndex < uTime + 100.0 && aIndex > uTime - 100.0) {
          vSize = (aIndex + 100.0 - uTime) / 60.0;
      }
      gl_PointSize =vSize;
    }
`
let fragmentShader = `
    varying float vSize;
    uniform vec3 uColor;
    void main() {
      if ( vSize <= 0.0 ) {
        gl_FragColor = vec4(1,0,0,0);
      } else {
        gl_FragColor = vec4(uColor,1);
      }
    }
`

const container = ref(null)

const initializeScene = () => {
  scene = new THREE.Scene()
  scene.add(new THREE.GridHelper(200, 200))
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

const addFlyLine = (dataInfo) => {
  dataInfo.map((data) => {
    const startPoint = data.begin // 起始点
    const endPoint = data.end // 终点
    const curveH = data.height // 飞线最大高

    const begin = new THREE.Vector3(startPoint[0], 0, startPoint[0])
    const end = new THREE.Vector3(endPoint[0], 0, endPoint[1])
    const len = begin.distanceTo(end)

    // 创建管道
    const pointInLine = [
      new THREE.Vector3(startPoint[0], 0, startPoint[0]),
      new THREE.Vector3(
        (startPoint[0] + endPoint[0]) / 2,
        curveH,
        (startPoint[1] + endPoint[1]) / 2
      ),
      new THREE.Vector3(endPoint[0], 0, endPoint[1])
    ]

    const lineCurve = new THREE.CatmullRomCurve3(pointInLine)

    const points = lineCurve.getPoints(1000)

    const indexList = []
    points.forEach((item, index) => {
      indexList.push(index)
    })

    const geometry = new THREE.BufferGeometry().setFromPoints(points)
    geometry.setAttribute('aIndex', new THREE.Float32BufferAttribute(indexList, 1))

    const material = new THREE.ShaderMaterial({
      uniforms: {
        uColor: {
          value: new THREE.Color(params.flyColor)
        },
        uTime: {
          value: 0
        },
        uLength: {
          value: points.length
        }
      },
      vertexShader,
      fragmentShader,
      transparent: true
    })
    scene.add(new THREE.Points(geometry, material))
    const p = {
      index: 0
    }
    gsap.to(p, {
      index: 1000,
      duration: 1,
      repeat: -1,
      onUpdate() {
        material.uniforms.uTime.value = Math.ceil(p.index)
      }
    })
  })
}

function generateData(arrayLength) {
  const result = []

  for (let i = 0; i < arrayLength; i++) {
    result.push({
      begin: [0, 0],
      end: [Math.floor(Math.random() * 401) - 200, Math.floor(Math.random() * 401) - 200],
      height: 10 + i
    })
  }

  return result
}

const initialize = () => {
  initializeScene()
  initializeCamera()
  initializeRenderer()
  initializeOrbitControls()
  addFlyLine(generateData(50))
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
