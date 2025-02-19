<script setup>
import { onMounted, onBeforeUnmount, ref } from 'vue'
import * as THREE from 'three'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'
import { RGBELoader } from 'three/examples/jsm/loaders/RGBELoader.js'
import AmmoLib from './assets/libs/ammo'
import { gsap } from 'gsap'
import LoadingIco from './components/LoadingIco.vue'
import PageWrap from './components/PageWrap.vue'
import Header from './components/Header.vue'
import FooterInfo from './components/FooterInfo.vue'

function init() {
    initGraphics()
    initPhysics()
    createObjects()
}

// Ref
const webgl = ref(null)

// Scene
const scene = new THREE.Scene()
scene.background = new THREE.Color('#cccccc')

// Clock for consistent animations
const clock = new THREE.Clock()

// Resize
const sizes = {
    width: window.innerWidth,
    height: window.innerHeight
}

const handleResize = () => {
    sizes.width = window.innerWidth
    sizes.height = window.innerHeight
    camera.aspect = sizes.width / sizes.height
    camera.updateProjectionMatrix()
    renderer.setSize(sizes.width, sizes.height)
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
}

// Renderer
const renderer = new THREE.WebGLRenderer({ antialias: true })
renderer.setSize(sizes.width, sizes.height)
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))

const camera = new THREE.PerspectiveCamera(45, sizes.width / sizes.height, 0.1, 1000)
camera.position.set(0, 0, 23)
camera.lookAt(new THREE.Vector3(0, 0, 0))
scene.add(camera)

function initGraphics() {
    // const handleResize = () => {
    //     sizes.width = window.innerWidth
    //     sizes.height = window.innerHeight
    //     camera.aspect = sizes.width / sizes.height
    //     camera.updateProjectionMatrix()
    //     renderer.setSize(sizes.width, sizes.height)
    //     renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
    // }
    window.addEventListener('resize', handleResize)

    renderer.shadowMap.enabled = true

    // Add ambient light
    const ambientLight = new THREE.AmbientLight(0xffffff, 0.6)
    scene.add(ambientLight)

    // Add directional light
    let dirLightA = new THREE.DirectionalLight(0xffffff, 0.8)
    dirLightA.position.set(-1, 3, 6)
    dirLightA.shadow.mapSize.width = 1024
    dirLightA.shadow.mapSize.height = 1024
    dirLightA.shadow.camera.near = 6
    dirLightA.shadow.camera.far = 30
    scene.add(dirLightA)

    // Add spotLight light
    let spotLight = new THREE.SpotLight(0xffffff, 290)
    spotLight.angle = Math.PI / 8
    spotLight.penumbra = 0.9
    spotLight.position.set(-2.5, 0.5, 15)
    spotLight.castShadow = true
    spotLight.shadow.camera.near = 8
    spotLight.shadow.camera.far = 30
    spotLight.shadow.mapSize.width = 1024
    spotLight.shadow.mapSize.height = 1024
    scene.add(spotLight)
}

// overlay for loading...
const overlayGeometry = new THREE.PlaneGeometry(2, 2, 1, 1)
const overlayMaterial = new THREE.ShaderMaterial({
    transparent: true,
    uniforms: { uAlpha: { value: 1 } },
    vertexShader: `
        void main() {
            gl_Position = vec4(position, 1.0);
        }
    `,
    fragmentShader: `
        uniform float uAlpha;

        void main() {
            gl_FragColor = vec4(0.0, 0.0, 0.0, uAlpha);
        }
    `
})
const overlay = new THREE.Mesh(overlayGeometry, overlayMaterial)
scene.add(overlay)

const loading = ref(true)
const loadingManager = new THREE.LoadingManager(
    () => {
        gsap.to(overlayMaterial.uniforms.uAlpha, { duration: 3, value: 0 })
        loading.value = false
    },
    (url, loaded, total) => {
        const progress = loaded / total
        console.log(`Loading: ${progress * 100}%`)
    }
)

// Ammojs Initialization
// Physics variables
let Ammo
const gravityConstant = 1
let collisionConfiguration
let dispatcher
let broadphase
let solver
let softBodySolver
let physicsWorld
const rigidBodies = []
const margin = 0
let rope
let transformAux1

const ballMaterial = new THREE.MeshPhongMaterial({ color: 0x202020 })

const pos = new THREE.Vector3()
const quat = new THREE.Quaternion()

function initPhysics() {
    collisionConfiguration = new Ammo.btSoftBodyRigidBodyCollisionConfiguration()
    dispatcher = new Ammo.btCollisionDispatcher(collisionConfiguration)
    broadphase = new Ammo.btDbvtBroadphase()
    solver = new Ammo.btSequentialImpulseConstraintSolver()
    softBodySolver = new Ammo.btDefaultSoftBodySolver()
    physicsWorld = new Ammo.btSoftRigidDynamicsWorld(
        dispatcher,
        broadphase,
        solver,
        collisionConfiguration,
        softBodySolver
    )
    physicsWorld.setGravity(new Ammo.btVector3(0, gravityConstant, 0))
    physicsWorld.getWorldInfo().set_m_gravity(new Ammo.btVector3(0, gravityConstant, 0))
    transformAux1 = new Ammo.btTransform()
}

function createObjects() {
    // Wall
    const geometry = new THREE.PlaneGeometry(10, 10)
    const material = new THREE.MeshPhongMaterial({ color: '#cccccc' })
    const wall = new THREE.Mesh(geometry, material)
    wall.position.z = -10
    wall.scale.set(10, 10, 10)
    wall.receiveShadow = true
    scene.add(wall)

    // Ground
    pos.set(0, -7, 0)
    quat.set(0, 0, 0, 1)
    const groundMass = 0
    const ground = createParalellepiped(
        3.5,
        7,
        3.5,
        groundMass,
        pos,
        quat,
        new THREE.MeshPhongMaterial({ color: 0xffffff, wireframe: false })
    )

    ground.castShadow = true

    const GropePos = ground.position.clone()

    function createParalellepiped(sx, sy, sz, mass, pos, quat, material) {
        const threeObject = new THREE.Mesh(new THREE.BoxGeometry(sx, sy, sz, 1, 1, 1), material)
        const shape = new Ammo.btBoxShape(new Ammo.btVector3(sx * 0.5, sy * 0.5, sz * 0.5))
        shape.setMargin(margin)

        createRigidBody(threeObject, shape, mass, pos, quat)

        return threeObject
    }

    // Balloon & Rope
    let balloonMesh = null
    let balloonChild = null

    const balloonMass = 4
    const balloonRadius = 2.1
    const gltfLoader = new GLTFLoader(loadingManager)
    const hdrEquirect = new RGBELoader(loadingManager)

    // Balloon
    hdrEquirect.load('./photo_studio_01_1k.hdr', (texture) => {
        texture.mapping = THREE.EquirectangularReflectionMapping
        gltfLoader.load('./balloon.glb', (gltf) => {
            gltf.scene.traverse(function (node) {
                if (node.isMesh || node.isLight) node.castShadow = true
            })
            balloonMesh = gltf.scene
            balloonChild = balloonMesh.getObjectByName('balloon')
            balloonChild.material = new THREE.MeshPhysicalMaterial({
                color: 0x000000,
                envMap: texture,
                envMapIntensity: 1,
                roughness: 0.05,
                metalness: 0.95,
                wireframe: false
            })
            balloonMesh.scale.set(2, 2, 2)

            const balloonShape = new Ammo.btSphereShape(balloonRadius)
            balloonShape.setMargin(margin)
            pos.set(0, 3, 0)
            quat.set(0, 0, 0, 1)
            createRigidBody(balloonMesh, balloonShape, balloonMass, pos, quat)
            // balloonMesh.userData.physicsBody.setFriction(5)

            // Rope
            const ropeNumSegments = 20
            const ropeLength = 4
            const ropeMass = 1
            const ropePos = balloonMesh.position.clone()
            ropePos.y -= balloonRadius

            const segmentLength = ropeLength / ropeNumSegments
            const ropeGeometry = new THREE.BufferGeometry()
            const ropeMaterial = new THREE.LineBasicMaterial({ color: 0x000000 })
            const ropePositions = []
            const ropeIndices = []

            for (let i = 0; i < ropeNumSegments + 1; i++) {
                ropePositions.push(ropePos.x, ropePos.y + i * segmentLength, ropePos.z)
            }

            for (let i = 0; i < ropeNumSegments; i++) {
                ropeIndices.push(i, i + 1)
            }

            ropeGeometry.setIndex(new THREE.BufferAttribute(new Uint16Array(ropeIndices), 1))
            ropeGeometry.setAttribute('position', new THREE.BufferAttribute(new Float32Array(ropePositions), 3))
            ropeGeometry.computeBoundingSphere()
            rope = new THREE.LineSegments(ropeGeometry, ropeMaterial)
            rope.castShadow = true
            scene.add(rope)

            // Rope physic object
            const softBodyHelpers = new Ammo.btSoftBodyHelpers()
            const ropeStart = new Ammo.btVector3(GropePos.x, GropePos.y + 3.2, GropePos.z)
            const ropeEnd = new Ammo.btVector3(ropePos.x, ropePos.y, ropePos.z)

            const ropeSoftBody = softBodyHelpers.CreateRope(
                physicsWorld.getWorldInfo(),
                ropeStart,
                ropeEnd,
                ropeNumSegments - 1,
                0
            )
            const sbConfig = ropeSoftBody.get_m_cfg()
            sbConfig.set_viterations(5)
            sbConfig.set_piterations(5)
            ropeSoftBody.setTotalMass(ropeMass, false)
            Ammo.castObject(ropeSoftBody, Ammo.btCollisionObject)
                .getCollisionShape()
                .setMargin(margin * 3)
            physicsWorld.addSoftBody(ropeSoftBody, 1, -1)
            rope.userData.physicsBody = ropeSoftBody

            // Disable deactivation
            ropeSoftBody.setActivationState(4)

            // Glue the rope extremes to the ball and the ground box
            const influence = 1
            ropeSoftBody.appendAnchor(0, ground.userData.physicsBody, true, influence)
            ropeSoftBody.appendAnchor(ropeNumSegments, balloonMesh.userData.physicsBody, true, influence)
        })
    })
}

function createRigidBody(threeObject, physicsShape, mass, pos, quat) {
    threeObject.position.copy(pos)
    threeObject.quaternion.copy(quat)

    const transform = new Ammo.btTransform()
    transform.setIdentity()
    transform.setOrigin(new Ammo.btVector3(pos.x, pos.y, pos.z))
    transform.setRotation(new Ammo.btQuaternion(quat.x, quat.y, quat.z, quat.w))
    const motionState = new Ammo.btDefaultMotionState(transform)

    const localInertia = new Ammo.btVector3(0, 0, 0)
    physicsShape.calculateLocalInertia(mass, localInertia)

    const rbInfo = new Ammo.btRigidBodyConstructionInfo(mass, motionState, physicsShape, localInertia)
    const body = new Ammo.btRigidBody(rbInfo)

    threeObject.userData.physicsBody = body

    scene.add(threeObject)

    if (mass > 0) {
        rigidBodies.push(threeObject)

        // Disable deactivation
        body.setActivationState(4)
    }

    physicsWorld.addRigidBody(body)
}

function updatePhysics(deltaTime) {
    physicsWorld.stepSimulation(deltaTime, 10)

    // Update rope
    if (rope) {
        const softBody = rope.userData.physicsBody
        const ropePositions = rope.geometry.attributes.position.array
        const numVerts = ropePositions.length / 3
        const nodes = softBody.get_m_nodes()
        let indexFloat = 0

        for (let i = 0; i < numVerts; i++) {
            const node = nodes.at(i)
            const nodePos = node.get_m_x()
            ropePositions[indexFloat++] = nodePos.x()
            ropePositions[indexFloat++] = nodePos.y()
            ropePositions[indexFloat++] = nodePos.z()
        }

        rope.geometry.attributes.position.needsUpdate = true
    }

    // Update rigid bodies
    for (let i = 0, il = rigidBodies.length; i < il; i++) {
        const objThree = rigidBodies[i]
        const objPhys = objThree.userData.physicsBody
        const ms = objPhys.getMotionState()
        if (ms) {
            ms.getWorldTransform(transformAux1)
            const p = transformAux1.getOrigin()
            const q = transformAux1.getRotation()
            objThree.position.set(p.x(), p.y(), p.z())
            objThree.quaternion.set(q.x(), q.y(), q.z(), q.w())
        }
    }
}

// input

let clickRequest = false
window.addEventListener('mousedown', () => !clickRequest && (clickRequest = true), false)
window.addEventListener('touchstart', () => !clickRequest && (clickRequest = true), false)

function processClick() {
    if (clickRequest) {
        // Creates a ball
        let ballMass = 0.3
        let ballRadius = 2.2
        let sphereGeo = new THREE.SphereGeometry(ballRadius, 18, 16)
        let ballMesh = new THREE.Mesh(sphereGeo, ballMaterial)
        let ballShape = new Ammo.btSphereShape(ballRadius)
        ballShape.setMargin(margin)

        pos.set(-1, 3.5, 0)
        quat.set(0, 0, 0, 1)
        let ballBody = createRigidBody(ballMesh, ballShape, ballMass, pos, quat)

        ballMesh.visible = false

        setTimeout(function () {
            physicsWorld.removeRigidBody(ballBody)
            scene.remove(ballMesh)
        }, 200)

        clickRequest = false
    }
}

const tick = () => {
    const deltaTime = clock.getDelta()

    updatePhysics(deltaTime)
    processClick()

    renderer.render(scene, camera)

    window.requestAnimationFrame(tick)
}

onMounted(() => {
    webgl.value.appendChild(renderer.domElement)

    // Ammo
    AmmoLib().then(function (re) {
        Ammo = re
        init()
        tick()
    })
})

onBeforeUnmount(() => {
    window.removeEventListener('resize', handleResize)
})
</script>

<template>
    <PageWrap>
        <Header />
        <div v-if="loading" class="z-10 h-screen inset-0 flex items-center justify-center">
            <LoadingIco />
        </div>
        <div class="outline-none w-full h-dvh" ref="webgl"></div>

        <FooterInfo>
            <template v-slot:first></template>
            <template v-slot:second>
                <a href="https://jirigeller.com/Supersculptures/happy/" target="_blank" class="underline-offset-2"
                    >Jiri Geller HAPPY SERIES</a
                >
            </template>
            <template v-slot:github>
                <a href="https://github.com/pccathome/happy-balloon" target="_blank" class="underline-offset-2"
                    >GitHub</a
                >
            </template>
        </FooterInfo>
    </PageWrap>
</template>
