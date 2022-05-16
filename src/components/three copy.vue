<template>
  <div ref="three" />
</template>

<script>
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader'
import TWEEN from 'three-tween'

const bufArrays = []
let current = 0
const particalNum = 1000
const modalNum = 10000
let windowHalfX = window.innerWidth / 2;
let windowHalfY = window.innerHeight / 2;

export default {
  data() {
    return {
      timeout: undefined,
      mouseX: 0,
      mouseY: 0,
    }
  },
  mounted () {
    this._initBase()
  },
  destroyed() {
    clearTimeout(this.timeout)
  },
  deactivated() {
    clearTimeout(this.timeout)
  },
  methods: {
    _initBase () {
      this.scene = new THREE.Scene()
      this.scene.background = this.backgroundTexture()

      this.camera = new THREE.PerspectiveCamera(45, window.innerWidth / window.innerHeight, 0.1, 80)
      this.camera.position.set(0, 0, 6)

      this.renderer = new THREE.WebGLRenderer({ antialias: true })
      this.renderer.setPixelRatio(window.devicePixelRatio)
      this.renderer.setSize(window.innerWidth, window.innerHeight)

      const manager = new THREE.LoadingManager()

      manager.onStart = (url, itemsLoaded, itemsTotal) => {
        console.log('onStart')
      }

      manager.onLoad = () => {
        console.log('onLoad')
        this.transition()
      }

      manager.onError = url => {
        console.log(url)
      }

      const gltfLoader = new GLTFLoader(manager)

      gltfLoader.load('/box.glb', gltf => {
        gltf.scene.traverse(child => {
          if (child.isMesh) {
            child.geometry.translate(0, 0.5, 0)
            const { array } = child.geometry.attributes.position
            bufArrays.push(array)
          }
        })
      })
      gltfLoader.load('/box1.glb', gltf => {
        gltf.scene.traverse(child => {
          if (child.isMesh) {
            child.geometry.scale(0.5, 0.5, 0.5)
            const { array } = child.geometry.attributes.position
            bufArrays.push(array)
          }
        })
      })
      gltfLoader.load('/sphere.glb', gltf => {
        gltf.scene.traverse(child => {
          if (child.isMesh) {
            child.geometry.translate(1, 0, 0)
            const { array } = child.geometry.attributes.position
            bufArrays.push(array)
          }
        })
      })

      this.geometry = new THREE.BufferGeometry()
      this.geometry.tween = []
      const vertices = []

      for (let i = 0; i < modalNum; i++) {
        const position = THREE.MathUtils.randFloat(-4, 4)
        this.geometry.tween.push(new TWEEN.Tween({ position }).easing(TWEEN.Easing.Exponential.In))
        vertices.push(position)
      }

      this.geometry.setAttribute('position', new THREE.BufferAttribute(new Float32Array(vertices), 3))

      this.points = new THREE.Points(this.geometry, new THREE.PointsMaterial({
        size: 0.08,
        map: new THREE.TextureLoader().load('/white-dot.png'),
        alphaTest: 0.1,
        opacity: 0.5,
        transparent: true,
        depthTest: true
      }))

      this.scene.add(this.points)

      this.geometry2 = new THREE.BufferGeometry()
      this.geometry2.tween = []
      const vertices2 = []

      for (let i = 0; i < particalNum; i++) {
        const position = THREE.MathUtils.randFloat(-4, 4)
        this.geometry2.tween.push(new TWEEN.Tween({ position }).easing(TWEEN.Easing.Exponential.In))
        vertices2.push(position)
      }

      this.geometry2.setAttribute('position', new THREE.BufferAttribute(new Float32Array(vertices2), 3))

      this.points2 = new THREE.Points(this.geometry2, new THREE.PointsMaterial({
        size: 0.1,
        map: new THREE.TextureLoader().load('/white-dot.png'),
        alphaTest: 0.1,
        opacity: 0.2,
        transparent: true,
        depthTest: true
      }))

      this.scene.add(this.points2)

      this.controls = new OrbitControls(this.camera, this.renderer.domElement)
      this.controls.update()

      this.$refs.three.appendChild(this.renderer.domElement)
      window.addEventListener('resize', this.onWindowResize, false)
      window.addEventListener( 'mousemove', this.onDocumentMouseMove );
      this.render()
    },

    transition () {
      const self = this
      for (let i = 0, j = 0; i < modalNum; i++, j++) {
        const item = this.geometry.tween[i]
        if (j >= bufArrays[current].length) {
          j = 0
        }
        item.to({ position: bufArrays[current][j] }, THREE.MathUtils.randFloat(1000, 4000)).onUpdate(function () {
          self.geometry.attributes.position.array[i] = this.position
          self.geometry.attributes.position.needsUpdate = true
        }).start()
      }

      this.timeout = setTimeout(() => { this.transition() }, 10000)
      current = (current + 1) % 3
    },

    render () {
      this.points2.rotation.x += 0.0003
      this.points2.rotation.y += 0.001
      this.points2.rotation.z += 0.002
      TWEEN.update()

      if(this.mouseX != 0 || this.mouseY != 0) {
        this.camera.position.x += ( this.mouseX - this.camera.position.x ) * .0002;
        this.camera.position.y += ( - this.mouseY - this.camera.position.y ) * .0002;
        this.camera.lookAt( this.scene.position );
      }

      this.renderer.render(this.scene, this.camera)
      requestAnimationFrame(this.render)
    },
    onWindowResize () {
      this.renderer.setSize(window.innerWidth, window.innerHeight)
      this.camera.aspect = window.innerWidth / window.innerHeight
      this.camera.updateProjectionMatrix()
    },
    onDocumentMouseMove(event) {
      this.mouseX = ( event.clientX - windowHalfX ) / 2;
      this.mouseY = ( event.clientY - windowHalfY ) / 2;
      if (event.clientX < 20 || event.clientX > (windowHalfX * 2 - 20) || event.clientY < 20 || event.clientY > (windowHalfY * 2 -20)) {
        this.mouseX = 0
        this.mouseY = 0
      }
    },
    backgroundTexture () {
      const canvas = document.createElement('canvas')
      canvas.width = window.innerWidth
      canvas.height = window.innerHeight
      const ctx = canvas.getContext('2d')
      const gradient = ctx.createLinearGradient(0, 0, window.innerWidth, 0)
      gradient.addColorStop(0, '#4e22b7')
      gradient.addColorStop(1, '#3292ff')
      ctx.fillStyle = gradient
      ctx.fillRect(0, 0, canvas.width, canvas.height)
      const canvasTexture = new THREE.CanvasTexture(canvas)
      return canvasTexture
    }
  }
}
</script>
