<template>
    <div class="container" ref="container" id="container">
        <canvas id="canvas"/>
        <canvas id="name"/>
    </div>
</template>
<script setup>
import * as THREE from 'three'
import chinaJson from './config/china.json'
import * as d3geo from 'd3-geo'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls'
import { onMounted } from 'vue'
import { Scene, Object3D } from 'three'
import axios from 'axios'
import { ref } from 'vue'
import TWEEN from '@tweenjs/tween.js'

const scene = new Scene()
let map = new Object3D()
const container = ref(null)

let camera = null
let renderer = null
let requestAF = 0
let controls = null

let raycaster = new THREE.Raycaster()
let mouse = new THREE.Vector2()
let INTERSECTED = null
let ISDRAG = false, ISDOWN = false
let mapDataStack = [ chinaJson ]

const getMapData = (code) => {
    const url = `https://geo.datav.aliyun.com/areas_v3/bound/geojson?code=${code}_full`
    return axios
        .get(url)
        .then((res) => res.data)
        .catch(() => {
            const tempUrl = `https://geo.datav.aliyun.com/areas_v3/bound/geojson?code=${code}`
            return axios.get(tempUrl).then((res) => res.data)
        })
}

// 初始化3D环境
const initEnvironment = () => {
    scene.background = new THREE.Color(0xf0f0f0)
    // 设置相机参数
    setCamera()
    // 初始化
    renderer = new THREE.WebGLRenderer({
        alpha: true,
        canvas: document.querySelector('canvas')
    })
    renderer.setPixelRatio(window.devicePixelRatio)
    renderer.setSize(window.innerWidth, window.innerHeight)
    document.addEventListener('mousemove', onDocumentMouseMove, false)
    document.addEventListener('mousedown', onMouseDown, false)
    document.addEventListener('mouseup', onMouseUp, false)
    window.addEventListener('resize', onWindowResize, false)
}

const onMouseUp = async () => {
    if (INTERSECTED && !ISDRAG) {
        const adcode = INTERSECTED.parent.properties.adcode
        const center = INTERSECTED.parent.properties._centroid
        const camloc = new TWEEN.Tween(camera.position).to({x: center[0], y: -center[1] - 10, z: 40}, 2000)
            .onComplete(() => {
                // controls.update()
            })
            .easing(TWEEN.Easing.Linear.None)
            .start()
        const target = {...controls.target, z: 0}
        const camtar = new TWEEN.Tween(target).to({x: center[0], y: -center[1], z: 0}, 2000)
            .easing(TWEEN.Easing.Linear.None)
            .onComplete(() => {
                controls.target = new THREE.Vector3(center[0], -center[1], 0)
                controls.update()
            })
            .onUpdate(() => {
                camera.lookAt(new THREE.Vector3(target.x, target.y, target.z))
            }).start()
        // controls.target = new THREE.Vector3(center[0], -center[1], 0)
        // controls.update()
        // camera.position.set(center[0], -center[1], 40)
        // controls.target = new THREE.Vector3(center[0], -center[1], 0)
        
        // controls.reset()
        initMap(adcode)
        map.remove(INTERSECTED.parent)
        
    }
    ISDOWN = false
}

const onMouseDown = () => {
    ISDOWN = true
    ISDRAG = false
}

// const getMapData = (code) => {

// }

const initMap = async (code) => {
    // d3-geo转化坐标
    const projection = d3geo.geoMercator().center([104.0, 37.5]).scale(80).translate([0, 0])
    let data = null
    if (code) {
        data = await getMapData(code)
    } else {
        data = chinaJson
    }
    // 遍历省份构建模型
    data.features.forEach(elem => {
        // 新建一个省份容器：用来存放省份对应的模型和轮廓线
        const province = new THREE.Object3D()
        const coordinates = elem.geometry.coordinates
        coordinates.forEach(multiPolygon => {
            multiPolygon.forEach(polygon => {
                // 这里的坐标要做2次使用：1次用来构建模型，1次用来构建轮廓线
                const shape = new THREE.Shape()
                const lineMaterial = new THREE.LineBasicMaterial({ color: 0xffffff })
                const linePoints = []
                for (let i = 0; i < polygon.length; i++) {
                    const [x, y] = projection(polygon[i])
                    if (i === 0) {
                        shape.moveTo(x, -y)
                    }
                    shape.lineTo(x, -y)
                    linePoints.push(new THREE.Vector3(x, -y, 4.01))
                }
                const extrudeSettings = {
                    depth: 4,
                    bevelEnabled: false
                }
                const geometry = new THREE.ExtrudeGeometry(shape, extrudeSettings)
                const material = new THREE.MeshLambertMaterial({ color: '#d13a34' })
                const mesh = new THREE.Mesh(geometry, material)
                const lineGeometry = new THREE.BufferGeometry().setFromPoints(linePoints)
                const line = new THREE.Line(lineGeometry, lineMaterial)
                province.add(mesh)
                province.add(line)
            })
        })
        // 将geojson的properties放到模型中，后面会用到
        province.properties = elem.properties
        if (elem.properties.centroid) {
            const [x, y] = projection(elem.properties.centroid)
            province.properties._centroid = [x, y]
            province.properties.adcode = elem.properties.adcode
        }
        map.add(province)
    })
    scene.add(map)
    cancelLoop()
    loop()
}

const setCamera = () => {
    camera = new THREE.PerspectiveCamera(35, window.innerWidth / window.innerHeight, 1, 10000)
    camera.position.set(0, -70, 90)
    camera.up = new THREE.Vector3(0, 0, 1)
    camera.lookAt(0, 0, 0)
    window.camera = camera
}
    
// 显示名称
const showName = () => {
    const width = window.innerWidth
    const height = window.innerHeight
    let canvas = document.querySelector('#name')
    if (!canvas) return
    canvas.width = width
    canvas.height = height
    const ctx = canvas.getContext('2d')
    // 新建一个离屏canvas
    const offCanvas = document.createElement('canvas')
    offCanvas.width = width
    offCanvas.height = height
    const ctxOffCanvas = canvas.getContext('2d')
    // 设置canvas字体样式
    ctxOffCanvas.font = '16.5px Arial'
    ctxOffCanvas.strokeStyle = '#FFFFFF'
    ctxOffCanvas.fillStyle = '#000000'
    // texts用来存储显示的名称，重叠的部分就不会放到里面
    const texts = []
    /**
     * 遍历省份数据，有2个核心功能
     * 1. 将3维坐标转化成2维坐标
     * 2. 后面遍历到的数据，要和前面的数据做碰撞对比，重叠的就不绘制
     * */
    map.children.forEach((elem, index) => {
        if (!elem.properties._centroid) return
        // 找到中心点
        const y = -elem.properties._centroid[1]
        const x = elem.properties._centroid[0]
        const z = 4
        // 转化为二维坐标
        const vector = new THREE.Vector3(x, y, z)
        const position = vector.project(camera)
        // 构建文本的基本属性：名称，left, top, width, height -> 碰撞对比需要这些坐标数据
        const name = elem.properties.name
        const left = (position.x + 1) / 2 * width
        const top = -(position.y - 1) / 2 * height
        const text = {
            name,
            left,
            top,
            width: ctxOffCanvas.measureText(name).width,
            height: 16.5
        }
        // 碰撞对比
        let show = true
        for (let i = 0; i < texts.length; i++) {
            if (
                (text.left + text.width) < texts[i].left ||
                (text.top + text.height) < texts[i].top ||
                (texts[i].left + texts[i].width) < text.left ||
                (texts[i].top + texts[i].height) < text.top
            ) {
                show = true
            } else {
                show = false
                break
            }
        }
        if (show) {
            texts.push(text)
            ctxOffCanvas.strokeText(name, left, top)
            ctxOffCanvas.fillText(name, left, top)
        }
    })
    // 离屏canvas绘制到canvas中
    ctx.drawImage(offCanvas, 0, 0)
}

// 构建辅助系统: 网格和坐标
const buildAuxSystem = () => {
    // let axisHelper = new THREE.AxesHelper(2000)
    // scene.add(axisHelper)
    // let gridHelper = new THREE.GridHelper(600, 60)
    // scene.add(gridHelper)

    controls = new OrbitControls(camera, renderer.domElement)
    controls.enableDamping = true
    controls.dampingFactor = 0.25
    controls.rotateSpeed = 0.35
    controls.mouseButtons = {
        LEFT: THREE.MOUSE.PAN,
        MIDDLE:THREE.MOUSE.DOLLY,
        RIGHT:THREE.MOUSE.ROTATE
    }

    // var controls = new TrackballControls( camera, renderer.domElement )
    // var dragControls = new DragControls( map, camera, renderer.domElement )

    // dragControls.addEventListener( 'dragstart', function ( event ) { controls.enabled = false } )
    // dragControls.addEventListener( 'dragend', function ( event ) { controls.enabled = true } )
}

// 光照系统
const buildLightSystem = () => {
    let directionalLight = new THREE.DirectionalLight(0xffffff, 1.1)
    directionalLight.position.set(0, -70, 90)
    directionalLight.target.position.set(0, 0, 0)
    directionalLight.castShadow = true

    let d = 300
    const fov = 45 //拍摄距离  视野角值越大，场景中的物体越小
    const near = 1 //相机离视体积最近的距离
    const far = 1000//相机离视体积最远的距离
    const aspect = window.innerWidth / window.innerHeight //纵横比
    directionalLight.shadow.camera = new THREE.PerspectiveCamera(fov, aspect, near, far)
    directionalLight.shadow.bias = 0.0001
    directionalLight.shadow.mapSize.width = directionalLight.shadow.mapSize.height = 1024
    scene.add(directionalLight)

    let light = new THREE.AmbientLight(0xffffff, 0.6)
    scene.add(light)

}

// 根据浏览器窗口变化动态更新尺寸
const onWindowResize =  () => {
    camera.aspect = window.innerWidth / window.innerHeight
    camera.updateProjectionMatrix()
    renderer.setSize(window.innerWidth, window.innerHeight)
}

const onDocumentMouseMove = event => {
    if (ISDOWN) ISDRAG = true
    event.preventDefault()
    mouse.x = ( event.clientX / window.innerWidth ) * 2 - 1
    mouse.y = -( event.clientY / window.innerHeight ) * 2 + 1
}

// 动画循环
const loop = () => {
    showName()
    TWEEN.update()
    renderCanvas()
    requestAF = requestAnimationFrame(loop)
}

// 取消循环
const cancelLoop = () => {
    cancelAnimationFrame(requestAF)
}

// 渲染画布
const renderCanvas = () => {
    renderer.render(scene, camera)
    if (!ISDRAG) {
        raycaster.setFromCamera(mouse, camera)
        let intersects = raycaster.intersectObjects(scene.children).filter(v => v.object.geometry.type === 'ExtrudeGeometry')
        if (intersects.length > 0) {
            if (INTERSECTED != intersects[0].object) {
                if (INTERSECTED) {
                    INTERSECTED.material.color.setHex(INTERSECTED.currentHex)
                }
                INTERSECTED = intersects[0].object
                INTERSECTED.currentHex = INTERSECTED.material.color.getHex()
                INTERSECTED.material.color.set(0xff0000)
            }
        } else {
            if (INTERSECTED) {
                INTERSECTED.material.color.set(INTERSECTED.currentHex)
            }
            INTERSECTED = null
        }
    }

}

onMounted(() => {
    // 初始化3D环境
    initEnvironment()
    // 构建光照系统
    buildLightSystem()
    // 构建辅助系统
    buildAuxSystem()
    initMap()

    // 天地图
    // const map = new T.Map('container')
    // map.centerAndZoom(new T.LngLat(116.40769, 39.89945), 12)

})

</script>
<style lang="scss" scoped>
.container {
    position: absolute;
    width: 100%;
    height: 100%;
    text-align: center;
    canvas {
        position: absolute;
        top: 0;
        left: 0;
        &#name {
            user-select: none;
            pointer-events: none;
        }
    }
}
</style>
