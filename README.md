game.js
import * as THREE from 'three';
import { PointerLockControls } from 'three/addons/controls/PointerLockControls.js';

// --- НАСТРОЙКИ ---
const CHUNK_SIZE = 16;
const RENDER_DISTANCE = 3; 
const TEXTURE_PATH = './Assets/';
const GRAVITY = 32.0;
const JUMP_FORCE = 10.0;
const SPEED = 5.0;
const PLAYER_HEIGHT = 1.7;
const PLAYER_RADIUS = 0.3;
const PICKUP_RANGE = 2.0;

// --- СЦЕНА ---
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x87CEEB);
scene.fog = new THREE.Fog(0x87CEEB, 20, (RENDER_DISTANCE * CHUNK_SIZE) - 5);

const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
scene.add(camera); // Add camera to scene so children (hand) are visible
const renderer = new THREE.WebGLRenderer({ antialias: false });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(window.devicePixelRatio);
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;
document.body.appendChild(renderer.domElement);

// --- РЕСУРСЫ ---
const loader = new THREE.TextureLoader();
const loadTex = (url) => {
    const tex = loader.load(TEXTURE_PATH + url);
    tex.magFilter = THREE.NearestFilter;
    tex.minFilter = THREE.NearestFilter;
    tex.colorSpace = THREE.SRGBColorSpace;
    return tex;
};

const textures = {
    dirt: loadTex('Dirt.png'),
    grass: loadTex('grass.png'), 
    stone: loadTex('stone.png'),
    wood: loadTex('tree.jpg'),
    leaves: loadTex('foliage.jpg'),
    sand: loadTex('sand.png'),
    planks: loadTex('planks.png'),
    oreCoal: loadTex('ore_coal.png'),
    oreIron: loadTex('ore_iron.png'),
    oreGold: loadTex('ore_gold.png'),
    oreDiamond: loadTex('ore_diamond.png'),
    craftingTable: loadTex('crafting_table.png'),
    chest: loadTex('chest.png')
};

const materials = [];
materials[1] = new THREE.MeshLambertMaterial({ map: textures.grass });
materials[2] = new THREE.MeshLambertMaterial({ map: textures.dirt });
materials[3] = new THREE.MeshLambertMaterial({ map: textures.stone });
materials[4] = new THREE.MeshLambertMaterial({ map: textures.wood });
materials[5] = new THREE.MeshLambertMaterial({ map: textures.leaves, transparent: true, alphaTest: 0.5 });
materials[6] = new THREE.MeshBasicMaterial({ color: 0xffffff, transparent: true, opacity: 0.8 });
materials[10] = new THREE.MeshLambertMaterial({ map: textures.sand });
materials[11] = new THREE.MeshLambertMaterial({ map: textures.planks });
materials[12] = new THREE.MeshLambertMaterial({ map: textures.oreCoal });
materials[13] = new THREE.MeshLambertMaterial({ map: textures.oreIron });
materials[14] = new THREE.MeshLambertMaterial({ map: textures.oreGold });
materials[15] = new THREE.MeshLambertMaterial({ map: textures.oreDiamond });
materials[17] = new THREE.MeshLambertMaterial({ map: textures.craftingTable });
materials[18] = new THREE.MeshLambertMaterial({ map: textures.chest });

const itemIcons = {
    1: './Assets/grass.png',
    2: './Assets/Dirt.png',
    3: './Assets/stone.png',
    4: './Assets/tree.jpg',
    5: './Assets/foliage.jpg',
    10: './Assets/sand.png',
    11: './Assets/planks.png',
    12: './Assets/ore_coal.png',
    13: './Assets/ore_iron.png',
    14: './Assets/ore_gold.png',
    15: './Assets/ore_diamond.png',
    17: './Assets/crafting_table.png',
    18: './Assets/chest.png'
};

const blockBreakSound = new Audio('./Assets/sound of a block breaking.mp3');
let soundVolume = 0.5;
blockBreakSound.volume = soundVolume;

// --- ИНВЕНТАРЬ ---
// 9 хотбар + 27 инвентарь = 36
const inventory = new Array(36).fill(null); 
// null или { type: int, count: int }
let cursorItem = null;
let isSprinting = false;

const STARTER_ITEMS = [
  { slot: 0, type: 1, count: 32 },
  { slot: 1, type: 2, count: 32 },
  { slot: 2, type: 3, count: 32 },
  { slot: 3, type: 4, count: 24 },
  { slot: 4, type: 5, count: 24 },
  { slot: 5, type: 10, count: 32 },
  { slot: 6, type: 11, count: 32 },
  { slot: 7, type: 12, count: 16 },
  { slot: 8, type: 13, count: 16 },
  { slot: 9, type: 14, count: 8 },
  { slot: 10, type: 15, count: 4 },
  { slot: 11, type: 17, count: 4 },
  { slot: 12, type: 18, count: 4 }
];
for (const starter of STARTER_ITEMS) {
  inventory[starter.slot] = { type: starter.type, count: starter.count };
}

// --- HAND (Minecraft style) ---
const handScene = new THREE.Scene();
const handCamera = new THREE.PerspectiveCamera(70, window.innerWidth / window.innerHeight, 0.01, 10);
handCamera.position.set(0, 0, 0);

// Add light to hand scene
const handAmbient = new THREE.AmbientLight(0xffffff, 0.8);
handScene.add(handAmbient);
const handDirLight = new THREE.DirectionalLight(0xffffff, 0.5);
handDirLight.position.set(1, 1, 1);
handScene.add(handDirLight);

// Arm (Steve skin colored)
const armGeo = new THREE.BoxGeometry(0.15, 0.5, 0.15);
const armMat = new THREE.MeshLambertMaterial({ color: 0xc9a065 });
const armMesh = new THREE.Mesh(armGeo, armMat);

// Block in hand
const blockInHandGeo = new THREE.BoxGeometry(0.2, 0.2, 0.2);
const blockInHandMesh = new THREE.Mesh(blockInHandGeo, materials[2]);
blockInHandMesh.visible = false;
blockInHandMesh.position.set(0, 0.15, -0.05);

// Hand pivot for animation
const handPivot = new THREE.Group();
handPivot.add(armMesh);
handPivot.add(blockInHandMesh);

// Position hand in bottom right of screen
handPivot.position.set(0.4, -0.35, -0.5);
handPivot.rotation.set(-0.1, -0.3, 0);
armMesh.position.set(0, -0.1, 0);

handScene.add(handPivot);

let handSwingTime = 0;
const HAND_SWING_DURATION = 0.25;
let handBaseRotX = -0.1;
let handBaseRotZ = 0;

function updateHand(dt) {
    // Swing animation
    if (handSwingTime > 0) {
        handSwingTime -= dt;
        const t = 1 - (handSwingTime / HAND_SWING_DURATION);
        // Swing down and to the left
        const swing = Math.sin(t * Math.PI);
        handPivot.rotation.x = handBaseRotX + swing * 0.6;
        handPivot.rotation.z = handBaseRotZ - swing * 0.4;
    } else {
        // Idle bobbing
        const bobTime = performance.now() * 0.003;
        handPivot.rotation.x = handBaseRotX + Math.sin(bobTime) * 0.02;
        handPivot.rotation.z = handBaseRotZ + Math.cos(bobTime * 0.7) * 0.01;
    }
    
    // Update block in hand based on selected slot
    const item = inventory[activeSlotIndex];
    if (item && materials[item.type]) {
        armMesh.visible = false;
        blockInHandMesh.visible = true;
        blockInHandMesh.material = materials[item.type];
    } else {
        armMesh.visible = true;
        blockInHandMesh.visible = false;
    }
}

function swingHand() {
    if (handSwingTime <= 0) {
        handSwingTime = HAND_SWING_DURATION;
    }
}

// --- CLOUDS (Smooth moving plane) ---
const CLOUD_HEIGHT = 45;
const CLOUD_SPEED = 2; // blocks per second
let cloudOffset = 0;

// Create cloud texture procedurally
const cloudCanvas = document.createElement('canvas');
cloudCanvas.width = 256;
cloudCanvas.height = 256;
const cloudCtx = cloudCanvas.getContext('2d');

// Draw Minecraft-style cloud pattern
cloudCtx.fillStyle = 'rgba(0,0,0,0)';
cloudCtx.fillRect(0, 0, 256, 256);
cloudCtx.fillStyle = 'rgba(255,255,255,0.9)';

// Create blocky cloud shapes
const cloudPattern = [
    [0,0,1,1,1,1,1,0,0,0,0,0,1,1,1,0],
    [0,1,1,1,1,1,1,1,0,0,0,1,1,1,1,1],
    [1,1,1,1,1,1,1,1,1,0,1,1,1,1,1,1],
    [1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1],
    [0,1,1,1,1,1,1,1,1,1,1,1,1,1,1,0],
    [0,0,1,1,1,1,1,1,1,1,1,1,1,0,0,0],
    [0,0,0,0,1,1,1,1,1,1,0,0,0,0,0,0],
    [0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0],
    [0,0,0,0,0,1,1,1,1,0,0,0,0,0,0,0],
    [0,0,0,0,1,1,1,1,1,1,0,0,1,1,0,0],
    [0,0,0,1,1,1,1,1,1,1,1,1,1,1,1,0],
    [0,0,1,1,1,1,1,1,1,1,1,1,1,1,1,1],
    [0,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1],
    [0,1,1,1,1,1,1,1,1,1,1,1,1,1,0,0],
    [0,0,1,1,1,1,1,0,0,1,1,1,1,0,0,0],
    [0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
];

for (let y = 0; y < 16; y++) {
    for (let x = 0; x < 16; x++) {
        if (cloudPattern[y][x]) {
            cloudCtx.fillRect(x * 16, y * 16, 16, 16);
        }
    }
}

const cloudTexture = new THREE.CanvasTexture(cloudCanvas);
cloudTexture.wrapS = THREE.RepeatWrapping;
cloudTexture.wrapT = THREE.RepeatWrapping;
cloudTexture.magFilter = THREE.NearestFilter;
cloudTexture.minFilter = THREE.NearestFilter;
cloudTexture.repeat.set(8, 8);

const cloudGeo = new THREE.PlaneGeometry(500, 500);
const cloudMat = new THREE.MeshBasicMaterial({
    map: cloudTexture,
    transparent: true,
    opacity: 0.9,
    side: THREE.DoubleSide,
    depthWrite: false
});
const cloudMesh = new THREE.Mesh(cloudGeo, cloudMat);
cloudMesh.rotation.x = -Math.PI / 2;
cloudMesh.position.y = CLOUD_HEIGHT;
scene.add(cloudMesh);

function updateClouds(dt) {
    cloudOffset += CLOUD_SPEED * dt * 0.01;
    cloudTexture.offset.x = cloudOffset;
    // Follow player horizontally
    cloudMesh.position.x = player.pos.x;
    cloudMesh.position.z = player.pos.z;
}

// --- СВЕТ ---
const ambientLight = new THREE.AmbientLight(0xffffff, 0.7);
scene.add(ambientLight);

const dirLight = new THREE.DirectionalLight(0xffffff, 0.6);
dirLight.position.set(100, 200, 50);
dirLight.castShadow = true;
dirLight.shadow.mapSize.width = 2048;
dirLight.shadow.mapSize.height = 2048;
scene.add(dirLight);

// --- ГЕНЕРАЦИЯ ---
const simplex = new SimplexNoise();
const chunks = new Map(); 
const savedChunkMods = new Map(); 

function getChunkKey(x, z) { return `${x},${z}`; }

class Chunk {
    constructor(cx, cz) {
        this.cx = cx;
        this.cz = cz;
        this.blocks = new Map(); 
        this.meshGroup = new THREE.Group();
        this.meshGroup.position.set(cx * CHUNK_SIZE, 0, cz * CHUNK_SIZE);
        
        this.generate();
        this.applyMods();
        this.buildMesh();
        
        scene.add(this.meshGroup);
    }

    generate() {
        for (let x = 0; x < CHUNK_SIZE; x++) {
            for (let z = 0; z < CHUNK_SIZE; z++) {
                const wx = this.cx * CHUNK_SIZE + x;
                const wz = this.cz * CHUNK_SIZE + z;
                
                const scale = 0.02;
                const noise = simplex.noise2D(wx * scale, wz * scale);
                const h = Math.floor((noise + 1) * 8 + 4); 

                for (let y = 0; y <= h; y++) {
                    let type = 2; // Dirt
                    const belowSurface = h - y;

                    if (y === h) {
                        type = h <= 6 ? 10 : 1;
                    } else if (belowSurface <= 2 && h <= 6) {
                        type = 10;
                    } else if (y < h - 4) {
                        type = 3;

                        const oreRoll = simplex.noise3D(wx * 0.09, wz * 0.09, y * 0.09);
                        if (y < 20 && oreRoll > 0.55) type = 12;
                        if (y < 16 && oreRoll > 0.68) type = 13;
                        if (y < 10 && oreRoll > 0.8) type = 14;
                        if (y < 6 && oreRoll > 0.9) type = 15;
                    }
                    this.setBlockLocal(x, y, z, type);
                }

                if (x > 2 && x < 13 && z > 2 && z < 13 && Math.random() < 0.01 && h > 4) {
                   this.buildTree(x, h + 1, z);
                }
            }
        }
        
    }

    applyMods() {
        const key = getChunkKey(this.cx, this.cz);
        if (savedChunkMods.has(key)) {
            const mods = savedChunkMods.get(key);
            for (const m of mods) {
                this.setBlockLocal(m.x, m.y, m.z, m.type);
            }
        }
    }

    buildTree(lx, ly, lz) {
        const height = 4 + Math.floor(Math.random() * 3);
        for (let i = 0; i < height; i++) {
            this.setBlockLocal(lx, ly + i, lz, 4);
        }
        for (let y = ly + height - 3; y <= ly + height; y++) {
            const range = (y >= ly + height - 1) ? 1 : 2; 
            for (let x = lx - range; x <= lx + range; x++) {
                for (let z = lz - range; z <= lz + range; z++) {
                    if (x === lx && z === lz && y < ly + height) continue; 
                    if ((Math.abs(x - lx) === range && Math.abs(z - lz) === range) && (y > ly + height - 2 || Math.random() < 0.3)) continue;
                    this.setBlockLocal(x, y, z, 5);
                }
            }
        }
    }

    setBlockLocal(x, y, z, type) {
        const key = `${x},${y},${z}`;
        if (type === 0) this.blocks.delete(key);
        else this.blocks.set(key, type);
    }

    getBlockLocal(x, y, z) {
        return this.blocks.get(`${x},${y},${z}`) || 0;
    }

    buildMesh() {
        this.meshGroup.clear();
        
        const matrices = {};
        for (let i = 1; i < materials.length; i++) {
            if (materials[i]) matrices[i] = [];
        }

        const dummy = new THREE.Object3D();

        for (const [key, type] of this.blocks) {
            const [x, y, z] = key.split(',').map(Number);
            if (type !== 5 && type !== 6 && this.isOccluded(x, y, z)) continue;
            dummy.position.set(x + 0.5, y + 0.5, z + 0.5);
            dummy.updateMatrix();
            if(matrices[type]) matrices[type].push(dummy.matrix.clone());
        }

        const geom = new THREE.BoxGeometry(1, 1, 1);
        
        for (const type in matrices) {
            const arr = matrices[type];
            if (arr.length === 0) continue;
            const mesh = new THREE.InstancedMesh(geom, materials[type], arr.length);
            for (let i = 0; i < arr.length; i++) {
                mesh.setMatrixAt(i, arr[i]);
            }
            mesh.castShadow = true;
            mesh.receiveShadow = true;
            this.meshGroup.add(mesh);
        }
    }

    isOccluded(x, y, z) {
        const neighbors = [[1,0,0], [-1,0,0], [0,1,0], [0,-1,0], [0,0,1], [0,0,-1]];
        for (let n of neighbors) {
            const t = this.getBlockLocal(x+n[0], y+n[1], z+n[2]);
            if (t === 0 || t === 5 || t === 6) return false;
        }
        return true; 
    }

    dispose() {
        scene.remove(this.meshGroup);
        this.meshGroup.traverse(o => {
            if(o.geometry) o.geometry.dispose();
        });
    }
}

// --- DROPS ---
const drops = [];
const dropGeo = new THREE.BoxGeometry(0.3, 0.3, 0.3);

function spawnDrop(x, y, z, type) {
    if(!materials[type]) return; // don't spawn drops for invalid blocks
    const mesh = new THREE.Mesh(dropGeo, materials[type]);
    mesh.position.set(x + 0.5, y + 0.5, z + 0.5);
    mesh.castShadow = true;
    scene.add(mesh);
    drops.push({
        mesh: mesh,
        type: type,
        vel: new THREE.Vector3((Math.random()-0.5)*2, 5, (Math.random()-0.5)*2),
        creationTime: performance.now()
    });
}

function updateDrops(dt) {
    for (let i = drops.length - 1; i >= 0; i--) {
        const d = drops[i];
        d.vel.y -= GRAVITY * dt;
        d.mesh.position.addScaledVector(d.vel, dt);

        // Simple ground check
        if (world.getBlock(d.mesh.position.x, d.mesh.position.y - 0.15, d.mesh.position.z) !== 0) {
            d.vel.y = 0;
            d.vel.x *= 0.9;
            d.vel.z *= 0.9;
            d.mesh.position.y = Math.ceil(d.mesh.position.y - 0.5) + 0.15;
        }

        d.mesh.rotation.y += dt;

        // Pickup
        const dist = d.mesh.position.distanceTo(player.pos);
        if (dist < PICKUP_RANGE && (performance.now() - d.creationTime > 500)) {
            // Add to inventory
            if (addToInventory(d.type)) {
                scene.remove(d.mesh);
                drops.splice(i, 1);
            }
        }
    }
}

function addToInventory(type, amount = 1) {
    // Try calculate stack
    for(let i=0; i<inventory.length; i++) {
        if (inventory[i] && inventory[i].type === type && inventory[i].count < 64) {
            const space = 64 - inventory[i].count;
            const toAdd = Math.min(space, amount);
            inventory[i].count += toAdd;
            amount -= toAdd;
            if(amount <= 0) {
                renderUI();
                return true;
            }
        }
    }
    // Try empty slot
    while(amount > 0) {
        let emptyIdx = -1;
        for(let i=0; i<inventory.length; i++) {
            if (!inventory[i]) { emptyIdx = i; break; }
        }
        if (emptyIdx !== -1) {
            const toAdd = Math.min(64, amount);
            inventory[emptyIdx] = { type: type, count: toAdd };
            amount -= toAdd;
        } else {
             renderUI();
             return false;
        }
    }
    renderUI();
    return true;
}

// --- WORLD ---
const world = {
    getBlock: (wx, wy, wz) => {
        const cx = Math.floor(wx / CHUNK_SIZE);
        const cz = Math.floor(wz / CHUNK_SIZE);
        const lx = Math.floor(wx) - cx * CHUNK_SIZE;
        const ly = Math.floor(wy);
        const lz = Math.floor(wz) - cz * CHUNK_SIZE;
        
        const chunk = chunks.get(getChunkKey(cx, cz));
        if (!chunk) return 0;
        return chunk.getBlockLocal(lx, ly, lz);
    },
    setBlock: (wx, wy, wz, type) => {
        const cx = Math.floor(wx / CHUNK_SIZE);
        const cz = Math.floor(wz / CHUNK_SIZE);
        const lx = Math.floor(wx) - cx * CHUNK_SIZE;
        const ly = Math.floor(wy);
        const lz = Math.floor(wz) - cz * CHUNK_SIZE;

        const key = getChunkKey(cx, cz);
        
        if (!savedChunkMods.has(key)) savedChunkMods.set(key, []);
        const mods = savedChunkMods.get(key);
        const existIdx = mods.findIndex(m => m.x === lx && m.y === ly && m.z === lz);
        if (existIdx !== -1) mods.splice(existIdx, 1);
        mods.push({x:lx, y:ly, z:lz, type});

        const chunk = chunks.get(key);
        if (chunk) {
            // Spawn drop if breaking
            const currentBlock = chunk.getBlockLocal(lx, ly, lz);
            if (type === 0 && currentBlock !== 0 && currentBlock !== 6) {
                spawnDrop(wx, wy, wz, currentBlock);
            }
            
            chunk.setBlockLocal(lx, ly, lz, type);
            chunk.buildMesh();
        }
    },
    update: (px, pz) => {
        const cx = Math.floor(px / CHUNK_SIZE);
        const cz = Math.floor(pz / CHUNK_SIZE);

        for (let x = cx - RENDER_DISTANCE; x <= cx + RENDER_DISTANCE; x++) {
            for (let z = cz - RENDER_DISTANCE; z <= cz + RENDER_DISTANCE; z++) {
                const key = getChunkKey(x, z);
                if (!chunks.has(key)) {
                    chunks.set(key, new Chunk(x, z));
                }
            }
        }
        for (const [key, chunk] of chunks) {
            const dist = Math.sqrt((chunk.cx - cx)**2 + (chunk.cz - cz)**2);
            if (dist > RENDER_DISTANCE + 1) {
                chunk.dispose();
                chunks.delete(key);
            }
        }
    }
};

// --- GAMEPLAY ---
const controls = new PointerLockControls(camera, document.body);
const player = {
    pos: new THREE.Vector3(0, 30, 0),
    vel: new THREE.Vector3(),
    onGround: false
};
const keys = { w:false, a:false, s:false, d:false, space:false };

camera.position.copy(player.pos);

function checkCollision(newPos) {
    const x = newPos.x;
    const y = newPos.y;
    const z = newPos.z;

    const minX = Math.floor(x - PLAYER_RADIUS);
    const maxX = Math.floor(x + PLAYER_RADIUS);
    const minY = Math.floor(y); 
    const maxY = Math.floor(y + PLAYER_HEIGHT); 
    const minZ = Math.floor(z - PLAYER_RADIUS);
    const maxZ = Math.floor(z + PLAYER_RADIUS);

    for (let bx = minX; bx <= maxX; bx++) {
        for (let by = minY; by <= maxY; by++) {
            for (let bz = minZ; bz <= maxZ; bz++) {
                const block = world.getBlock(bx, by, bz);
                if (block !== 0) { 
                     return true;
                }
            }
        }
    }
    return false;
}

function updatePhysics(dt) {
    if (controls.isLocked) {
        player.vel.y -= GRAVITY * dt;

        const forward = new THREE.Vector3(0,0,-1).applyQuaternion(camera.quaternion);
        forward.y = 0; forward.normalize();
        const right = new THREE.Vector3(1,0,0).applyQuaternion(camera.quaternion);
        right.y = 0; right.normalize();

        const moveDir = new THREE.Vector3();
        if (keys.w) moveDir.add(forward);
        if (keys.s) moveDir.sub(forward);
        if (keys.d) moveDir.add(right);
        if (keys.a) moveDir.sub(right);

        if (moveDir.length() > 0) moveDir.normalize();
        else isSprinting = false; // Disable sprint if stopped
        
        player.vel.x -= player.vel.x * 10 * dt; 
        player.vel.z -= player.vel.z * 10 * dt;
        
        const currentSpeed = isSprinting ? SPEED * 1.8 : SPEED;
        player.vel.x += moveDir.x * currentSpeed * 10 * dt; 
        player.vel.z += moveDir.z * currentSpeed * 10 * dt;

        if (keys.space && player.onGround) {
            player.vel.y = JUMP_FORCE;
            player.onGround = false;
        }
        
        let nextX = player.pos.x + player.vel.x * dt;
        if (checkCollision(new THREE.Vector3(nextX, player.pos.y, player.pos.z))) {
            player.vel.x = 0; 
        } else {
            player.pos.x = nextX;
        }

        let nextZ = player.pos.z + player.vel.z * dt;
        if (checkCollision(new THREE.Vector3(player.pos.x, player.pos.y, nextZ))) {
            player.vel.z = 0;
        } else {
            player.pos.z = nextZ;
        }
        
        let nextY = player.pos.y + player.vel.y * dt;
        if (checkCollision(new THREE.Vector3(player.pos.x, nextY, player.pos.z))) {
             if (player.vel.y < 0) player.onGround = true; 
             player.vel.y = 0;
        } else {
            player.pos.y = nextY;
            player.onGround = false;
        }

        if (player.pos.y < -30) {
            player.pos.set(0, 40, 0);
            player.vel.set(0, 0, 0);
        }

        camera.position.copy(player.pos);
        camera.position.y += PLAYER_HEIGHT * 0.9; 
    }
}

const raycaster = new THREE.Raycaster();
raycaster.far = 5;
const center = new THREE.Vector2(0, 0);

const wireGeo = new THREE.BoxGeometry(1.005, 1.005, 1.005);
const wireMat = new THREE.LineBasicMaterial({ color: 0x000000, linewidth: 2 });
const wireMesh = new THREE.LineSegments(new THREE.EdgesGeometry(wireGeo), wireMat);
scene.add(wireMesh);
wireMesh.visible = false;

let highlightPos = null;
let buildPos = null;

function updateRaycaster() {
    raycaster.setFromCamera(center, camera);
    const targets = Array.from(chunks.values()).map(c => c.meshGroup);
    const intersects = raycaster.intersectObjects(targets, true); 
    
    if (intersects.length > 0) {
        for (let hit of intersects) {
            if (hit.object.isInstancedMesh) {
                const p = hit.point.clone().add(hit.face.normal.clone().multiplyScalar(-0.001));
                const bx = Math.floor(p.x);
                const by = Math.floor(p.y);
                const bz = Math.floor(p.z);
                
                wireMesh.position.set(bx + 0.5, by + 0.5, bz + 0.5);
                wireMesh.visible = true;
                highlightPos = { x: bx, y: by, z: bz };
                
                const bp = hit.point.clone().add(hit.face.normal.clone().multiplyScalar(0.001));
                buildPos = { x: Math.floor(bp.x), y: Math.floor(bp.y), z: Math.floor(bp.z) };
                return;
            }
        }
    }
    wireMesh.visible = false;
    highlightPos = null;
    buildPos = null;
}

// --- UI & EVENTS ---
let activeSlotIndex = 0; // 0-8
const menuOverlay = document.getElementById('menu-overlay');
const resumeBtn = document.getElementById('resume-btn');
const volumeSlider = document.getElementById('volume-slider');
const inventoryScreen = document.getElementById('inventory-screen');
const hotbarDiv = document.getElementById('hotbar');
const inventoryGrid = document.getElementById('inventory-grid');
const inventoryHotbarGrid = document.getElementById('inventory-hotbar-grid');
const cursorItemEl = document.getElementById('cursor-item');

let isInventoryOpen = false;

function renderUI() {
    // Render Hotbar
    hotbarDiv.innerHTML = '';
    for (let i = 0; i < 9; i++) {
        const slotEl = document.createElement('div');
        slotEl.className = 'slot' + (i === activeSlotIndex ? ' active' : '');
        slotEl.dataset.id = i;
        createSlotContent(slotEl, inventory[i]);
        hotbarDiv.appendChild(slotEl);
    }
    
    // Render Inventory Screen
    if (isInventoryOpen) {
        inventoryGrid.innerHTML = '';
        // Main inventory slots (9-35)
        for (let i = 9; i < 36; i++) {
            const slotEl = document.createElement('div');
            slotEl.className = 'slot';
            slotEl.onclick = () => handleSlotClick(i);
            createSlotContent(slotEl, inventory[i]);
            inventoryGrid.appendChild(slotEl);
        }
        
        if (inventoryHotbarGrid) {
            inventoryHotbarGrid.innerHTML = '';
            // Hotbar slots (0-8)
            for (let i = 0; i < 9; i++) {
                const slotEl = document.createElement('div');
                slotEl.className = 'slot';
                slotEl.onclick = () => handleSlotClick(i);
                createSlotContent(slotEl, inventory[i]);
                inventoryHotbarGrid.appendChild(slotEl);
            }
        }
    }
}

function createSlotContent(slot, item) {
    if (item) {
        const img = document.createElement('img');
        img.src = itemIcons[item.type];
        slot.appendChild(img);
        
        const count = document.createElement('div');
        count.className = 'item-count';
        count.innerText = item.count;
        slot.appendChild(count);
    }
}

function handleSlotClick(index) {
    const item = inventory[index];
    
    if (!cursorItem) {
        if (item) {
            cursorItem = item;
            inventory[index] = null;
        }
    } else {
        if (!item) {
            inventory[index] = cursorItem;
            cursorItem = null;
        } else {
            if (item.type === cursorItem.type) {
                const space = 64 - item.count;
                const toAdd = Math.min(space, cursorItem.count);
                item.count += toAdd;
                cursorItem.count -= toAdd;
                if (cursorItem.count <= 0) cursorItem = null;
            } else {
                const temp = item;
                inventory[index] = cursorItem;
                cursorItem = temp; 
            }
        }
    }
    renderUI();
    updateCursorVisual();
}

function updateCursorVisual() {
    if (cursorItem && cursorItemEl) {
        cursorItemEl.style.display = 'block';
        const img = cursorItemEl.querySelector('img');
        if(img) img.src = itemIcons[cursorItem.type];
        const count = cursorItemEl.querySelector('.item-count');
        if(count) count.innerText = cursorItem.count;
    } else if (cursorItemEl) {
        cursorItemEl.style.display = 'none';
    }
}

document.addEventListener('mousemove', (e) => {
    if (cursorItem && cursorItemEl) {
        cursorItemEl.style.left = (e.pageX - 20) + 'px';
        cursorItemEl.style.top = (e.pageY - 20) + 'px';
    }
});

function toggleInventory() {
    isInventoryOpen = !isInventoryOpen;
    if (isInventoryOpen) {
        controls.unlock();
        inventoryScreen.style.display = 'flex';
        renderUI();
        updateCursorVisual();
    } else {
        controls.lock();
        inventoryScreen.style.display = 'none';
        menuOverlay.style.display = 'none';
        
        if (cursorItem) {
             addToInventory(cursorItem.type, cursorItem.count);
             cursorItem = null;
             updateCursorVisual();
        }
    }
}

resumeBtn.addEventListener('click', () => { 
    if(!isInventoryOpen) controls.lock(); 
});
volumeSlider.addEventListener('input', (e) => {
    soundVolume = e.target.value;
    blockBreakSound.volume = soundVolume;
});

controls.addEventListener('lock', () => {
    menuOverlay.style.display = 'none';
    inventoryScreen.style.display = 'none';
    isInventoryOpen = false;
    if(cursorItemEl) cursorItemEl.style.display = 'none';
});
controls.addEventListener('unlock', () => {
    if (!isInventoryOpen) menuOverlay.style.display = 'flex';
});

document.addEventListener('keydown', (e) => {
    if (e.code === 'KeyE') {
        toggleInventory();
        return;
    }
    
    if (isInventoryOpen) return; // Block input when inventory open

    switch (e.code) {
        case 'KeyW': keys.w = true; break;
        case 'KeyA': keys.a = true; break;
        case 'KeyS': keys.s = true; break;
        case 'KeyD': keys.d = true; break;
        case 'Space': keys.space = true; break;
        case 'ControlLeft': isSprinting = !isSprinting; break;
        case 'Digit1': activeSlotIndex = 0; break;
        case 'Digit2': activeSlotIndex = 1; break;
        case 'Digit3': activeSlotIndex = 2; break;
        case 'Digit4': activeSlotIndex = 3; break;
        case 'Digit5': activeSlotIndex = 4; break;
        case 'Digit6': activeSlotIndex = 5; break;
        case 'Digit7': activeSlotIndex = 6; break;
        case 'Digit8': activeSlotIndex = 7; break;
        case 'Digit9': activeSlotIndex = 8; break;
    }
    if(e.code >= 'Digit1' && e.code <= 'Digit9') renderUI();
});
document.addEventListener('keyup', (e) => {
    switch (e.code) {
        case 'KeyW': keys.w = false; break;
        case 'KeyA': keys.a = false; break;
        case 'KeyS': keys.s = false; break;
        case 'KeyD': keys.d = false; break;
        case 'Space': keys.space = false; break;
    }
});
window.addEventListener('wheel', (e) => {
    if (isInventoryOpen) return;
    if (e.deltaY > 0) activeSlotIndex = (activeSlotIndex + 1) % 9;
    else activeSlotIndex = (activeSlotIndex - 1 + 9) % 9;
    renderUI();
});

document.addEventListener('mousedown', (e) => {
    if (!controls.isLocked || isInventoryOpen) return;
    
    if (e.button === 0 && highlightPos) { 
        world.setBlock(highlightPos.x, highlightPos.y, highlightPos.z, 0);
        blockBreakSound.currentTime = 0;
        blockBreakSound.play();
        swingHand();
    }
    if (e.button === 0 && !highlightPos) {
        swingHand();
    }
    if (e.button === 2 && buildPos) { 
        swingHand();
        const item = inventory[activeSlotIndex];
        if (!item) return;

        const pBoxMin = new THREE.Vector3(player.pos.x - 0.3, player.pos.y, player.pos.z - 0.3);
        const pBoxMax = new THREE.Vector3(player.pos.x + 0.3, player.pos.y + 1.8, player.pos.z + 0.3);
        const bBoxMaxPos = new THREE.Vector3(buildPos.x+1, buildPos.y+1, buildPos.z+1);
        
        const intersect = (pBoxMin.x < bBoxMaxPos.x && pBoxMax.x > buildPos.x) &&
                          (pBoxMin.y < bBoxMaxPos.y && pBoxMax.y > buildPos.y) &&
                          (pBoxMin.z < bBoxMaxPos.z && pBoxMax.z > buildPos.z);
                          
        if (!intersect) {
            world.setBlock(buildPos.x, buildPos.y, buildPos.z, item.type);
            item.count--;
            if (item.count <= 0) {
                inventory[activeSlotIndex] = null;
            }
            renderUI();
        }
    }
});

let prevTime = performance.now();

function animate() {
    requestAnimationFrame(animate);

    const time = performance.now();
    const dt = Math.min((time - prevTime) / 1000, 0.1); 
    prevTime = time;

    updatePhysics(dt);
    updateDrops(dt);
    updateHand(dt);
    updateClouds(dt);
    world.update(player.pos.x, player.pos.z);
    updateRaycaster();

    // Render main scene
    renderer.render(scene, camera);
    
    // Render hand on top (no depth clear, just color)
    renderer.autoClear = false;
    renderer.clearDepth();
    renderer.render(handScene, handCamera);
    renderer.autoClear = true;
}

world.update(0,0);
renderUI();
animate();

window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    handCamera.aspect = window.innerWidth / window.innerHeight;
    handCamera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
});

index.html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Minecraft 2.0 - WebGL</title>
    <!-- Import Map для Three.js -->
    <script type="importmap">
        {
            "imports": {
                "three": "https://unpkg.com/three@0.160.0/build/three.module.js",
                "three/addons/": "https://unpkg.com/three@0.160.0/examples/jsm/"
            }
        }
    </script>
    <!-- Библиотека шума -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/simplex-noise/2.4.0/simplex-noise.min.js"></script>
<style>
body {
    margin: 0;
    overflow: hidden;
    font-family: 'Courier New', Courier, monospace;
    user-select: none;
}

#crosshair {
    position: absolute;
    top: 50%;
    left: 50%;
    width: 20px;
    height: 20px;
    margin-top: -10px;
    margin-left: -10px;
    text-align: center;
    line-height: 20px;
    color: white;
    font-size: 20px;
    pointer-events: none;
    z-index: 10;
    text-shadow: 1px 1px 0 #000;
}

#menu-overlay {
    position: absolute;
    width: 100%;
    height: 100%;
    background-color: rgba(0,0,0,0.7);
    display: flex;
    justify-content: center;
    align-items: center;
    z-index: 20;
    backdrop-filter: blur(5px);
}

#menu-box {
    background: rgba(0, 0, 0, 0.8);
    padding: 40px;
    border: 4px solid #fff;
    border-radius: 10px;
    text-align: center;
    color: white;
    width: 400px;
    box-shadow: 0 0 20px rgba(0,0,0,0.5);
}

#menu-box h1 {
    margin-top: 0;
    text-shadow: 2px 2px 0 #555;
    font-size: 32px;
}

#resume-btn {
    padding: 15px 30px;
    font-size: 20px;
    font-family: inherit;
    cursor: pointer;
    background: #4caf50;
    color: white;
    border: none;
    border-radius: 5px;
    margin-bottom: 20px;
    width: 100%;
    transition: background 0.2s;
}

#resume-btn:hover {
    background: #45a049;
}

#single-btn:hover { background:#1e88e5; }

.settings-row {
    margin: 20px 0;
    display: flex;
    flex-direction: column;
    gap: 10px;
    align-items: center;
}

.controls-info {
    margin-top: 20px;
    font-size: 14px;
    color: #ccc;
    border-top: 1px solid #555;
    padding-top: 10px;
}

#hotbar {
    position: absolute;
    bottom: 10px;
    left: 50%;
    transform: translateX(-50%);
    display: flex;
    background: rgba(0, 0, 0, 0.5);
    padding: 5px;
    border-radius: 5px;
    gap: 5px;
    z-index: 10;
}

.slot {
    width: 50px;
    height: 50px;
    border: 2px solid #555;
    background: rgba(0,0,0,0.3);
    display: flex;
    justify-content: center;
    align-items: center;
    position: relative;
}

.slot img {
    width: 90%;
    height: 90%;
    image-rendering: pixelated; 
}

.slot.active {
    border: 3px solid white;
    background: rgba(255,255,255,0.2);
    transform: scale(1.1);
}

.item-count {
    position: absolute;
    bottom: 2px;
    right: 2px;
    font-size: 14px;
    color: white;
    text-shadow: 1px 1px 0 #000;
}

#inventory-screen {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: rgba(0,0,0,0.7);
    display: flex;
    justify-content: center;
    align-items: center;
    z-index: 15;
}

#inventory-container {
    background: #c6c6c6;
    padding: 20px;
    border-radius: 5px;
    border: 4px solid #555;
    color: #333;
}

#inv-skinbar{
  width: 100%;
  display:flex;
  justify-content:center;
  margin: 10px 0 14px;
}
#inv-skin-canvas{
  background:#000;
  border: 4px solid #111;
  border-radius: 6px;
  image-rendering: pixelated;
}


#inventory-grid {
    display: grid;
    grid-template-columns: repeat(9, 1fr);
    gap: 5px;
    margin-top: 10px;
}

#inventory-hotbar-grid {
    display: grid;
    grid-template-columns: repeat(9, 1fr);
    gap: 5px;
}

#inventory-grid .slot, #inventory-hotbar-grid .slot {
    background: #8b8b8b;
    border: 2px solid #fff;
    border-right-color: #373737;
    border-bottom-color: #373737;
}

#inventory-grid .slot:hover, #inventory-hotbar-grid .slot:hover {
    background: #a0a0a0;
}

/* Pixel / Minecraft-like font */
@import url('https://fonts.googleapis.com/css2?family=Press+Start+2P&display=swap');

body {
  font-family: 'Press Start 2P', monospace !important;
}

/* Multiplayer / Chat UI */
#net-status {
  margin-top: 10px;
  font-size: 12px;
  opacity: 0.85;
}

.settings-row input, .settings-row label, #menu-box p, #menu-box h1, #menu-box button, #menu-box input {
  font-family: inherit;
}

#menu-box input[type="text"]{
  width: 100%;
  box-sizing: border-box;
  margin: 8px 0;
  padding: 10px 12px;
  border: 2px solid #fff;
  background: rgba(0,0,0,0.55);
  color: #fff;
  border-radius: 6px;
  font-size: 12px;
}

#chat-box {
  position: absolute;
  left: 12px;
  bottom: 64px;
  width: min(420px, calc(100vw - 24px));
  max-height: 34vh;
  overflow: hidden;
  z-index: 50;
  display: block;
  pointer-events: none;
}

#chat-messages {
  background: rgba(0,0,0,0.55);
  border: 2px solid rgba(255,255,255,0.65);
  padding: 10px;
  border-radius: 8px;
  max-height: 26vh;
  overflow-y: auto;
  color: #fff;
  font-size: 12px;
  line-height: 1.35;
  text-shadow: 2px 2px 0 #000;
}

#chat-input-line {
  margin-top: 6px;
  background: rgba(0,0,0,0.55);
  border: 2px solid rgba(255,255,255,0.65);
  padding: 10px;
  border-radius: 8px;
  color: #fff;
  font-size: 12px;
  text-shadow: 2px 2px 0 #000;
  display: none;
}

#chat-input-line .prompt { opacity: 0.8; }
#chat-draft { white-space: pre-wrap; }


/* Pixel font everywhere */
html, body, button, input, textarea, #chat-box, #chat-messages, #menu-box, #hotbar, #inventory-screen {
  font-family: 'Press Start 2P', monospace !important;
}

/* Mobile controls */
#mobile-controls {
  position: fixed;
  left: 0;
  right: 0;
  bottom: 0;
  height: 40vh;
  pointer-events: none;
  z-index: 9999;
}
#mobile-controls .m-pad {
  position: absolute;
  bottom: 16px;
  display: flex;
  flex-direction: column;
  gap: 10px;
  pointer-events: none;
}
#mobile-controls .m-left { left: 16px; align-items: center; }
#mobile-controls .m-right { right: 16px; align-items: flex-end; }

#mobile-controls .m-row {
  display: flex;
  gap: 10px;
  pointer-events: none;
}
#mobile-controls .m-btn {
  pointer-events: auto;
  width: 70px;
  height: 70px;
  border: 2px solid #111;
  border-radius: 14px;
  background: rgba(255,255,255,0.20);
  color: #fff;
  text-shadow: 2px 2px 0 #000;
  backdrop-filter: blur(2px);
  font-size: 18px;
}
#mobile-controls .m-btn:active {
  transform: translateY(1px);
  background: rgba(255,255,255,0.32);
}
#mobile-controls .m-jump {
  width: 110px;
  height: 70px;
  font-size: 14px;
}
@media (hover: hover) and (pointer: fine) {
  #mobile-controls { display: none !important; }
}


#chat-messages .sys {
  color: #ffe28a;
  opacity: 0.95;
}


/* Refreshed main menu */
#menu-overlay{
  background-image:
    linear-gradient(rgba(0,0,0,0.34), rgba(0,0,0,0.58)),
    url('./Assets/menu_bg.webp');
  background-size: cover;
  background-position: center;
  background-repeat: no-repeat;
}
#menu-box{
  background: rgba(0,0,0,0.18);
  border: none;
  border-radius: 0;
  box-shadow: none;
  padding: 18px 32px 26px;
  width: min(92vw, 560px);
}
#menu-box h1{
  font-size: clamp(34px, 7vw, 58px);
  line-height: 1;
  margin: 0 0 22px;
  letter-spacing: 2px;
  color: #fff;
  text-shadow:
    0 3px 0 #1a1a1a,
    0 6px 0 #000,
    3px 3px 0 #000,
    -3px 3px 0 #000;
}
#resume-btn, #single-btn{
  display:block;
  width:100%;
  margin:0 0 12px;
  padding:14px 16px;
  font-size:16px;
  border:3px solid #111;
  border-top-color:#6f6f6f;
  border-left-color:#6f6f6f;
  border-right-color:#2b2b2b;
  border-bottom-color:#2b2b2b;
  border-radius:0;
  background: linear-gradient(#8a8a8a, #5e5e5e);
  color:#fff;
  cursor:pointer;
  box-shadow: inset 0 1px 0 rgba(255,255,255,0.18);
}
#resume-btn:hover, #single-btn:hover{
  background: linear-gradient(#9b9b9b, #6c6c6c);
}
#player-name{
  text-align:center;
  margin-top: 6px !important;
  margin-bottom: 4px !important;
}
#name-status{
  min-height: 16px;
  margin: 4px 0 10px;
  font-size: 10px;
  color: #ff9b9b;
  text-shadow: 1px 1px 0 #000;
}
#net-status{
  margin: 0 0 12px;
  font-size: 11px;
  color: #dfdfdf;
}
.settings-row{
  margin: 14px 0;
}
.controls-info{
  font-size: 11px;
  line-height: 1.6;
  background: rgba(0,0,0,0.35);
  border: 2px solid rgba(255,255,255,0.12);
}
</style>
</head>
<body>
    <!-- Прицел -->
    <div id="crosshair">+</div>

    <!-- Меню -->
    <div id="menu-overlay">
        <div id="menu-box">
            <h1>MINECRAFT WEB</h1>
            <button id="single-btn">ОДИНОЧНАЯ ИГРА</button>
            <button id="resume-btn">СЕТЕВАЯ ИГРА</button>
            <div style="height: 14px;"></div>
            <input id="player-name" type="text" value="Player" maxlength="16" placeholder="Никнейм (до 16)">
            <div id="name-status"></div>
            <input id="room-id" type="hidden" value="global">
            <div id="net-status">OFFLINE</div>

            <div class="settings-row">
                <label>Громкость звуков:</label>
                <input type="range" id="volume-slider" min="0" max="1" step="0.1" value="0.5">
            </div>
            <div class="controls-info">
                <p>WASD - Ходить | SPACE - Прыжок</p>
                <p>ЛКМ - Ломать | ПКМ - Строить</p>
                <p>1-9 или Колесо - Выбор блока</p>
                <p>L-Ctrl - Бег</p>
                <p>E - Инвентарь</p>
                <p>ЛКМ — взять | ПКМ — разложить по 1 | Q — выбросить</p>
            </div>
        </div>
    </div>

    <!-- Хотбар -->
    <div id="hotbar">
        <!-- Генерируется скриптом -->
    </div>

    <!-- Инвентарь -->
    <div id="inventory-screen" style="display: none;">
        <div id="inventory-container">
            <h2>Inventory</h2>
            <div id="inv-skinbar">
              <canvas id="inv-skin-canvas" width="160" height="200"></canvas>
            </div>
            <div id="inventory-grid">
                <!-- Генерируется скриптом -->
            </div>
            <div style="height: 20px;"></div>
            <div id="inventory-hotbar-grid">
                 <!-- Генерируется скриптом (Хотбар) -->
            </div>
        </div>
    </div>
    
    <!-- Элемент для перетаскивания -->
    <div id="cursor-item" style="display: none; position: absolute; pointer-events: none; z-index: 1000; width: 40px; height: 40px;">
        <img src="" style="width: 100%; height: 100%; image-rendering: pixelated;">
        <div class="item-count" style="position: absolute; bottom: 0; right: 0; color: white; text-shadow: 1px 1px 0 #000;"></div>
    </div>

    
    <!-- Chat (T открыть, Enter отправить) -->
    <div id="chat-box">
        <div id="chat-messages"></div>
        <div id="chat-input-line"><span class="prompt">&gt; </span><span id="chat-draft"></span></div>
    </div>


    <!-- Mobile controls -->
    <div id="mobile-controls" style="display:none;">
        <div class="m-pad m-left">
            <button class="m-btn" data-key="w">▲</button>
            <div class="m-row">
                <button class="m-btn" data-key="a">◀</button>
                <button class="m-btn" data-key="s">▼</button>
                <button class="m-btn" data-key="d">▶</button>
            </div>
        </div>
        <div class="m-pad m-right">
            <button class="m-btn m-jump" data-key="space">JUMP</button>
            <button class="m-btn m-flydown" data-key="shift" style="display:none;">DOWN</button>
        </div>
    </div>

    <!-- Основной скрипт -->
    <script type="module">
import * as THREE from 'three';
import { PointerLockControls } from 'three/addons/controls/PointerLockControls.js';


// --- Firebase (CDN ESM) ---
import { initializeApp } from 'https://www.gstatic.com/firebasejs/12.9.0/firebase-app.js';
import {
  getDatabase, ref as dbRef, set as dbSet, update as dbUpdate, push as dbPush,
  onValue, onChildAdded, query as dbQuery, limitToLast,
  serverTimestamp, onDisconnect, remove as dbRemove, runTransaction
} from 'https://www.gstatic.com/firebasejs/12.9.0/firebase-database.js';

// --- НАСТРОЙКИ ---
const CHUNK_SIZE = 16;
const RENDER_DISTANCE = 3; 
const TEXTURE_PATH = './Assets/';
const GRAVITY = 32.0;
const JUMP_FORCE = 10.0;
const SPEED = 5.0;
const PLAYER_HEIGHT = 1.7;
const PLAYER_RADIUS = 0.3;
const PICKUP_RANGE = 2.0;

// --- СЦЕНА ---
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x87CEEB);
scene.fog = new THREE.Fog(0x87CEEB, 20, (RENDER_DISTANCE * CHUNK_SIZE) - 5);

const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
scene.add(camera); // Add camera to scene so children (hand) are visible
// Дополнительная камера для рендера (режимы: 1-е лицо / 3-е лицо спереди / 3-е лицо сзади)
const viewCamera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
viewCamera.position.copy(camera.position);
viewCamera.quaternion.copy(camera.quaternion);
viewCamera.updateProjectionMatrix();

// 0 = первое лицо, 1 = камера спереди (вид на игрока), 2 = камера сзади
let cameraMode = 0;

// Локальный аватар (чтобы в 3-м лице было видно себя)
let localAvatar = null;
let localAvatarTag = null;

// --- CREATIVE / FLIGHT (включается командой /gamemodeezes) ---
let creativeUnlocked = false;
let flyMode = false;

function sysChatLine(msg) {
  // локальная строка в чате (не уходит в Firebase)
  if (!netUI.chatMessages) return;
  const line = document.createElement('div');
  line.className = 'msg system';
  line.textContent = msg;
  netUI.chatMessages.appendChild(line);
  netUI.chatMessages.scrollTop = netUI.chatMessages.scrollHeight;
}

function isSolidBlock(t) {
  return t !== 0 && t !== 6; // воздух и листья считаем "несолидными"
}

// Подрезаем камеру, чтобы она не залезала в блоки
function clampCameraToWorld(eye, desired) {
  const dir = desired.clone().sub(eye);
  const dist = dir.length();
  if (dist < 0.001) return desired;
  dir.normalize();

  const steps = Math.ceil(dist / 0.25);
  let lastSafe = eye.clone();
  for (let i = 1; i <= steps; i++) {
    const p = eye.clone().addScaledVector(dir, dist * (i / steps));
    const bt = world.getBlock(p.x, p.y, p.z);
    if (isSolidBlock(bt)) {
      return lastSafe;
    }
    lastSafe = p;
  }
  return desired;
}

function updateViewCamera() {
  // "глаза" игрока (первичная камера) остаётся всегда на месте для физики/луча/управления
  const eye = camera.position.clone();

  // расстояние 3-го лица
  const dist = 4.0;

  // forward для игрока
  const forward = new THREE.Vector3(0, 0, -1).applyQuaternion(camera.quaternion).normalize();
  const up = new THREE.Vector3(0, 1, 0);

  if (cameraMode === 0) {
    // First person: рендерим основной камерой
    viewCamera.position.copy(camera.position);
    viewCamera.quaternion.copy(camera.quaternion);

    if (localAvatar) localAvatar.visible = false;
    document.getElementById('crosshair').style.display = 'block';
  } else if (cameraMode === 2) {
    // Third person (behind): камера позади, смотрит как игрок
    const desired = eye.clone()
      .addScaledVector(up, 0.15)
      .addScaledVector(forward, -dist);
    const pos = clampCameraToWorld(eye, desired);

    viewCamera.position.copy(pos);
    viewCamera.quaternion.copy(camera.quaternion);

    if (localAvatar) localAvatar.visible = true;
    document.getElementById('crosshair').style.display = 'block';
  } else {
    // cameraMode === 1 (front): камера спереди, смотрит НА игрока
    const desired = eye.clone()
      .addScaledVector(up, 0.15)
      .addScaledVector(forward, dist);
    const pos = clampCameraToWorld(eye, desired);

    viewCamera.position.copy(pos);
    viewCamera.lookAt(eye);

    if (localAvatar) localAvatar.visible = true;
    document.getElementById('crosshair').style.display = 'block';
  }

  // синхронизируем проекцию
  viewCamera.fov = camera.fov;
  viewCamera.near = camera.near;
  viewCamera.far = camera.far;
  viewCamera.updateProjectionMatrix();

  // Локальный аватар позиция/поворот (если создан)
  if (localAvatar) {
    localAvatar.position.set(player.pos.x, player.pos.y, player.pos.z);

    // Поворот по yaw игрока
    const eul = new THREE.Euler().setFromQuaternion(camera.quaternion, 'YXZ');
    localAvatar.rotation.y = eul.y;
  }
}

const renderer = new THREE.WebGLRenderer({ antialias: false });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(window.devicePixelRatio);
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;
document.body.appendChild(renderer.domElement);

// Клик по игре тоже захватывает мышь (если меню уже закрыто)
renderer.domElement.addEventListener('click', () => {
  if (menuOverlay.style.display === 'none' && !isInventoryOpen && !controls.isLocked) {
    try { controls.lock(); } catch(e) {}
  }
});
// --- РЕСУРСЫ ---
const loader = new THREE.TextureLoader();
const loadTex = (url) => {
    const tex = loader.load(TEXTURE_PATH + url);
    tex.magFilter = THREE.NearestFilter;
    tex.minFilter = THREE.NearestFilter;
    tex.colorSpace = THREE.SRGBColorSpace;
    return tex;
};

const textures = {
    dirt: loadTex('Dirt.png'),
    grass: loadTex('grass.png'), 
    stone: loadTex('stone.png'),
    wood: loadTex('tree.jpg'),
    leaves: loadTex('foliage.jpg'),
    sand: loadTex('sand.png'),
    planks: loadTex('planks.png'),
    oreCoal: loadTex('ore_coal.png'),
    oreIron: loadTex('ore_iron.png'),
    oreGold: loadTex('ore_gold.png'),
    oreDiamond: loadTex('ore_diamond.png'),
    craftingTable: loadTex('crafting_table.png'),
    chest: loadTex('chest.png')
};

const materials = [];
materials[1] = new THREE.MeshLambertMaterial({ map: textures.grass });
materials[2] = new THREE.MeshLambertMaterial({ map: textures.dirt });
materials[3] = new THREE.MeshLambertMaterial({ map: textures.stone });
materials[4] = new THREE.MeshLambertMaterial({ map: textures.wood });
materials[5] = new THREE.MeshLambertMaterial({ map: textures.leaves, transparent: true, alphaTest: 0.5 });
materials[6] = new THREE.MeshBasicMaterial({ color: 0xffffff, transparent: true, opacity: 0.8 }); // Clouds
materials[10] = new THREE.MeshLambertMaterial({ map: textures.sand });
materials[11] = new THREE.MeshLambertMaterial({ map: textures.planks });
materials[12] = new THREE.MeshLambertMaterial({ map: textures.oreCoal });
materials[13] = new THREE.MeshLambertMaterial({ map: textures.oreIron });
materials[14] = new THREE.MeshLambertMaterial({ map: textures.oreGold });
materials[15] = new THREE.MeshLambertMaterial({ map: textures.oreDiamond });
materials[17] = new THREE.MeshLambertMaterial({ map: textures.craftingTable });
materials[18] = new THREE.MeshLambertMaterial({ map: textures.chest });

const itemIcons = {
    1: './Assets/grass.png',
    2: './Assets/Dirt.png',
    3: './Assets/stone.png',
    4: './Assets/tree.jpg',
    5: './Assets/foliage.jpg',
    10: './Assets/sand.png',
    11: './Assets/planks.png',
    12: './Assets/ore_coal.png',
    13: './Assets/ore_iron.png',
    14: './Assets/ore_gold.png',
    15: './Assets/ore_diamond.png',
    17: './Assets/crafting_table.png',
    18: './Assets/chest.png'
};

/* --- ЗВУКИ (ogg, как ты скидывал) --- */
let soundVolume = 0.5;
const SOUND_DIR = TEXTURE_PATH + 'sounds/';

function createSound(file) {
  // Пытаемся сначала ./Assets/sounds/, если 404 — падаем на ./Assets/
  const a = new Audio(SOUND_DIR + file);
  a.preload = 'auto';
  a.volume = soundVolume;
  a.addEventListener('error', () => {
    // fallback
    if (a.src.includes('/sounds/')) {
      a.src = TEXTURE_PATH + file;
      a.load();
    }
  }, { once: true });
  return a;
}

const SND = {
  dig_grass: createSound('dig_grass.ogg'),
  dig_stone: createSound('dig_stone.ogg'),
  step_grass: createSound('step_grass.ogg'),
  step_stone: createSound('step_stone.ogg'),
  pickup: createSound('pickup.ogg'),
  craft: createSound('craft.ogg'),
};

function setSoundVolume(v) {
  soundVolume = v;
  for (const a of Object.values(SND)) a.volume = soundVolume;
}

function playSound(key) {
  const a = SND[key];
  if (!a) return;
  try {
    a.currentTime = 0;
    a.play();
  } catch (_) {}
}

// какие блоки считаем "каменными" для звуков
function isStoneLikeBlock(t) {
  return t === 3 || t === 12 || t === 13 || t === 14 || t === 15;
}

function playDigForBlock(t) {
  playSound(isStoneLikeBlock(t) ? 'dig_stone' : 'dig_grass');
}
// --- ИНВЕНТАРЬ ---
// 9 хотбар + 27 инвентарь = 36
const inventory = new Array(36).fill(null); 
// null или { type: int, count: int }
let cursorItem = null;
let isSprinting = false;

const STARTER_ITEMS = [];
for (const starter of STARTER_ITEMS) {
  inventory[starter.slot] = { type: starter.type, count: starter.count };
}

// --- HAND (Minecraft style) ---
const handScene = new THREE.Scene();
const handCamera = new THREE.PerspectiveCamera(70, window.innerWidth / window.innerHeight, 0.01, 10);
handCamera.position.set(0, 0, 0);

// Add light to hand scene
const handAmbient = new THREE.AmbientLight(0xffffff, 0.8);
handScene.add(handAmbient);
const handDirLight = new THREE.DirectionalLight(0xffffff, 0.5);
handDirLight.position.set(1, 1, 1);
handScene.add(handDirLight);

// Arm (Steve skin colored)
const armGeo = new THREE.BoxGeometry(0.15, 0.5, 0.15);
const armMat = new THREE.MeshLambertMaterial({ color: 0xc9a065 });
const armMesh = new THREE.Mesh(armGeo, armMat);

// Block in hand
const blockInHandGeo = new THREE.BoxGeometry(0.2, 0.2, 0.2);
const blockInHandMesh = new THREE.Mesh(blockInHandGeo, materials[2]);
blockInHandMesh.visible = false;
blockInHandMesh.position.set(0, 0.15, -0.05);

// Hand pivot for animation
const handPivot = new THREE.Group();
handPivot.add(armMesh);
handPivot.add(blockInHandMesh);

// Position hand in bottom right of screen
handPivot.position.set(0.4, -0.35, -0.5);
handPivot.rotation.set(-0.1, -0.3, 0);
armMesh.position.set(0, -0.1, 0);

handScene.add(handPivot);

let handSwingTime = 0;
const HAND_SWING_DURATION = 0.25;
let handBaseRotX = -0.1;
let handBaseRotZ = 0;

function updateHand(dt) {
    // Swing animation
    if (handSwingTime > 0) {
        handSwingTime -= dt;
        const t = 1 - (handSwingTime / HAND_SWING_DURATION);
        // Swing down and to the left
        const swing = Math.sin(t * Math.PI);
        handPivot.rotation.x = handBaseRotX + swing * 0.6;
        handPivot.rotation.z = handBaseRotZ - swing * 0.4;
    } else {
        // Idle bobbing
        const bobTime = performance.now() * 0.003;
        handPivot.rotation.x = handBaseRotX + Math.sin(bobTime) * 0.02;
        handPivot.rotation.z = handBaseRotZ + Math.cos(bobTime * 0.7) * 0.01;
    }
    
    // Update block in hand based on selected slot
    const item = inventory[activeSlotIndex];
    if (item && materials[item.type]) {
        armMesh.visible = false;
        blockInHandMesh.visible = true;
        blockInHandMesh.material = materials[item.type];
    } else {
        armMesh.visible = true;
        blockInHandMesh.visible = false;
    }
}

function swingHand() {
    if (handSwingTime <= 0) {
        handSwingTime = HAND_SWING_DURATION;
    }
}

// --- CLOUDS (Smooth moving plane) ---
const CLOUD_HEIGHT = 45;
const CLOUD_SPEED = 2; // blocks per second
let cloudOffset = 0;

// Create cloud texture procedurally
const cloudCanvas = document.createElement('canvas');
cloudCanvas.width = 256;
cloudCanvas.height = 256;
const cloudCtx = cloudCanvas.getContext('2d');

// Draw Minecraft-style cloud pattern
cloudCtx.fillStyle = 'rgba(0,0,0,0)';
cloudCtx.fillRect(0, 0, 256, 256);
cloudCtx.fillStyle = 'rgba(255,255,255,0.9)';

// Create blocky cloud shapes
const cloudPattern = [
    [0,0,1,1,1,1,1,0,0,0,0,0,1,1,1,0],
    [0,1,1,1,1,1,1,1,0,0,0,1,1,1,1,1],
    [1,1,1,1,1,1,1,1,1,0,1,1,1,1,1,1],
    [1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1],
    [0,1,1,1,1,1,1,1,1,1,1,1,1,1,1,0],
    [0,0,1,1,1,1,1,1,1,1,1,1,1,0,0,0],
    [0,0,0,0,1,1,1,1,1,1,0,0,0,0,0,0],
    [0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0],
    [0,0,0,0,0,1,1,1,1,0,0,0,0,0,0,0],
    [0,0,0,0,1,1,1,1,1,1,0,0,1,1,0,0],
    [0,0,0,1,1,1,1,1,1,1,1,1,1,1,1,0],
    [0,0,1,1,1,1,1,1,1,1,1,1,1,1,1,1],
    [0,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1],
    [0,1,1,1,1,1,1,1,1,1,1,1,1,1,0,0],
    [0,0,1,1,1,1,1,0,0,1,1,1,1,0,0,0],
    [0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
];

for (let y = 0; y < 16; y++) {
    for (let x = 0; x < 16; x++) {
        if (cloudPattern[y][x]) {
            cloudCtx.fillRect(x * 16, y * 16, 16, 16);
        }
    }
}

const cloudTexture = new THREE.CanvasTexture(cloudCanvas);
cloudTexture.wrapS = THREE.RepeatWrapping;
cloudTexture.wrapT = THREE.RepeatWrapping;
cloudTexture.magFilter = THREE.NearestFilter;
cloudTexture.minFilter = THREE.NearestFilter;
cloudTexture.repeat.set(8, 8);

const cloudGeo = new THREE.PlaneGeometry(500, 500);
const cloudMat = new THREE.MeshBasicMaterial({
    map: cloudTexture,
    transparent: true,
    opacity: 0.9,
    side: THREE.DoubleSide,
    depthWrite: false
});
const cloudMesh = new THREE.Mesh(cloudGeo, cloudMat);
cloudMesh.rotation.x = -Math.PI / 2;
cloudMesh.position.y = CLOUD_HEIGHT;
scene.add(cloudMesh);

function updateClouds(dt) {
    cloudOffset += CLOUD_SPEED * dt * 0.01;
    cloudTexture.offset.x = cloudOffset;
    // Follow player horizontally
    cloudMesh.position.x = player.pos.x;
    cloudMesh.position.z = player.pos.z;
}

// --- СВЕТ ---
const ambientLight = new THREE.AmbientLight(0xffffff, 0.7);
scene.add(ambientLight);

const dirLight = new THREE.DirectionalLight(0xffffff, 0.6);
dirLight.position.set(100, 200, 50);
dirLight.castShadow = true;
dirLight.shadow.mapSize.width = 2048;
dirLight.shadow.mapSize.height = 2048;
scene.add(dirLight);

// --- ГЕНЕРАЦИЯ ---
const DEFAULT_WORLD_SEED = 13371337; // общий детерминированный мир (пока seed из Firebase не пришёл)
let worldSeed = DEFAULT_WORLD_SEED;

// быстрый PRNG (детерминированный)
function mulberry32(a) {
  return function() {
    let t = a += 0x6D2B79F5;
    t = Math.imul(t ^ (t >>> 15), t | 1);
    t ^= t + Math.imul(t ^ (t >>> 7), t | 61);
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296;
  }
}
// детерминированный "рандом" по координатам
function randHash(seed, a, b, c = 0) {
  let x = (seed ^ 0x9E3779B9) >>> 0;
  x = Math.imul(x ^ (a | 0), 0x85EBCA6B) >>> 0;
  x = Math.imul(x ^ (b | 0), 0xC2B2AE35) >>> 0;
  x = Math.imul(x ^ (c | 0), 0x27D4EB2F) >>> 0;
  x ^= x >>> 16;
  return (x >>> 0) / 4294967296;
}
function setWorldSeed(seed) {
  worldSeed = (seed >>> 0) || DEFAULT_WORLD_SEED;
  const rng = mulberry32(worldSeed);
  // SimplexNoise v2 принимает randomFn
  simplex = new SimplexNoise(rng);
}

let simplex = null;
setWorldSeed(DEFAULT_WORLD_SEED);

const chunks = new Map(); 
const savedChunkMods = new Map(); 

function getChunkKey(x, z) { return `${x},${z}`; }

class Chunk {
    constructor(cx, cz) {
        this.cx = cx;
        this.cz = cz;
        this.blocks = new Map(); 
        this.meshGroup = new THREE.Group();
        this.meshGroup.position.set(cx * CHUNK_SIZE, 0, cz * CHUNK_SIZE);
        
        this.generate();
        this.applyMods();
        this.buildMesh();
        
        scene.add(this.meshGroup);
    }

    generate() {
        if (!simplex) return;
        for (let x = 0; x < CHUNK_SIZE; x++) {
            for (let z = 0; z < CHUNK_SIZE; z++) {
                const wx = this.cx * CHUNK_SIZE + x;
                const wz = this.cz * CHUNK_SIZE + z;
                
                const scale = 0.02;
                const noise = simplex.noise2D(wx * scale, wz * scale);
                const h = Math.floor((noise + 1) * 8 + 4); 

                for (let y = 0; y <= h; y++) {
                    let type = 2; // Dirt
                    const belowSurface = h - y;

                    if (y === h) {
                        type = h <= 6 ? 10 : 1; // Low terrain becomes sand
                    } else if (belowSurface <= 2 && h <= 6) {
                        type = 10; // Sand layer near beaches
                    } else if (y < h - 4) {
                        type = 3; // Stone

                        const oreRoll = randHash(worldSeed, wx, wz, y + 2000);
                        if (y < 20 && oreRoll < 0.018) type = 12; // Coal ore
                        if (y < 16 && oreRoll < 0.011) type = 13; // Iron ore
                        if (y < 10 && oreRoll < 0.006) type = 14; // Gold ore
                        if (y < 6 && oreRoll < 0.003) type = 15;  // Diamond ore
                    }

                    this.setBlockLocal(x, y, z, type);
                }

                if (x > 2 && x < 13 && z > 2 && z < 13 && randHash(worldSeed, wx, wz) < 0.01 && h > 4) {
                   this.buildTree(x, h + 1, z);
                }
            }
        }
        
    }

    applyMods() {
        const key = getChunkKey(this.cx, this.cz);
        if (savedChunkMods.has(key)) {
            const mods = savedChunkMods.get(key);
            for (const m of mods) {
                this.setBlockLocal(m.x, m.y, m.z, m.type);
            }
        }
    }

    buildTree(lx, ly, lz) {
        const wx0 = this.cx * CHUNK_SIZE + lx;
        const wz0 = this.cz * CHUNK_SIZE + lz;
        const height = 4 + Math.floor(randHash(worldSeed, wx0, wz0, ly) * 3);
        for (let i = 0; i < height; i++) {
            this.setBlockLocal(lx, ly + i, lz, 4);
        }
        for (let y = ly + height - 3; y <= ly + height; y++) {
            const range = (y >= ly + height - 1) ? 1 : 2; 
            for (let x = lx - range; x <= lx + range; x++) {
                for (let z = lz - range; z <= lz + range; z++) {
                    if (x === lx && z === lz && y < ly + height) continue; 
                    if ((Math.abs(x - lx) === range && Math.abs(z - lz) === range) && (y > ly + height - 2 || randHash(worldSeed, (this.cx*CHUNK_SIZE + x), (this.cz*CHUNK_SIZE + z), y) < 0.3)) continue;
                    this.setBlockLocal(x, y, z, 5);
                }
            }
        }
    }

    setBlockLocal(x, y, z, type) {
        const key = `${x},${y},${z}`;
        if (type === 0) this.blocks.delete(key);
        else this.blocks.set(key, type);
    }

    getBlockLocal(x, y, z) {
        return this.blocks.get(`${x},${y},${z}`) || 0;
    }

    buildMesh() {
        this.meshGroup.clear();
        
        const matrices = {};
        for (let i = 1; i < materials.length; i++) {
            if (materials[i]) matrices[i] = [];
        }

        const dummy = new THREE.Object3D();

        for (const [key, type] of this.blocks) {
            const [x, y, z] = key.split(',').map(Number);
            if (type !== 5 && type !== 6 && this.isOccluded(x, y, z)) continue;
            dummy.position.set(x + 0.5, y + 0.5, z + 0.5);
            dummy.updateMatrix();
            if(matrices[type]) matrices[type].push(dummy.matrix.clone());
        }

        const geom = new THREE.BoxGeometry(1, 1, 1);
        
        for (const type in matrices) {
            const arr = matrices[type];
            if (arr.length === 0) continue;
            const mesh = new THREE.InstancedMesh(geom, materials[type], arr.length);
            for (let i = 0; i < arr.length; i++) {
                mesh.setMatrixAt(i, arr[i]);
            }
            mesh.castShadow = true;
            mesh.receiveShadow = true;
            this.meshGroup.add(mesh);
        }
    }

    isOccluded(x, y, z) {
        const neighbors = [[1,0,0], [-1,0,0], [0,1,0], [0,-1,0], [0,0,1], [0,0,-1]];
        for (let n of neighbors) {
            const t = this.getBlockLocal(x+n[0], y+n[1], z+n[2]);
            if (t === 0 || t === 5 || t === 6) return false;
        }
        return true; 
    }

    dispose() {
        scene.remove(this.meshGroup);
        this.meshGroup.traverse(o => {
            if(o.geometry) o.geometry.dispose();
        });
    }
}

// --- DROPS ---
const drops = [];
const dropGeo = new THREE.BoxGeometry(0.3, 0.3, 0.3);

function spawnDrop(x, y, z, type, count = 1, toss = true) {
    if(!materials[type]) return; // don't spawn drops for invalid blocks
    const mesh = new THREE.Mesh(dropGeo, materials[type]);
    mesh.position.set(x + 0.5, y + 0.5, z + 0.5);
    mesh.castShadow = true;
    scene.add(mesh);
    drops.push({
        mesh,
        type,
        count: Math.max(1, count | 0),
        vel: toss ? new THREE.Vector3((Math.random()-0.5)*2, 5, (Math.random()-0.5)*2) : new THREE.Vector3(),
        creationTime: performance.now()
    });
}

function updateDrops(dt) {
    for (let i = drops.length - 1; i >= 0; i--) {
        const d = drops[i];
        d.vel.y -= GRAVITY * dt;
        d.mesh.position.addScaledVector(d.vel, dt);

        // Simple ground check
        if (world.getBlock(d.mesh.position.x, d.mesh.position.y - 0.15, d.mesh.position.z) !== 0) {
            d.vel.y = 0;
            d.vel.x *= 0.9;
            d.vel.z *= 0.9;
            d.mesh.position.y = Math.ceil(d.mesh.position.y - 0.5) + 0.15;
        }

        d.mesh.rotation.y += dt;

        // Pickup
        const dist = d.mesh.position.distanceTo(player.pos);
        if (dist < PICKUP_RANGE && (performance.now() - d.creationTime > 500)) {
            if (addToInventory(d.type, d.count)) {
                scene.remove(d.mesh);
                drops.splice(i, 1);
                playSound('pickup');
            }
        }
    }
}

function addToInventory(type, amount = 1) {
    // Try calculate stack
    for(let i=0; i<inventory.length; i++) {
        if (inventory[i] && inventory[i].type === type && inventory[i].count < 64) {
            const space = 64 - inventory[i].count;
            const toAdd = Math.min(space, amount);
            inventory[i].count += toAdd;
            amount -= toAdd;
            if(amount <= 0) {
                renderUI();
                return true;
            }
        }
    }
    // Try empty slot
    while(amount > 0) {
        let emptyIdx = -1;
        for(let i=0; i<inventory.length; i++) {
            if (!inventory[i]) { emptyIdx = i; break; }
        }
        if (emptyIdx !== -1) {
            const toAdd = Math.min(64, amount);
            inventory[emptyIdx] = { type: type, count: toAdd };
            amount -= toAdd;
        } else {
             renderUI();
             return false;
        }
    }
    renderUI();
    return true;
}

function dropSelectedItem(dropAll = false) {
    const item = inventory[activeSlotIndex];
    if (!item || item.count <= 0) return false;

    const dropCount = dropAll ? item.count : 1;
    const dropType = item.type;
    const dir = new THREE.Vector3();
    viewCamera.getWorldDirection(dir);
    const baseX = player.pos.x + dir.x * 0.9;
    const baseY = player.pos.y + PLAYER_HEIGHT * 0.35;
    const baseZ = player.pos.z + dir.z * 0.9;

    item.count -= dropCount;
    if (item.count <= 0) inventory[activeSlotIndex] = null;

    spawnDrop(baseX, baseY, baseZ, dropType, dropCount, true);

    renderUI();
    updateCursorVisual();
    swingHand();
    return true;
}

// --- WORLD ---
const world = {
    getBlock: (wx, wy, wz) => {
        const cx = Math.floor(wx / CHUNK_SIZE);
        const cz = Math.floor(wz / CHUNK_SIZE);
        const lx = Math.floor(wx) - cx * CHUNK_SIZE;
        const ly = Math.floor(wy);
        const lz = Math.floor(wz) - cz * CHUNK_SIZE;

        const key = getChunkKey(cx, cz);
        let chunk = chunks.get(key);
        // ВАЖНО: если чанка ещё нет — создаём сразу, иначе физика может "провалиться" в пустоту
        if (!chunk) {
            chunk = new Chunk(cx, cz);
            chunks.set(key, chunk);
        }
        return chunk.getBlockLocal(lx, ly, lz);
    },
    setBlock: (wx, wy, wz, type) => {
        const cx = Math.floor(wx / CHUNK_SIZE);
        const cz = Math.floor(wz / CHUNK_SIZE);
        const lx = Math.floor(wx) - cx * CHUNK_SIZE;
        const ly = Math.floor(wy);
        const lz = Math.floor(wz) - cz * CHUNK_SIZE;

        const key = getChunkKey(cx, cz);
        
        if (!savedChunkMods.has(key)) savedChunkMods.set(key, []);
        const mods = savedChunkMods.get(key);
        const existIdx = mods.findIndex(m => m.x === lx && m.y === ly && m.z === lz);
        if (existIdx !== -1) mods.splice(existIdx, 1);
        mods.push({x:lx, y:ly, z:lz, type});

        const chunk = chunks.get(key);
        if (chunk) {
            // Spawn drop if breaking
            const currentBlock = chunk.getBlockLocal(lx, ly, lz);
            if (type === 0 && currentBlock !== 0 && currentBlock !== 6) {
                spawnDrop(wx, wy, wz, currentBlock);
            }
            
            chunk.setBlockLocal(lx, ly, lz, type);
            chunk.buildMesh();
        }
    },
    update: (px, pz) => {
        const cx = Math.floor(px / CHUNK_SIZE);
        const cz = Math.floor(pz / CHUNK_SIZE);

        for (let x = cx - RENDER_DISTANCE; x <= cx + RENDER_DISTANCE; x++) {
            for (let z = cz - RENDER_DISTANCE; z <= cz + RENDER_DISTANCE; z++) {
                const key = getChunkKey(x, z);
                if (!chunks.has(key)) {
                    chunks.set(key, new Chunk(x, z));
                }
            }
        }
        for (const [key, chunk] of chunks) {
            const dist = Math.sqrt((chunk.cx - cx)**2 + (chunk.cz - cz)**2);
            if (dist > RENDER_DISTANCE + 1) {
                chunk.dispose();
                chunks.delete(key);
            }
        }
    }
};

// --- GAMEPLAY ---
const controls = new PointerLockControls(camera, document.body);
const player = {
    pos: new THREE.Vector3(0, 30, 0),
    vel: new THREE.Vector3(),
    onGround: false
};
const keys = { w:false, a:false, s:false, d:false, space:false, shift:false };

// --- Mobile / touch controls ---
const isTouchDevice = (('ontouchstart' in window) || (navigator.maxTouchPoints > 0));
const mobileControlsEl = document.getElementById('mobile-controls');
if (isTouchDevice && mobileControlsEl) {
    mobileControlsEl.style.display = 'block';

    // Touch buttons -> keys
    const setKey = (k, v) => {
        if (k === 'space') keys.space = v;
        else if (k in keys) keys[k] = v;
    };

    const btns = mobileControlsEl.querySelectorAll('.m-btn');
    btns.forEach(btn => {
        const k = btn.dataset.key;
        const start = (e) => { e.preventDefault(); setKey(k, true); };
        const end = (e) => { e.preventDefault(); setKey(k, false); };
        btn.addEventListener('touchstart', start, {passive:false});
        btn.addEventListener('touchend', end, {passive:false});
        btn.addEventListener('touchcancel', end, {passive:false});
    });

    // Touch look (drag anywhere outside buttons)
    let touchLookActive = false;
    let lastX = 0, lastY = 0;
    let yaw = 0, pitch = 0;

    // init from current camera rotation
    camera.rotation.order = 'YXZ';
    yaw = camera.rotation.y;
    pitch = camera.rotation.x;

    const isOnMobileButton = (target) => !!(target && target.closest && target.closest('#mobile-controls'));

    window.addEventListener('touchstart', (e) => {
        if (e.touches.length !== 1) return;
        if (isInventoryOpen || specialScreenOpen || menuOverlay.style.display !== 'none') return;
        const t = e.touches[0];
        if (isOnMobileButton(e.target) || (e.target && e.target.closest && e.target.closest('#hotbar'))) return;
        touchLookActive = true;
        lastX = t.clientX; lastY = t.clientY;
    }, {passive:false});

    window.addEventListener('touchmove', (e) => {
        if (isInventoryOpen || specialScreenOpen || menuOverlay.style.display !== 'none' || mobileDropTouchActive) {
            touchLookActive = false;
            return;
        }
        if (!touchLookActive || e.touches.length !== 1) return;
        const t = e.touches[0];
        const dx = t.clientX - lastX;
        const dy = t.clientY - lastY;
        lastX = t.clientX; lastY = t.clientY;

        const sens = 0.003;
        yaw -= dx * sens;
        pitch -= dy * sens;
        const lim = Math.PI/2 - 0.05;
        pitch = Math.max(-lim, Math.min(lim, pitch));

        camera.rotation.y = yaw;
        camera.rotation.x = pitch;
    }, {passive:false});

    window.addEventListener('touchend', () => { touchLookActive = false; }, {passive:true});
}


camera.position.copy(player.pos);

function getSurfaceY(wx, wz) {
  // обеспечить чанки
  world.update(wx, wz);
  const cx = Math.floor(wx / CHUNK_SIZE);
  const cz = Math.floor(wz / CHUNK_SIZE);
  const lx = Math.floor(wx) - cx * CHUNK_SIZE;
  const lz = Math.floor(wz) - cz * CHUNK_SIZE;
  const key = getChunkKey(cx, cz);
  const chunk = chunks.get(key);
  if (!chunk) return 0;
  // ищем самый верхний твёрдый блок
  for (let y = 80; y >= 0; y--) {
    const t = chunk.getBlockLocal(lx, y, lz);
    if (t !== 0 && t !== 6) return y;
  }
  return 0;
}

function checkCollision(newPos) {
    const x = newPos.x;
    const y = newPos.y;
    const z = newPos.z;

    // гарантируем чанки вокруг точки, чтобы коллизия не считала "пустоту"
    world.update(x, z);

    const minX = Math.floor(x - PLAYER_RADIUS);
    const maxX = Math.floor(x + PLAYER_RADIUS);
    const minY = Math.floor(y); 
    const maxY = Math.floor(y + PLAYER_HEIGHT); 
    const minZ = Math.floor(z - PLAYER_RADIUS);
    const maxZ = Math.floor(z + PLAYER_RADIUS);

    for (let bx = minX; bx <= maxX; bx++) {
        for (let by = minY; by <= maxY; by++) {
            for (let bz = minZ; bz <= maxZ; bz++) {
                const block = world.getBlock(bx, by, bz);
                if (isSolidBlock(block)) { 
                     return true;
                }
            }
        }
    }
    return false;
}

function updatePhysics(dt) {
    if (controls.isLocked || isTouchDevice) {
        // Flight mode (creative): no gravity, vertical movement with SPACE (up) / Shift (down)
        if (flyMode) {
            // damping
            player.vel.y -= player.vel.y * 8 * dt;
            const flySpeed = (isSprinting ? SPEED * 2.2 : SPEED * 1.6);
            if (keys.space) player.vel.y = flySpeed;
            else if (keys.shift) player.vel.y = -flySpeed;
            else player.vel.y = 0;
        } else {
            player.vel.y -= GRAVITY * dt;
        }

        const forward = new THREE.Vector3(0,0,-1).applyQuaternion(camera.quaternion);
        forward.y = 0; forward.normalize();
        const right = new THREE.Vector3(1,0,0).applyQuaternion(camera.quaternion);
        right.y = 0; right.normalize();

        const moveDir = new THREE.Vector3();
        if (keys.w) moveDir.add(forward);
        if (keys.s) moveDir.sub(forward);
        if (keys.d) moveDir.add(right);
        if (keys.a) moveDir.sub(right);

        if (moveDir.length() > 0) moveDir.normalize();
        else isSprinting = false; // Disable sprint if stopped
        
        player.vel.x -= player.vel.x * 10 * dt; 
        player.vel.z -= player.vel.z * 10 * dt;
        
        const currentSpeed = isSprinting ? SPEED * 1.8 : SPEED;
        player.vel.x += moveDir.x * currentSpeed * 10 * dt; 
        player.vel.z += moveDir.z * currentSpeed * 10 * dt;

        if (!flyMode) {
            if (keys.space && player.onGround) {
                player.vel.y = JUMP_FORCE;
                player.onGround = false;
            }
        } else {
            player.onGround = false;
        }
        
        let nextX = player.pos.x + player.vel.x * dt;
        if (checkCollision(new THREE.Vector3(nextX, player.pos.y, player.pos.z))) {
            player.vel.x = 0; 
        } else {
            player.pos.x = nextX;
        }

        let nextZ = player.pos.z + player.vel.z * dt;
        if (checkCollision(new THREE.Vector3(player.pos.x, player.pos.y, nextZ))) {
            player.vel.z = 0;
        } else {
            player.pos.z = nextZ;
        }
        
        let nextY = player.pos.y + player.vel.y * dt;
        if (checkCollision(new THREE.Vector3(player.pos.x, nextY, player.pos.z))) {
             if (!flyMode && player.vel.y < 0) player.onGround = true;
             player.vel.y = 0;
        } else {
            player.pos.y = nextY;
            player.onGround = false;
        }

        if (player.pos.y < -30) {
            player.pos.set(0, 40, 0);
            player.vel.set(0, 0, 0);
        }

        camera.position.copy(player.pos);
        camera.position.y += PLAYER_HEIGHT * 0.9; 
    }
}

const raycaster = new THREE.Raycaster();
raycaster.far = 5;
const center = new THREE.Vector2(0, 0);

const wireGeo = new THREE.BoxGeometry(1.005, 1.005, 1.005);
const wireMat = new THREE.LineBasicMaterial({ color: 0x000000, linewidth: 2 });
const wireMesh = new THREE.LineSegments(new THREE.EdgesGeometry(wireGeo), wireMat);
scene.add(wireMesh);
wireMesh.visible = false;

let highlightPos = null;
let buildPos = null;

function updateRaycaster() {
    raycaster.setFromCamera(center, camera);
    const targets = Array.from(chunks.values()).map(c => c.meshGroup);
    const intersects = raycaster.intersectObjects(targets, true); 
    
    if (intersects.length > 0) {
        for (let hit of intersects) {
            if (hit.object.isInstancedMesh) {
                const p = hit.point.clone().add(hit.face.normal.clone().multiplyScalar(-0.001));
                const bx = Math.floor(p.x);
                const by = Math.floor(p.y);
                const bz = Math.floor(p.z);
                
                wireMesh.position.set(bx + 0.5, by + 0.5, bz + 0.5);
                wireMesh.visible = true;
                highlightPos = { x: bx, y: by, z: bz };
                
                const bp = hit.point.clone().add(hit.face.normal.clone().multiplyScalar(0.001));
                buildPos = { x: Math.floor(bp.x), y: Math.floor(bp.y), z: Math.floor(bp.z) };
                return;
            }
        }
    }
    wireMesh.visible = false;
    highlightPos = null;
    buildPos = null;
}

// --- UI & EVENTS ---
let activeSlotIndex = 0; // 0-8
const menuOverlay = document.getElementById('menu-overlay');
const resumeBtn = document.getElementById('resume-btn');
const singleBtn = document.getElementById('single-btn');
const volumeSlider = document.getElementById('volume-slider');
const inventoryScreen = document.getElementById('inventory-screen');
const hotbarDiv = document.getElementById('hotbar');
const inventoryGrid = document.getElementById('inventory-grid');
const inventoryHotbarGrid = document.getElementById('inventory-hotbar-grid');
const cursorItemEl = document.getElementById('cursor-item');
const nameStatusEl = document.getElementById('name-status');

let isInventoryOpen = false;
let touchSplitActive = false;
const touchSplitVisited = new Set();
let slotTouchIdCounter = 0;
let dropHoldTimer = null;
let dropHoldTriggered = false;
let mobileDropTouchActive = false;
let mobileDropEligible = false;
let mobileDropHoldTimer = null;
let mobileDropDroppedAll = false;

function renderUI() {
    // Render Hotbar
    hotbarDiv.innerHTML = '';
    for (let i = 0; i < 9; i++) {
        const slotEl = document.createElement('div');
        slotEl.className = 'slot' + (i === activeSlotIndex ? ' active' : '');
        slotEl.dataset.id = i;
        slotEl.onclick = (e) => {
            e.preventDefault();
            activeSlotIndex = i;
            renderUI();
        };
        slotEl.addEventListener('touchstart', (e) => {
            if (!isTouchDevice) return;
            e.preventDefault();
            beginMobileHotbarGesture(i);
        }, { passive: false });
        createSlotContent(slotEl, inventory[i]);
        hotbarDiv.appendChild(slotEl);
    }
    
    // Render Inventory Screen
    if (isInventoryOpen) {
        inventoryGrid.innerHTML = '';
        // Main inventory slots (9-35)
        for (let i = 9; i < 36; i++) {
            const slotEl = document.createElement('div');
            slotEl.className = 'slot';
            bindInventorySlot(slotEl, i);
            createSlotContent(slotEl, inventory[i]);
            inventoryGrid.appendChild(slotEl);
        }
        
        if (inventoryHotbarGrid) {
            inventoryHotbarGrid.innerHTML = '';
            // Hotbar slots (0-8)
            for (let i = 0; i < 9; i++) {
                const slotEl = document.createElement('div');
                slotEl.className = 'slot';
                bindInventorySlot(slotEl, i);
                createSlotContent(slotEl, inventory[i]);
                inventoryHotbarGrid.appendChild(slotEl);
            }
        }
    }
}

// --- ШАГИ (step_grass / step_stone) ---
let stepDistAcc = 0;
function updateFootsteps(dt) {
  if (!controls.isLocked || isInventoryOpen) { stepDistAcc = 0; return; }
  if (!player.onGround) { stepDistAcc = 0; return; }
  const speed = Math.hypot(player.vel.x, player.vel.z);
  if (speed < 0.2) { stepDistAcc = 0; return; }

  stepDistAcc += speed * dt;
  if (stepDistAcc >= 1.6) { // примерно как шаг в Minecraft
    stepDistAcc = 0;
    const under = world.getBlock(player.pos.x, player.pos.y - PLAYER_HEIGHT - 0.08, player.pos.z);
    playSound(isStoneLikeBlock(under) ? 'step_stone' : 'step_grass');
  }
}

function createSlotContent(slot, item) {
    if (item) {
        const img = document.createElement('img');
        img.src = itemIcons[item.type];
        slot.appendChild(img);
        
        const count = document.createElement('div');
        count.className = 'item-count';
        count.innerText = item.count;
        slot.appendChild(count);
    }
}



function setNameStatus(text = '', isError = true) {
    if (!nameStatusEl) return;
    nameStatusEl.textContent = text || '';
    nameStatusEl.style.color = text ? (isError ? '#ff9b9b' : '#a8ff9b') : '#ff9b9b';
}

function setCursorVisualPosition(clientX, clientY) {
    if (!cursorItemEl) return;
    cursorItemEl.style.left = (clientX - 20) + 'px';
    cursorItemEl.style.top = (clientY - 20) + 'px';
}

function clearDropHoldTimers() {
    if (dropHoldTimer) {
        clearTimeout(dropHoldTimer);
        dropHoldTimer = null;
    }
    dropHoldTriggered = false;
}

function stopTouchSplit() {
    touchSplitActive = false;
    touchSplitVisited.clear();
}

function clearMobileDropGesture() {
    if (mobileDropHoldTimer) {
        clearTimeout(mobileDropHoldTimer);
        mobileDropHoldTimer = null;
    }
    mobileDropTouchActive = false;
    mobileDropEligible = false;
    mobileDropDroppedAll = false;
}

function applyTouchSplitToElement(slotEl) {
    if (!touchSplitActive || !cursorItem || !slotEl || !slotEl._mcAccess) return;
    const slotId = slotEl.dataset.touchSlotId || '';
    if (touchSplitVisited.has(slotId)) return;
    touchSplitVisited.add(slotId);

    const access = slotEl._mcAccess;
    if (access.takeOnly) return;
    rightClickSlotByAccess(access.getter, access.setter, access.acceptFn);
}

function attachSlotInteractions(slotEl, leftHandler, rightHandler = null, access = null) {
    if (!slotEl) return;

    slotEl.onclick = (e) => {
        e.preventDefault();
        leftHandler();
    };
    slotEl.oncontextmenu = (e) => {
        e.preventDefault();
        if (rightHandler) rightHandler();
        return false;
    };

    slotEl._mcAccess = access;
    slotEl.dataset.touchSlotId = `slot-${++slotTouchIdCounter}`;

    slotEl.addEventListener('touchstart', (e) => {
        if (!isTouchDevice) return;
        e.preventDefault();
        const touch = e.touches && e.touches[0];
        if (touch) setCursorVisualPosition(touch.clientX, touch.clientY);

        if (cursorItem && access && !access.takeOnly) {
            touchSplitActive = true;
            touchSplitVisited.clear();
            applyTouchSplitToElement(slotEl);
            return;
        }

        leftHandler();
    }, { passive: false });
}

function bindInventorySlot(slotEl, index) {
    attachSlotInteractions(
        slotEl,
        () => handleSlotClick(index),
        () => handleSlotRightClick(index),
        {
            getter: () => inventory[index],
            setter: (v) => { inventory[index] = v; },
            acceptFn: null,
            takeOnly: false
        }
    );
}

function beginMobileHotbarGesture(index) {
    activeSlotIndex = index;
    renderUI();

    if (!isTouchDevice || isInventoryOpen || specialScreenOpen || menuOverlay.style.display !== 'none') {
        clearMobileDropGesture();
        return;
    }

    if (!inventory[index]) {
        clearMobileDropGesture();
        return;
    }

    mobileDropTouchActive = true;
    mobileDropEligible = false;
    mobileDropDroppedAll = false;
    if (mobileDropHoldTimer) clearTimeout(mobileDropHoldTimer);
    mobileDropHoldTimer = setTimeout(() => {
        if (mobileDropTouchActive && mobileDropEligible) {
            mobileDropDroppedAll = dropSelectedItem(true);
        }
        mobileDropHoldTimer = null;
    }, 3000);
}

function updateMobileDropGesture(clientY) {
    if (!mobileDropTouchActive) return;
    const rect = hotbarDiv ? hotbarDiv.getBoundingClientRect() : null;
    if (!rect) return;
    mobileDropEligible = clientY < (rect.top - 18);
}

function finishMobileDropGesture() {
    if (!mobileDropTouchActive) return;
    if (mobileDropHoldTimer) {
        clearTimeout(mobileDropHoldTimer);
        mobileDropHoldTimer = null;
    }
    if (mobileDropEligible && !mobileDropDroppedAll) {
        dropSelectedItem(false);
    }
    mobileDropTouchActive = false;
    mobileDropEligible = false;
    mobileDropDroppedAll = false;
}
function handleSlotClick(index) {
    const item = inventory[index];
    
    if (!cursorItem) {
        if (item) {
            cursorItem = item;
            inventory[index] = null;
        }
    } else {
        if (!item) {
            inventory[index] = cursorItem;
            cursorItem = null;
        } else {
            if (item.type === cursorItem.type) {
                const space = 64 - item.count;
                const toAdd = Math.min(space, cursorItem.count);
                item.count += toAdd;
                cursorItem.count -= toAdd;
                if (cursorItem.count <= 0) cursorItem = null;
            } else {
                const temp = item;
                inventory[index] = cursorItem;
                cursorItem = temp; 
            }
        }
    }
    renderUI();
    updateCursorVisual();
}


function handleSlotRightClick(index) {
    rightClickSlotByAccess(
        () => inventory[index],
        (v) => { inventory[index] = v; }
    );
}

function updateCursorVisual() {
    if (cursorItem && cursorItemEl) {
        cursorItemEl.style.display = 'block';
        const img = cursorItemEl.querySelector('img');
        if(img) img.src = itemIcons[cursorItem.type];
        const count = cursorItemEl.querySelector('.item-count');
        if(count) count.innerText = cursorItem.count;
    } else if (cursorItemEl) {
        cursorItemEl.style.display = 'none';
    }
}

document.addEventListener('mousemove', (e) => {
    if (cursorItem && cursorItemEl) {
        setCursorVisualPosition(e.pageX, e.pageY);
    }
});

document.addEventListener('touchmove', (e) => {
    if (e.touches.length !== 1) return;
    const touch = e.touches[0];

    if (cursorItem && cursorItemEl) {
        setCursorVisualPosition(touch.clientX, touch.clientY);
    }
    if (mobileDropTouchActive) {
        updateMobileDropGesture(touch.clientY);
    }

    if (!touchSplitActive) return;

    const hovered = document.elementFromPoint(touch.clientX, touch.clientY);
    const slotEl = hovered && hovered.closest ? hovered.closest('.slot') : null;
    if (!slotEl) return;

    e.preventDefault();
    applyTouchSplitToElement(slotEl);
}, { passive: false });

document.addEventListener('touchend', () => {
    stopTouchSplit();
    finishMobileDropGesture();
}, { passive: true });
document.addEventListener('touchcancel', () => {
    stopTouchSplit();
    clearMobileDropGesture();
}, { passive: true });

function toggleInventory() {
    isInventoryOpen = !isInventoryOpen;
    clearDropHoldTimers();
    clearMobileDropGesture();
    if (isInventoryOpen) {
        controls.unlock();
        inventoryScreen.style.display = 'flex';
        renderUI();
        updateCursorVisual();
    } else {
        controls.lock();
        inventoryScreen.style.display = 'none';
        menuOverlay.style.display = 'none';

        if (cursorItem) {
             addToInventory(cursorItem.type, cursorItem.count);
             cursorItem = null;
             updateCursorVisual();
        }
    }
}

// основной обработчик кнопки сети ниже
volumeSlider.addEventListener('input', (e) => {
    setSoundVolume(parseFloat(e.target.value));
});

controls.addEventListener('lock', () => {
    clearDropHoldTimers();
    clearMobileDropGesture();
    menuOverlay.style.display = 'none';
    inventoryScreen.style.display = 'none';
    isInventoryOpen = false;
    if(cursorItemEl) cursorItemEl.style.display = 'none';
});
controls.addEventListener('unlock', () => {
    clearDropHoldTimers();
    clearMobileDropGesture();
    if (!isInventoryOpen) menuOverlay.style.display = 'flex';
});

document.addEventListener('keydown', (e) => {
    if (e.code === 'KeyQ') {
        if (!e.repeat && controls.isLocked && !isInventoryOpen && !specialScreenOpen && !net.chatOpen) {
            clearDropHoldTimers();
            dropHoldTimer = setTimeout(() => {
                dropHoldTriggered = dropSelectedItem(true);
                dropHoldTimer = null;
            }, 3000);
        }
        return;
    }

    // Камера: F5 (как в Minecraft) — спереди -> сзади -> первое лицо
    if (e.code === 'F5' && controls.isLocked && !isInventoryOpen) {
        e.preventDefault();
        cameraMode = (cameraMode + 1) % 3;
        return;
    }

    if (e.code === 'KeyE') {
        toggleInventory();
        return;
    }

    if (isInventoryOpen) return; // Block input when inventory open

    switch (e.code) {
        case 'KeyW': keys.w = true; break;
        case 'KeyA': keys.a = true; break;
        case 'KeyS': keys.s = true; break;
        case 'KeyD': keys.d = true; break;
        case 'Space': keys.space = true; break;
        case 'ShiftLeft': keys.shift = true; break;
        case 'ShiftRight': keys.shift = true; break;
        case 'ControlLeft': isSprinting = !isSprinting; break;
        case 'Digit1': activeSlotIndex = 0; break;
        case 'Digit2': activeSlotIndex = 1; break;
        case 'Digit3': activeSlotIndex = 2; break;
        case 'Digit4': activeSlotIndex = 3; break;
        case 'Digit5': activeSlotIndex = 4; break;
        case 'Digit6': activeSlotIndex = 5; break;
        case 'Digit7': activeSlotIndex = 6; break;
        case 'Digit8': activeSlotIndex = 7; break;
        case 'Digit9': activeSlotIndex = 8; break;
    }
    if(e.code >= 'Digit1' && e.code <= 'Digit9') renderUI();
});
document.addEventListener('keyup', (e) => {
    if (e.code === 'KeyQ') {
        if (dropHoldTimer) {
            clearTimeout(dropHoldTimer);
            dropHoldTimer = null;
            if (!dropHoldTriggered && controls.isLocked && !isInventoryOpen && !specialScreenOpen && !net.chatOpen) {
                dropSelectedItem(false);
            }
        }
        dropHoldTriggered = false;
        return;
    }

    switch (e.code) {
        case 'KeyW': keys.w = false; break;
        case 'KeyA': keys.a = false; break;
        case 'KeyS': keys.s = false; break;
        case 'KeyD': keys.d = false; break;
        case 'Space': keys.space = false; break;
    }
});
window.addEventListener('wheel', (e) => {
    if (isInventoryOpen) return;
    if (e.deltaY > 0) activeSlotIndex = (activeSlotIndex + 1) % 9;
    else activeSlotIndex = (activeSlotIndex - 1 + 9) % 9;
    renderUI();
});

document.addEventListener('mousedown', (e) => {
    if (!controls.isLocked || isInventoryOpen) return;
    
    if (e.button === 0 && highlightPos) { 
        const bt = world.getBlock(highlightPos.x, highlightPos.y, highlightPos.z);
        world.setBlock(highlightPos.x, highlightPos.y, highlightPos.z, 0);
        playDigForBlock(bt);
        swingHand();
    }
    if (e.button === 0 && !highlightPos) {
        swingHand();
    }
    if (e.button === 2 && buildPos) { 
        swingHand();
        const item = inventory[activeSlotIndex];
        if (!item) return;

        const pBoxMin = new THREE.Vector3(player.pos.x - 0.3, player.pos.y, player.pos.z - 0.3);
        const pBoxMax = new THREE.Vector3(player.pos.x + 0.3, player.pos.y + 1.8, player.pos.z + 0.3);
        const bBoxMaxPos = new THREE.Vector3(buildPos.x+1, buildPos.y+1, buildPos.z+1);
        
        const intersect = (pBoxMin.x < bBoxMaxPos.x && pBoxMax.x > buildPos.x) &&
                          (pBoxMin.y < bBoxMaxPos.y && pBoxMax.y > buildPos.y) &&
                          (pBoxMin.z < bBoxMaxPos.z && pBoxMax.z > buildPos.z);
                          
        if (!intersect) {
            world.setBlock(buildPos.x, buildPos.y, buildPos.z, item.type);
            item.count--;
            if (item.count <= 0) {
                inventory[activeSlotIndex] = null;
            }
            renderUI();
        }
    }
});

let prevTime = performance.now();

function animate() {
    requestAnimationFrame(animate);

    const time = performance.now();
    const dt = Math.min((time - prevTime) / 1000, 0.1); 
    prevTime = time;

    world.update(player.pos.x, player.pos.z);

    updatePhysics(dt);
    updateFootsteps(dt);
    updateDrops(dt);
    updateHand(dt);
    updateClouds(dt);
    world.update(player.pos.x, player.pos.z);
    updateRaycaster();

    netTick(dt);
    // анимация всех удалённых игроков
    const tSec = time / 1000;
    for (const [id, item] of net.remoteMeshes) {
      animateAvatar(item.avatar, item.speed || 0, tSec);
    }

    // анимация локального игрока (когда 3-е лицо)
    if (localAvatar && localAvatar.visible) {
      const sp = Math.sqrt(player.vel.x * player.vel.x + player.vel.z * player.vel.z);
      animateAvatar(localAvatar, sp, tSec);
    }

    updateViewCamera();

    // Render main scene
    renderer.render(scene, viewCamera);
    
    // Render hand only in first-person
    if (cameraMode === 0) {
        renderer.autoClear = false;
        renderer.clearDepth();
        renderer.render(handScene, handCamera);
        renderer.autoClear = true;
    }

    // мини-превью скина в инвентаре
    updateInvPreview(tSec);
}


world.update(0,0);

// ===============================
// Доп. система: сундук / мини-крафт / верстак / печь
// ===============================
const BLOCK_FURNACE = 19;
const ITEM_COAL = 20;
const ITEM_IRON_INGOT = 21;
const ITEM_GOLD_INGOT = 22;
const ITEM_DIAMOND = 23;
const ITEM_STICK = 24;

textures.furnace = loadTex('furnace.png');
textures.coalItem = loadTex('coal_item.png');
textures.ironIngot = loadTex('iron_ingot.png');
textures.goldIngot = loadTex('gold_ingot.png');
textures.diamondItem = loadTex('diamond_item.png');
textures.stickItem = loadTex('stick_item.png');

materials[BLOCK_FURNACE] = new THREE.MeshLambertMaterial({ map: textures.furnace });
materials[ITEM_COAL] = new THREE.MeshLambertMaterial({ map: textures.coalItem });
materials[ITEM_IRON_INGOT] = new THREE.MeshLambertMaterial({ map: textures.ironIngot });
materials[ITEM_GOLD_INGOT] = new THREE.MeshLambertMaterial({ map: textures.goldIngot });
materials[ITEM_DIAMOND] = new THREE.MeshLambertMaterial({ map: textures.diamondItem });
materials[ITEM_STICK] = new THREE.MeshLambertMaterial({ map: textures.stickItem });

itemIcons[BLOCK_FURNACE] = './Assets/furnace.png';
itemIcons[ITEM_COAL] = './Assets/coal_item.png';
itemIcons[ITEM_IRON_INGOT] = './Assets/iron_ingot.png';
itemIcons[ITEM_GOLD_INGOT] = './Assets/gold_ingot.png';
itemIcons[ITEM_DIAMOND] = './Assets/diamond_item.png';
itemIcons[ITEM_STICK] = './Assets/stick_item.png';

// Старт пустой: без бесплатных предметов.

const PLACEABLE_TYPES = new Set([1, 2, 3, 4, 5, 10, 11, 12, 13, 14, 15, 17, 18, BLOCK_FURNACE]);
const FURNACE_OUTPUTS = {
  12: ITEM_COAL,
  13: ITEM_IRON_INGOT,
  14: ITEM_GOLD_INGOT,
  15: ITEM_DIAMOND,
};
const FUEL_TYPES = new Set([4, 11, ITEM_STICK]);

const chestStores = new Map();
const furnaceStores = new Map();
const miniCraftSlots = [null, null, null, null];
const tableCraftSlots = [null, null, null, null, null, null, null, null, null];
let specialScreenOpen = null;
let openChestKey = null;
let openTableKey = null;
let openFurnaceKey = null;

function blockPosKey(x, y, z) {
  return `${Math.floor(x)},${Math.floor(y)},${Math.floor(z)}`;
}

function getChestStore(key) {
  if (!chestStores.has(key)) chestStores.set(key, new Array(27).fill(null));
  return chestStores.get(key);
}

function getFurnaceStore(key) {
  if (!furnaceStores.has(key)) {
    furnaceStores.set(key, {
      input: null,
      fuel: null,
      output: null,
      progress: 0,
    });
  }
  return furnaceStores.get(key);
}

function resetLocalWorldToClean(clearInventory = true) {
  net.enabled = false;
  net.lastSend = 0;
  net.playersInitialized = false;
  net.seenMods.clear();
  net.prevOnline.clear();
  net.prevNames.clear();
  for (const [id, item] of net.remoteMeshes) {
    if (!item || !item.avatar) continue;
    scene.remove(item.avatar);
    disposeObject3D(item.avatar);
  }
  net.remoteMeshes.clear();

  for (const d of drops) {
    if (d && d.mesh) scene.remove(d.mesh);
  }
  drops.length = 0;

  chestStores.clear();
  furnaceStores.clear();
  miniCraftSlots.fill(null);
  tableCraftSlots.fill(null);
  specialScreenOpen = null;
  openChestKey = null;
  openTableKey = null;
  openFurnaceKey = null;

  ['mc-chest-screen', 'mc-table-screen', 'mc-furnace-screen'].forEach((id) => {
    const el = document.getElementById(id);
    if (el) el.style.display = 'none';
  });

  if (clearInventory) {
    for (let i = 0; i < inventory.length; i++) inventory[i] = null;
    cursorItem = null;
    activeSlotIndex = 0;
  }

  clearDropHoldTimers();
  clearMobileDropGesture();

  setWorldSeed(DEFAULT_WORLD_SEED);
  for (const [key, chunk] of chunks) {
    chunk.dispose();
  }
  chunks.clear();
  savedChunkMods.clear();

  const spawnX = 0;
  const spawnZ = 0;
  const safeY = getSurfaceY(spawnX, spawnZ);
  player.pos.set(spawnX, safeY + PLAYER_HEIGHT + 0.2, spawnZ);
  player.vel.set(0, 0, 0);
  camera.position.copy(player.pos);

  if (localAvatar) {
    localAvatar.visible = false;
    localAvatar.position.copy(player.pos);
  }

  isInventoryOpen = false;
  inventoryScreen.style.display = 'none';
  world.update(player.pos.x, player.pos.z);
  renderUI();
  updateCursorVisual();
}

function canAcceptFurnaceInput(item) {
  return !!(item && FURNACE_OUTPUTS[item.type]);
}

function canAcceptFuel(item) {
  return !!(item && FUEL_TYPES.has(item.type));
}

function makePixelUI() {
  const style = document.createElement('style');
  style.textContent = `
  .mc-panel-overlay{position:absolute;inset:0;display:none;justify-content:center;align-items:center;background:rgba(0,0,0,0.72);z-index:18;}
  .mc-panel{background:#c6c6c6;border:4px solid #555;border-radius:6px;padding:18px;color:#222;max-width:min(96vw,780px);max-height:92vh;overflow:auto;box-sizing:border-box;}
  .mc-head{display:flex;justify-content:space-between;align-items:center;gap:10px;margin-bottom:10px;}
  .mc-title{font-size:14px;color:#111;}
  .mc-close{font-family:inherit;font-size:12px;padding:8px 12px;border:2px solid #333;background:#e3e3e3;cursor:pointer;}
  .mc-subtitle{font-size:10px;margin:14px 0 8px;color:#333;}
  .mc-row{display:flex;align-items:center;gap:12px;flex-wrap:wrap;}
  .mc-arrow{font-size:18px;color:#333;padding:0 2px;}
  .mc-grid-2,.mc-grid-3,.mc-grid-9{display:grid;gap:5px;}
  .mc-grid-2{grid-template-columns:repeat(2, 50px);}
  .mc-grid-3{grid-template-columns:repeat(3, 50px);}
  .mc-grid-9{grid-template-columns:repeat(9, 50px);}
  .mc-output-slot{background:#b8c8b8 !important;}
  .mc-sep{height:10px;}
  .mc-furnace-wrap{display:flex;align-items:center;gap:14px;flex-wrap:wrap;}
  .mc-furnace-left{display:flex;flex-direction:column;gap:10px;}
  .mc-progress{width:110px;height:12px;background:#6e6e6e;border:2px solid #333;position:relative;}
  .mc-progress-fill{height:100%;width:0%;background:#f39c12;}
  #mc-mini-craft-wrap{margin:10px 0 12px;display:flex;flex-direction:column;gap:8px;}
  #mc-mini-craft-wrap .mc-subtitle{margin:0;}
  `;
  document.head.appendChild(style);

  const inventoryContainer = document.getElementById('inventory-container');
  if (inventoryContainer && !document.getElementById('mc-mini-craft-wrap')) {
    const wrap = document.createElement('div');
    wrap.id = 'mc-mini-craft-wrap';
    wrap.innerHTML = `
      <div class="mc-subtitle">КРАФТ 2x2</div>
      <div class="mc-row">
        <div id="mc-mini-grid" class="mc-grid-2"></div>
        <div class="mc-arrow">→</div>
        <div id="mc-mini-result" class="slot mc-output-slot"></div>
      </div>
    `;
    inventoryContainer.insertBefore(wrap, inventoryGrid);
  }

  const chestScreen = document.createElement('div');
  chestScreen.id = 'mc-chest-screen';
  chestScreen.className = 'mc-panel-overlay';
  chestScreen.innerHTML = `
    <div class="mc-panel">
      <div class="mc-head">
        <div class="mc-title">СУНДУК</div>
        <button id="mc-chest-close" class="mc-close">X</button>
      </div>
      <div id="mc-chest-grid" class="mc-grid-9"></div>
      <div class="mc-subtitle">ИНВЕНТАРЬ</div>
      <div id="mc-chest-player-main" class="mc-grid-9"></div>
      <div class="mc-sep"></div>
      <div id="mc-chest-player-hotbar" class="mc-grid-9"></div>
    </div>
  `;
  document.body.appendChild(chestScreen);

  const tableScreen = document.createElement('div');
  tableScreen.id = 'mc-table-screen';
  tableScreen.className = 'mc-panel-overlay';
  tableScreen.innerHTML = `
    <div class="mc-panel">
      <div class="mc-head">
        <div class="mc-title">ВЕРСТАК 3x3</div>
        <button id="mc-table-close" class="mc-close">X</button>
      </div>
      <div class="mc-row">
        <div id="mc-table-grid" class="mc-grid-3"></div>
        <div class="mc-arrow">→</div>
        <div id="mc-table-result" class="slot mc-output-slot"></div>
      </div>
      <div class="mc-subtitle">ИНВЕНТАРЬ</div>
      <div id="mc-table-player-main" class="mc-grid-9"></div>
      <div class="mc-sep"></div>
      <div id="mc-table-player-hotbar" class="mc-grid-9"></div>
    </div>
  `;
  document.body.appendChild(tableScreen);

  const furnaceScreen = document.createElement('div');
  furnaceScreen.id = 'mc-furnace-screen';
  furnaceScreen.className = 'mc-panel-overlay';
  furnaceScreen.innerHTML = `
    <div class="mc-panel">
      <div class="mc-head">
        <div class="mc-title">ПЕЧЬ</div>
        <button id="mc-furnace-close" class="mc-close">X</button>
      </div>
      <div class="mc-furnace-wrap">
        <div class="mc-furnace-left">
          <div id="mc-furnace-input" class="slot"></div>
          <div id="mc-furnace-fuel" class="slot"></div>
        </div>
        <div class="mc-progress"><div id="mc-furnace-progress" class="mc-progress-fill"></div></div>
        <div id="mc-furnace-output" class="slot mc-output-slot"></div>
      </div>
      <div class="mc-subtitle">ИНВЕНТАРЬ</div>
      <div id="mc-furnace-player-main" class="mc-grid-9"></div>
      <div class="mc-sep"></div>
      <div id="mc-furnace-player-hotbar" class="mc-grid-9"></div>
    </div>
  `;
  document.body.appendChild(furnaceScreen);

  document.getElementById('mc-chest-close').onclick = () => closeAllSpecialScreens(true);
  document.getElementById('mc-table-close').onclick = () => closeAllSpecialScreens(true);
  document.getElementById('mc-furnace-close').onclick = () => closeAllSpecialScreens(true);
}

function renderInventoryRange(target, start, end) {
  if (!target) return;
  target.innerHTML = '';
  for (let i = start; i <= end; i++) {
    const slotEl = document.createElement('div');
    slotEl.className = 'slot';
    bindInventorySlot(slotEl, i);
    createSlotContent(slotEl, inventory[i]);
    target.appendChild(slotEl);
  }
}

function clickSlotByAccess(getter, setter, acceptFn = null) {
  const slotItem = getter();

  if (!cursorItem) {
    if (slotItem) {
      cursorItem = slotItem;
      setter(null);
    }
  } else {
    if (acceptFn && !acceptFn(cursorItem)) {
      renderUI();
      updateCursorVisual();
      return;
    }

    if (!slotItem) {
      setter(cursorItem);
      cursorItem = null;
    } else if (slotItem.type === cursorItem.type && slotItem.count < 64) {
      const space = 64 - slotItem.count;
      const toAdd = Math.min(space, cursorItem.count);
      slotItem.count += toAdd;
      cursorItem.count -= toAdd;
      if (cursorItem.count <= 0) cursorItem = null;
    } else {
      const temp = slotItem;
      setter(cursorItem);
      cursorItem = temp;
    }
  }

  renderUI();
  updateCursorVisual();
}

function rightClickSlotByAccess(getter, setter, acceptFn = null) {
  const slotItem = getter();

  if (!cursorItem) {
    if (!slotItem) return;

    if (slotItem.count <= 1) {
      cursorItem = slotItem;
      setter(null);
    } else {
      const takeCount = Math.ceil(slotItem.count / 2);
      cursorItem = { type: slotItem.type, count: takeCount };
      slotItem.count -= takeCount;
      if (slotItem.count <= 0) setter(null);
    }
  } else {
    if (acceptFn && !acceptFn(cursorItem)) return;

    if (!slotItem) {
      setter({ type: cursorItem.type, count: 1 });
      cursorItem.count -= 1;
      if (cursorItem.count <= 0) cursorItem = null;
    } else if (slotItem.type === cursorItem.type && slotItem.count < 64) {
      slotItem.count += 1;
      cursorItem.count -= 1;
      if (cursorItem.count <= 0) cursorItem = null;
    } else {
      return;
    }
  }

  renderUI();
  updateCursorVisual();
}

function clickTakeOnlySlot(getter, setter) {
  const slotItem = getter();
  if (!slotItem) return;

  if (!cursorItem) {
    cursorItem = slotItem;
    setter(null);
  } else if (cursorItem.type === slotItem.type && cursorItem.count < 64) {
    const space = 64 - cursorItem.count;
    const toMove = Math.min(space, slotItem.count);
    cursorItem.count += toMove;
    slotItem.count -= toMove;
    if (slotItem.count <= 0) setter(null);
  } else {
    return;
  }

  renderUI();
  updateCursorVisual();
}

function renderArraySlots(target, arr, acceptFn = null) {
  if (!target) return;
  target.innerHTML = '';
  for (let i = 0; i < arr.length; i++) {
    const slotEl = document.createElement('div');
    slotEl.className = 'slot';
    attachSlotInteractions(
      slotEl,
      () => clickSlotByAccess(() => arr[i], (v) => { arr[i] = v; }, acceptFn),
      () => rightClickSlotByAccess(() => arr[i], (v) => { arr[i] = v; }, acceptFn),
      {
        getter: () => arr[i],
        setter: (v) => { arr[i] = v; },
        acceptFn,
        takeOnly: false
      }
    );
    createSlotContent(slotEl, arr[i]);
    target.appendChild(slotEl);
  }
}

function renderSingleSlot(target, getter, setter, acceptFn = null, takeOnly = false) {
  if (!target) return;
  target.innerHTML = '';
  target.onclick = null;
  attachSlotInteractions(
    target,
    () => {
      if (takeOnly) clickTakeOnlySlot(getter, setter);
      else clickSlotByAccess(getter, setter, acceptFn);
    },
    takeOnly ? null : () => rightClickSlotByAccess(getter, setter, acceptFn),
    {
      getter,
      setter,
      acceptFn,
      takeOnly
    }
  );
  createSlotContent(target, getter());
}

function recipeForGrid(grid, width) {
  const nonEmpty = [];
  for (let i = 0; i < grid.length; i++) {
    if (grid[i]) nonEmpty.push({ idx: i, item: grid[i] });
  }
  if (nonEmpty.length === 0) return null;

  if (nonEmpty.length === 1 && nonEmpty[0].item.type === 4) {
    return { output: { type: 11, count: 4 }, use: [nonEmpty[0].idx] };
  }

  if (nonEmpty.length === 2 && nonEmpty.every((x) => x.item.type === 11)) {
    const ids = nonEmpty.map((x) => x.idx).sort((a, b) => a - b);
    if (ids[1] - ids[0] === width) {
      return { output: { type: ITEM_STICK, count: 4 }, use: ids };
    }
  }

  if (width === 2) {
    if (nonEmpty.length === 4 && grid.every((x) => x && x.type === 11)) {
      return { output: { type: 17, count: 1 }, use: [0, 1, 2, 3] };
    }
    return null;
  }

  const squares = [
    [0, 1, 3, 4],
    [1, 2, 4, 5],
    [3, 4, 6, 7],
    [4, 5, 7, 8],
  ];
  if (nonEmpty.length === 4) {
    for (const sq of squares) {
      if (sq.every((i) => grid[i] && grid[i].type === 11)) {
        const setSq = new Set(sq);
        const extra = nonEmpty.some((n) => !setSq.has(n.idx));
        if (!extra) return { output: { type: 17, count: 1 }, use: sq };
      }
    }
  }

  const verticalPairs = [
    [0, 3], [3, 6],
    [1, 4], [4, 7],
    [2, 5], [5, 8],
  ];
  if (nonEmpty.length === 2 && nonEmpty.every((x) => x.item.type === 11)) {
    for (const pair of verticalPairs) {
      const setPair = new Set(pair);
      if (nonEmpty.every((n) => setPair.has(n.idx))) {
        return { output: { type: ITEM_STICK, count: 4 }, use: pair };
      }
    }
  }

  const ring = [0, 1, 2, 3, 5, 6, 7, 8];
  if (nonEmpty.length === 8 && !grid[4]) {
    if (ring.every((i) => grid[i] && grid[i].type === 11)) {
      return { output: { type: 18, count: 1 }, use: ring };
    }
    if (ring.every((i) => grid[i] && grid[i].type === 3)) {
      return { output: { type: BLOCK_FURNACE, count: 1 }, use: ring };
    }
  }

  return null;
}

function takeCraftResult(grid, width) {
  const recipe = recipeForGrid(grid, width);
  if (!recipe) return;

  if (cursorItem && cursorItem.type !== recipe.output.type) return;
  if (cursorItem && (cursorItem.count + recipe.output.count > 64)) return;

  if (!cursorItem) cursorItem = { type: recipe.output.type, count: recipe.output.count };
  else cursorItem.count += recipe.output.count;

  for (const idx of recipe.use) {
    if (!grid[idx]) continue;
    grid[idx].count -= 1;
    if (grid[idx].count <= 0) grid[idx] = null;
  }

  playSound('craft');
  renderUI();
  updateCursorVisual();
}

function renderMiniCraft() {
  const miniGridEl = document.getElementById('mc-mini-grid');
  const miniResultEl = document.getElementById('mc-mini-result');
  if (!miniGridEl || !miniResultEl) return;

  if (!isInventoryOpen || specialScreenOpen) {
    miniGridEl.innerHTML = '';
    miniResultEl.innerHTML = '';
    miniResultEl.onclick = null;
    return;
  }

  renderArraySlots(miniGridEl, miniCraftSlots);
  const recipe = recipeForGrid(miniCraftSlots, 2);
  miniResultEl.innerHTML = '';
  if (recipe) createSlotContent(miniResultEl, recipe.output);
  miniResultEl.onclick = () => takeCraftResult(miniCraftSlots, 2);
}

function renderChestScreen() {
  const screen = document.getElementById('mc-chest-screen');
  if (!screen) return;
  if (specialScreenOpen !== 'chest' || !openChestKey) {
    screen.style.display = 'none';
    return;
  }

  const chestInv = getChestStore(openChestKey);
  screen.style.display = 'flex';
  renderArraySlots(document.getElementById('mc-chest-grid'), chestInv);
  renderInventoryRange(document.getElementById('mc-chest-player-main'), 9, 35);
  renderInventoryRange(document.getElementById('mc-chest-player-hotbar'), 0, 8);
}

function renderTableScreen() {
  const screen = document.getElementById('mc-table-screen');
  if (!screen) return;
  if (specialScreenOpen !== 'table') {
    screen.style.display = 'none';
    return;
  }

  screen.style.display = 'flex';
  renderArraySlots(document.getElementById('mc-table-grid'), tableCraftSlots);

  const resultEl = document.getElementById('mc-table-result');
  if (resultEl) {
    resultEl.innerHTML = '';
    const recipe = recipeForGrid(tableCraftSlots, 3);
    if (recipe) createSlotContent(resultEl, recipe.output);
    resultEl.onclick = () => takeCraftResult(tableCraftSlots, 3);
  }

  renderInventoryRange(document.getElementById('mc-table-player-main'), 9, 35);
  renderInventoryRange(document.getElementById('mc-table-player-hotbar'), 0, 8);
}

function renderFurnaceScreen() {
  const screen = document.getElementById('mc-furnace-screen');
  if (!screen) return;
  if (specialScreenOpen !== 'furnace' || !openFurnaceKey) {
    screen.style.display = 'none';
    return;
  }

  const furnace = getFurnaceStore(openFurnaceKey);
  screen.style.display = 'flex';

  renderSingleSlot(
    document.getElementById('mc-furnace-input'),
    () => furnace.input,
    (v) => { furnace.input = v; },
    canAcceptFurnaceInput,
    false
  );
  renderSingleSlot(
    document.getElementById('mc-furnace-fuel'),
    () => furnace.fuel,
    (v) => { furnace.fuel = v; },
    canAcceptFuel,
    false
  );
  renderSingleSlot(
    document.getElementById('mc-furnace-output'),
    () => furnace.output,
    (v) => { furnace.output = v; },
    null,
    true
  );

  const progEl = document.getElementById('mc-furnace-progress');
  if (progEl) progEl.style.width = `${Math.max(0, Math.min(100, (furnace.progress / 2.2) * 100))}%`;

  renderInventoryRange(document.getElementById('mc-furnace-player-main'), 9, 35);
  renderInventoryRange(document.getElementById('mc-furnace-player-hotbar'), 0, 8);
}

function renderExtendedUI() {
  renderMiniCraft();
  renderChestScreen();
  renderTableScreen();
  renderFurnaceScreen();
}

function showSpecialScreen(kind) {
  specialScreenOpen = kind;
  isInventoryOpen = true;
  inventoryScreen.style.display = 'none';
  menuOverlay.style.display = 'none';
  try { controls.unlock(); } catch (_) {}
  renderUI();
  updateCursorVisual();
}

function closeAllSpecialScreens(returnCursor = true) {
  if (!specialScreenOpen) return;

  document.getElementById('mc-chest-screen').style.display = 'none';
  document.getElementById('mc-table-screen').style.display = 'none';
  document.getElementById('mc-furnace-screen').style.display = 'none';

  specialScreenOpen = null;
  openChestKey = null;
  openTableKey = null;
  openFurnaceKey = null;

  if (returnCursor && cursorItem) {
    addToInventory(cursorItem.type, cursorItem.count);
    cursorItem = null;
  }
  updateCursorVisual();

  isInventoryOpen = false;
  menuOverlay.style.display = 'none';
  try { controls.lock(); } catch (_) {}
  renderUI();
}

function openChestAt(pos) {
  openChestKey = blockPosKey(pos.x, pos.y, pos.z);
  getChestStore(openChestKey);
  showSpecialScreen('chest');
}

function openTableAt(pos) {
  openTableKey = blockPosKey(pos.x, pos.y, pos.z);
  showSpecialScreen('table');
}

function openFurnaceAt(pos) {
  openFurnaceKey = blockPosKey(pos.x, pos.y, pos.z);
  getFurnaceStore(openFurnaceKey);
  showSpecialScreen('furnace');
}

function returnContainerItemsToInventory(items) {
  for (const item of items) {
    if (item) addToInventory(item.type, item.count);
  }
}

const _baseWorldSetBlock = world.setBlock.bind(world);
world.setBlock = (wx, wy, wz, type) => {
  const current = world.getBlock(wx, wy, wz);
  const key = blockPosKey(wx, wy, wz);

  if (type === 0) {
    if (current === 18 && chestStores.has(key)) {
      returnContainerItemsToInventory(getChestStore(key));
      chestStores.delete(key);
      if (openChestKey === key) closeAllSpecialScreens(true);
    }
    if (current === BLOCK_FURNACE && furnaceStores.has(key)) {
      const furnace = getFurnaceStore(key);
      returnContainerItemsToInventory([furnace.input, furnace.fuel, furnace.output]);
      furnaceStores.delete(key);
      if (openFurnaceKey === key) closeAllSpecialScreens(true);
    }
  }

  _baseWorldSetBlock(wx, wy, wz, type);
};

function tickFurnaces(dt) {
  for (const [key, furnace] of furnaceStores) {
    const outType = furnace.input ? FURNACE_OUTPUTS[furnace.input.type] : null;
    const canWork = !!(
      outType &&
      furnace.input &&
      furnace.fuel &&
      canAcceptFuel(furnace.fuel) &&
      (!furnace.output || (furnace.output.type === outType && furnace.output.count < 64))
    );

    if (!canWork) {
      furnace.progress = 0;
      continue;
    }

    furnace.progress += dt;
    if (furnace.progress < 2.2) continue;

    furnace.progress = 0;
    furnace.input.count -= 1;
    if (furnace.input.count <= 0) furnace.input = null;

    furnace.fuel.count -= 1;
    if (furnace.fuel.count <= 0) furnace.fuel = null;

    if (!furnace.output) furnace.output = { type: outType, count: 1 };
    else furnace.output.count += 1;

    if (specialScreenOpen === 'furnace' && openFurnaceKey === key) renderUI();
  }
}

const _baseUpdateDrops = updateDrops;
updateDrops = function(dt) {
  tickFurnaces(dt);
  _baseUpdateDrops(dt);
};

const _baseRenderUI = renderUI;
renderUI = function() {
  _baseRenderUI();
  renderExtendedUI();
};

const _baseToggleInventory = toggleInventory;
toggleInventory = function() {
  if (specialScreenOpen) {
    closeAllSpecialScreens(true);
    return;
  }
  _baseToggleInventory();
};

window.addEventListener('contextmenu', (e) => {
  e.preventDefault();
});

// Закрыть спец. окна по E / ESC до того, как сработает обычный контроллер
window.addEventListener('keydown', (e) => {
  if (!specialScreenOpen) return;
  if (e.code === 'KeyE' || e.code === 'Escape') {
    e.preventDefault();
    e.stopImmediatePropagation();
    closeAllSpecialScreens(true);
  }
}, true);

// ПКМ по сундуку / верстаку / печи + блокировка установки не-блоков
window.addEventListener('mousedown', (e) => {
  if (e.button !== 2) return;
  if (specialScreenOpen) return;
  if (!controls.isLocked || isInventoryOpen) return;

  if (highlightPos) {
    const hitType = world.getBlock(highlightPos.x, highlightPos.y, highlightPos.z);
    if (hitType === 18) {
      e.preventDefault();
      e.stopImmediatePropagation();
      openChestAt(highlightPos);
      return;
    }
    if (hitType === 17) {
      e.preventDefault();
      e.stopImmediatePropagation();
      openTableAt(highlightPos);
      return;
    }
    if (hitType === BLOCK_FURNACE) {
      e.preventDefault();
      e.stopImmediatePropagation();
      openFurnaceAt(highlightPos);
      return;
    }
  }

  const item = inventory[activeSlotIndex];
  if (item && !PLACEABLE_TYPES.has(item.type)) {
    e.preventDefault();
    e.stopImmediatePropagation();
  }
}, true);

makePixelUI();
renderUI();


// ===============================
// Firebase мультиплеер + чат
// ===============================

// TODO: ВСТАВЬ СВОЙ firebaseConfig ИЗ Firebase Console → Project settings → Your apps (Web)
const firebaseConfig = {
  apiKey: "AIzaSyDeEvXrWiDRY9ZDGDNUzrbuRTwSaL4IOxY",
  authDomain: "minecraftsss.firebaseapp.com",
  databaseURL: "https://minecraftsss-default-rtdb.firebaseio.com",
  projectId: "minecraftsss",
  storageBucket: "minecraftsss.firebasestorage.app",
  messagingSenderId: "1095636138018",
  appId: "1:1095636138018:web:a8485d7961c2e6bf0cf27b",
  measurementId: "G-TEZY9S3KEZ"
};

const netUI = {
  roomInput: document.getElementById('room-id'),
  nameInput: document.getElementById('player-name'),
  status: document.getElementById('net-status'),
  chatBox: document.getElementById('chat-box'),
  chatMessages: document.getElementById('chat-messages'),
  chatInputLine: document.getElementById('chat-input-line'),
  chatDraftSpan: document.getElementById('chat-draft'),
  nameStatus: document.getElementById('name-status'),
};


function appendChatLine(text, isSystem=false) {
  if (!netUI.chatMessages) return;
  const line = document.createElement('div');
  line.textContent = text;
  if (isSystem) line.className = 'sys';
  netUI.chatMessages.appendChild(line);
  netUI.chatMessages.scrollTop = netUI.chatMessages.scrollHeight;
}



const net = {
  enabled: false,
  roomId: '',
  playerId: (crypto && crypto.randomUUID) ? crypto.randomUUID() : (Math.random().toString(16).slice(2) + Date.now().toString(16)),
  name: 'Player',
  skin: 'steve',
  app: null,
  db: null,
  refs: {},
  lastSend: 0,
  remoteMeshes: new Map(),
  prevOnline: new Map(),
  prevNames: new Map(),
  playersInitialized: false,
  seenMods: new Set(),
  chatOpen: false,
  chatDraft: ''
};

function setNetStatus(text) {
  if (netUI.status) netUI.status.textContent = text;
}

function sanitizeRoomId(s) {
  return (s || '')
    .trim()
    .toLowerCase()
    .replace(/[^a-z0-9_-]/g, '')
    .slice(0, 32) || 'room';
}
function sanitizeName(s) {
  const t = (s || '').trim().slice(0, 16);
  return t || 'Player';
}

if (netUI.nameInput) {
  netUI.nameInput.addEventListener('input', () => setNameStatus(''));
}

async function isNicknameTaken(name) {
  try {
    ensureFirebaseInit();
    const playersRef = dbRef(net.db, 'rooms/global/players');
    const snap = await new Promise((resolve, reject) => onValue(playersRef, resolve, reject, { onlyOnce: true }));
    const players = snap && snap.val ? (snap.val() || {}) : {};
    const wanted = sanitizeName(name).toLowerCase();

    for (const [id, playerData] of Object.entries(players)) {
      if (id === net.playerId) continue;
      if (!playerData || playerData.online === false) continue;
      const currentName = sanitizeName(playerData.name || 'Player').toLowerCase();
      if (currentName === wanted) return true;
    }
  } catch (e) {
    console.warn('nickname check failed', e);
  }
  return false;
}


function chooseSkin(id) {
  // deterministic: one и тот же id -> один и тот же скин
  let h = 0;
  const s = String(id || '');
  for (let i = 0; i < s.length; i++) h = (h * 31 + s.charCodeAt(i)) >>> 0;
  return (h % 2 === 0) ? 'steve' : 'alex';
}

function createAvatar(kind) {
  const g = new THREE.Group();
  g.userData.kind = kind;

  // размеры (высота ~1.8), origin у ног
  const headH = 0.5, bodyH = 0.7, legH = 0.6, armH = 0.7;
  const headW = 0.5, bodyW = 0.5, bodyD = 0.25;
  const legW = 0.22, legD = 0.25;

  const isAlex = kind === 'alex';
  const armW = isAlex ? 0.16 : 0.20;
  const armD = 0.20;

  // материалы (без текстур, просто цвета)
  const skin = new THREE.MeshStandardMaterial({ color: 0xc9a065 });
  const steveShirt = new THREE.MeshStandardMaterial({ color: 0x2d6cdf });
  const stevePants = new THREE.MeshStandardMaterial({ color: 0x3b2a7a });
  const steveHair = new THREE.MeshStandardMaterial({ color: 0x3b2a1a });

  const alexShirt = new THREE.MeshStandardMaterial({ color: 0x3aa655 });
  const alexPants = new THREE.MeshStandardMaterial({ color: 0x6b4f2a });
  const alexHair = new THREE.MeshStandardMaterial({ color: 0xd06a2a });

  const shirtMat = isAlex ? alexShirt : steveShirt;
  const pantsMat = isAlex ? alexPants : stevePants;
  const hairMat = isAlex ? alexHair : steveHair;

  // голова
  const head = new THREE.Mesh(new THREE.BoxGeometry(headW, headH, headW), skin);
  head.position.set(0, legH + bodyH + headH/2, 0);
  head.castShadow = true;
  g.add(head);

  // "волосы" как тонкая "шапка" сверху головы (чтобы не влезали внутрь)
  const hairH = headH * 0.22;
  const hair = new THREE.Mesh(new THREE.BoxGeometry(headW*1.04, hairH, headW*1.04), hairMat);
  // ставим кап на самый верх головы + маленький зазор
  hair.position.set(0, legH + bodyH + headH - hairH/2 + 0.01, 0);
  hair.castShadow = true;
  g.add(hair);

  // тело
  const body = new THREE.Mesh(new THREE.BoxGeometry(bodyW, bodyH, bodyD), shirtMat);
  body.position.set(0, legH + bodyH/2, 0);
  body.castShadow = true;
  g.add(body);

  // ноги
  const leg1 = new THREE.Mesh(new THREE.BoxGeometry(legW, legH, legD), pantsMat);
  const leg2 = new THREE.Mesh(new THREE.BoxGeometry(legW, legH, legD), pantsMat);
  leg1.position.set(-0.12, legH/2, 0);
  leg2.position.set( 0.12, legH/2, 0);
  leg1.castShadow = leg2.castShadow = true;
  g.add(leg1, leg2);

  // руки
  const arm1 = new THREE.Mesh(new THREE.BoxGeometry(armW, armH, armD), shirtMat);
  const arm2 = new THREE.Mesh(new THREE.BoxGeometry(armW, armH, armD), shirtMat);
  arm1.position.set(-(bodyW/2 + armW/2 + 0.02), legH + bodyH - armH/2, 0);
  arm2.position.set( (bodyW/2 + armW/2 + 0.02), legH + bodyH - armH/2, 0);
  arm1.castShadow = arm2.castShadow = true;
  g.add(arm1, arm2);


  // сохранить части для анимации
  g.userData.parts = { head, body, leg1, leg2, arm1, arm2 };
  g.userData.anim = { speed: 0 };
  return g;
}


function animateAvatar(avatar, speed, tSec) {
  if (!avatar || !avatar.userData || !avatar.userData.parts) return;
  const { leg1, leg2, arm1, arm2, head } = avatar.userData.parts;
  const s = Math.min(1, Math.max(0, speed / 3)); // 0..1
  const swing = Math.sin(tSec * 10) * 0.9 * s;
  const swing2 = Math.sin(tSec * 10 + Math.PI) * 0.9 * s;

  // ноги/руки (простая "майнкрафт" походка)
  leg1.rotation.x = swing;
  leg2.rotation.x = swing2;
  arm1.rotation.x = swing2 * 0.85;
  arm2.rotation.x = swing * 0.85;

  // лёгкая "подпрыгивающая" походка (без накопления)
  const baseY = (avatar.userData.baseY !== undefined) ? avatar.userData.baseY : avatar.position.y;
  avatar.position.y = baseY + Math.sin(tSec * 20) * 0.05 * s;

  // голова чуть качается
  head.rotation.y = Math.sin(tSec * 3) * 0.1 * s;
}

// --- Inventory skin preview (мини-рендер на canvas с чёрным фоном) ---
const invSkinCanvas = document.getElementById('inv-skin-canvas');
let invSkinRenderer = null;
let invSkinScene = null;
let invSkinCam = null;
let invSkinAvatar = null;
let invSkinKind = null;

function ensureInvPreview() {
  if (!invSkinCanvas || invSkinRenderer) return;
  invSkinRenderer = new THREE.WebGLRenderer({ canvas: invSkinCanvas, antialias: false, alpha: false, preserveDrawingBuffer: true });
  invSkinRenderer.setPixelRatio(Math.min(window.devicePixelRatio || 1, 2));
  invSkinRenderer.setSize(invSkinCanvas.width, invSkinCanvas.height, false);
  invSkinRenderer.setClearColor(0x000000, 1);

  invSkinScene = new THREE.Scene();
  const amb = new THREE.AmbientLight(0xffffff, 0.9);
  invSkinScene.add(amb);
  const dl = new THREE.DirectionalLight(0xffffff, 1.2);
  dl.position.set(2, 3, 2);
  invSkinScene.add(dl);

  invSkinCam = new THREE.PerspectiveCamera(40, invSkinCanvas.width / invSkinCanvas.height, 0.1, 100);
  invSkinCam.position.set(1.8, 1.4, 2.6);
  invSkinCam.lookAt(0, 1.1, 0);
}

function updateInvPreview(tSec) {
  if (!isInventoryOpen) return;
  ensureInvPreview();
  if (!invSkinRenderer) return;

  const kind = (net && net.skin) ? net.skin : 'steve';
  if (!invSkinAvatar || invSkinKind !== kind) {
    if (invSkinAvatar) { invSkinScene.remove(invSkinAvatar); disposeObject3D(invSkinAvatar); }
    invSkinAvatar = createAvatar(kind);
    invSkinKind = kind;
    invSkinAvatar.position.set(0, 0, 0);
    invSkinScene.add(invSkinAvatar);
  }

  invSkinAvatar.rotation.y = tSec * 0.8;
  animateAvatar(invSkinAvatar, 0, tSec); // idle

  invSkinRenderer.render(invSkinScene, invSkinCam);
}

// Никнейм над головой (canvas -> sprite)
function makeNameTag(name) {
  const canvas = document.createElement('canvas');
  canvas.width = 256;
  canvas.height = 64;
  const ctx = canvas.getContext('2d');
  ctx.clearRect(0, 0, canvas.width, canvas.height);

  const safe = (name || 'Player').toString().slice(0, 16);

  // фон полупрозрачный
  ctx.fillStyle = 'rgba(0,0,0,0.45)';
  roundRect(ctx, 8, 10, canvas.width - 16, canvas.height - 20, 10);
  ctx.fill();

  // текст с обводкой (как в Minecraft UI)
  ctx.font = '20px "Press Start 2P", monospace';
  ctx.textAlign = 'center';
  ctx.textBaseline = 'middle';

  ctx.lineWidth = 4;
  ctx.strokeStyle = 'rgba(0,0,0,0.95)';
  ctx.strokeText(safe, canvas.width / 2, canvas.height / 2);

  ctx.fillStyle = '#ffffff';
  ctx.fillText(safe, canvas.width / 2, canvas.height / 2);

  const tex = new THREE.CanvasTexture(canvas);
  tex.minFilter = THREE.LinearFilter;
  tex.magFilter = THREE.LinearFilter;

  const mat = new THREE.SpriteMaterial({
    map: tex,
    transparent: true,
    depthTest: false,   // чтобы не пропадал за блоками (можешь поменять на true, если хочешь иначе)
    depthWrite: false
  });

  const sprite = new THREE.Sprite(mat);
  sprite.renderOrder = 999; // рисовать поверх
  sprite.scale.set(1.6, 0.4, 1); // размер таблички
  sprite.userData._nameCanvas = canvas;
  sprite.userData._nameCtx = ctx;
  sprite.userData._nameValue = safe;
  return sprite;
}

function updateNameTag(sprite, name) {
  if (!sprite) return;
  const safe = (name || 'Player').toString().slice(0, 16);
  if (sprite.userData._nameValue === safe) return;

  const canvas = sprite.userData._nameCanvas;
  const ctx = sprite.userData._nameCtx;
  if (!canvas || !ctx) return;

  ctx.clearRect(0, 0, canvas.width, canvas.height);

  ctx.fillStyle = 'rgba(0,0,0,0.45)';
  roundRect(ctx, 8, 10, canvas.width - 16, canvas.height - 20, 10);
  ctx.fill();

  ctx.font = '20px "Press Start 2P", monospace';
  ctx.textAlign = 'center';
  ctx.textBaseline = 'middle';

  ctx.lineWidth = 4;
  ctx.strokeStyle = 'rgba(0,0,0,0.95)';
  ctx.strokeText(safe, canvas.width / 2, canvas.height / 2);

  ctx.fillStyle = '#ffffff';
  ctx.fillText(safe, canvas.width / 2, canvas.height / 2);

  sprite.material.map.needsUpdate = true;
  sprite.userData._nameValue = safe;
}

function roundRect(ctx, x, y, w, h, r) {
  const rr = Math.min(r, w / 2, h / 2);
  ctx.beginPath();
  ctx.moveTo(x + rr, y);
  ctx.arcTo(x + w, y, x + w, y + h, rr);
  ctx.arcTo(x + w, y + h, x, y + h, rr);
  ctx.arcTo(x, y + h, x, y, rr);
  ctx.arcTo(x, y, x + w, y, rr);
  ctx.closePath();
}

function disposeObject3D(obj) {
  obj.traverse((child) => {
    if (child.geometry) child.geometry.dispose();
    if (child.material) {
      const mats = Array.isArray(child.material) ? child.material : [child.material];
      for (const m of mats) {
        if (m.map) m.map.dispose();
        if (m.alphaMap) m.alphaMap.dispose();
        m.dispose();
      }
    }
  });
}

function ensureFirebaseInit() {
  if (net.app) return;
  net.app = initializeApp(firebaseConfig);
  net.db = getDatabase(net.app);
}

async function connectRoom() {
  try {
    ensureFirebaseInit();

    net.roomId = 'global'; // один общий мир для всех
    
    net.name = sanitizeName(netUI.nameInput?.value || 'Player');
    net.skin = chooseSkin(net.playerId);

    // Создаём локального аватара (Steve/Alex) — он будет виден только в 3-м лице
    if (!localAvatar) {
      localAvatar = createAvatar(net.skin);
      localAvatar.position.set(player.pos.x, player.pos.y, player.pos.z);

      // ник над головой и себе тоже (приятно в фронт-камере)
      localAvatarTag = makeNameTag(net.name);
      localAvatarTag.position.set(0, 2.15, 0);
      localAvatar.add(localAvatarTag);

      localAvatar.visible = false; // по умолчанию первое лицо
      scene.add(localAvatar);
    } else {
      // если аватар уже есть — обновим ник/скин
      // (проще пересоздать при перезаходе)
    }

    const base = `rooms/${net.roomId}`;
    net.refs.players = dbRef(net.db, `${base}/players`);
    net.refs.me = dbRef(net.db, `${base}/players/${net.playerId}`);
    net.refs.chat = dbRef(net.db, `${base}/chat`);
    net.refs.mods = dbRef(net.db, `${base}/mods`);

// Общий seed мира (чтобы ни у кого не было разного ландшафта и никто не "проваливался")
const seedRef = dbRef(net.db, `${base}/seed`);
const tx = await runTransaction(seedRef, (current) => {
  if (current === null || current === undefined) return DEFAULT_WORLD_SEED;
  return current;
});
const remoteSeed = (tx && tx.snapshot) ? tx.snapshot.val() : DEFAULT_WORLD_SEED;

if ((remoteSeed >>> 0) !== (worldSeed >>> 0)) {
  // если seed в базе другой — пересобираем мир
  setWorldSeed(remoteSeed >>> 0);
  // сброс чанков
  for (const [k, ch] of chunks) { ch.dispose(); }
  chunks.clear();
  savedChunkMods.clear();
  net.seenMods.clear();
  // загрузить чанки вокруг игрока
  world.update(player.pos.x, player.pos.z);
  // поднять игрока на поверхность, если оказался внутри/ниже земли
  const safeY = getSurfaceY(player.pos.x, player.pos.z);
  player.pos.y = Math.max(player.pos.y, safeY + PLAYER_HEIGHT + 0.2);
  camera.position.copy(player.pos);
}

    // Presence / self node
    await dbSet(net.refs.me, {
      name: net.name,
      skin: net.skin,
      x: player.pos.x, y: player.pos.y, z: player.pos.z,
      online: true,
      t: serverTimestamp()
    });
    // При внезапном закрытии вкладки помечаем игрока как offline (не удаляем сразу, чтобы другие могли показать сообщение)
    onDisconnect(net.refs.me).update({ online: false, leftAt: serverTimestamp(), t: serverTimestamp() });

    // Best-effort: при обычном выходе тоже пометить offline
    const markOffline = () => {
      try { dbUpdate(net.refs.me, { online: false, leftAt: serverTimestamp(), t: serverTimestamp() }); } catch(e) {}
    };
    window.addEventListener('pagehide', markOffline);
    window.addEventListener('beforeunload', markOffline);

    // Listen players (including join/leave)
    onValue(net.refs.players, (snap) => {
      if (!net.enabled) return;
      const data = snap.val() || {};
      const alive = new Set(Object.keys(data));

      // Флаг "первой загрузки", чтобы не спамить "вошёл" на всех уже онлайн
      const initial = !net.playersInitialized;

      for (const [id, obj] of Object.entries(data)) {
        if (id === net.playerId) continue;
        const p = obj || {};
        const name = (p.name || net.prevNames.get(id) || 'Player').toString().slice(0, 16);
        const onlineNow = (p.online !== false);

        const onlinePrev = net.prevOnline.get(id);

        // запоминаем имя/статус
        net.prevNames.set(id, name);
        net.prevOnline.set(id, onlineNow);

        // системные сообщения (локально в чат для всех клиентов)
        if (!initial) {
          if (onlinePrev === undefined && onlineNow) appendChatLine(`* ${name} вошёл`, true);
          if (onlinePrev === true && !onlineNow) appendChatLine(`* ${name} вышел`, true);
          if (onlinePrev === false && onlineNow) appendChatLine(`* ${name} вошёл`, true);
        }

        // оффлайн игрока не рисуем
        if (!onlineNow) {
          const existing = net.remoteMeshes.get(id);
          if (existing) {
            scene.remove(existing.avatar);
            disposeObject3D(existing.avatar);
            net.remoteMeshes.delete(id);
          }
          continue;
        }

        // ---- Рендер удалённых игроков ----
        let item = net.remoteMeshes.get(id);
        if (!item) {
          const kind = (p.skin === 'alex' || p.skin === 'steve') ? p.skin : chooseSkin(id);
          const avatar = createAvatar(kind);
          avatar.position.set(p.x || 0, (p.y || 0), p.z || 0); // p.y = позиция ног (feet)

          const tag = makeNameTag(name);
          tag.position.set(0, 2.15, 0);
          avatar.add(tag);

          scene.add(avatar);
          item = { avatar, kind, tag, name, lastPos: new THREE.Vector3(p.x||0, p.y||0, p.z||0), lastAt: performance.now(), speed: 0 };
          net.remoteMeshes.set(id, item);
        }

        // обновление позиции, скорости и ника
        const now = performance.now();
        const nx = (p.x || 0), ny = (p.y || 0), nz = (p.z || 0);
        const dtms = Math.max(1, now - (item.lastAt || now));
        const dist = Math.hypot(nx - (item.lastPos?.x || nx), nz - (item.lastPos?.z || nz)); // по земле
        const sp = dist / (dtms / 1000);
        item.speed = item.speed * 0.7 + sp * 0.3;
        item.lastAt = now;
        if (item.lastPos) item.lastPos.set(nx, ny, nz);

        item.avatar.position.set(nx, ny, nz);
        updateNameTag(item.tag, name);
      }

      // После первой синхронизации считаем, что "начальный снимок" пройден
      if (initial) net.playersInitialized = true;

      // Удаляем тех, кого больше нет в data
      for (const [id, item] of net.remoteMeshes) {
        if (!alive.has(id)) {
          scene.remove(item.avatar);
          disposeObject3D(item.avatar);
          net.remoteMeshes.delete(id);
          net.prevOnline.delete(id);
          net.prevNames.delete(id);
        }
      }
    });

    // Listen chat
    const chatQ = dbQuery(net.refs.chat, limitToLast(40));
    onChildAdded(chatQ, (snap) => {
      if (!net.enabled) return;
      const m = snap.val();
      if (!m || !netUI.chatMessages) return;
      const line = document.createElement('div');
      const name = (m.name || 'Player').toString().slice(0, 16);
      const text = (m.text || '').toString().slice(0, 200);
      line.textContent = `[${name}] ${text}`;
      netUI.chatMessages.appendChild(line);
      netUI.chatMessages.scrollTop = netUI.chatMessages.scrollHeight;
    });

    // Listen mods (block changes)
    const modsQ = dbQuery(net.refs.mods, limitToLast(2000));
    onChildAdded(modsQ, (snap) => {
      if (!net.enabled) return;
      const key = snap.key;
      if (!key || net.seenMods.has(key)) return;
      net.seenMods.add(key);

      const m = snap.val();
      if (!m) return;
      if (m.by === net.playerId) return; // ignore own echo

      // apply without drops
      applyBlockNoDrop(m.wx, m.wy, m.wz, m.type);
    });

    net.enabled = true;
    setNetStatus(`ONLINE: ${net.roomId}`);
  } catch (e) {
    console.error(e);
    setNetStatus('OFFLINE (ошибка FirebaseConfig или Rules)');
  }
}

// Apply block changes like world.setBlock, but WITHOUT spawning drops (для удалённых игроков)
function applyBlockNoDrop(wx, wy, wz, type) {
  const cx = Math.floor(wx / CHUNK_SIZE);
  const cz = Math.floor(wz / CHUNK_SIZE);
  const lx = Math.floor(wx) - cx * CHUNK_SIZE;
  const ly = Math.floor(wy);
  const lz = Math.floor(wz) - cz * CHUNK_SIZE;

  const key = getChunkKey(cx, cz);
  if (!savedChunkMods.has(key)) savedChunkMods.set(key, []);
  const mods = savedChunkMods.get(key);
  const existIdx = mods.findIndex(m => m.x === lx && m.y === ly && m.z === lz);
  if (existIdx !== -1) mods.splice(existIdx, 1);
  mods.push({ x: lx, y: ly, z: lz, type });

  const chunk = chunks.get(key);
  if (chunk) {
    chunk.setBlockLocal(lx, ly, lz, type);
    chunk.buildMesh();
  }
}

// Wrap local world.setBlock to broadcast mods
(function hookWorldSetBlock() {
  const original = world.setBlock;
  world.setBlock = (wx, wy, wz, type) => {
    original(wx, wy, wz, type);

    if (!net.enabled) return;
    // Broadcast
    try {
      dbPush(net.refs.mods, {
        wx: Math.floor(wx),
        wy: Math.floor(wy),
        wz: Math.floor(wz),
        type: type,
        by: net.playerId,
        t: serverTimestamp()
      });
    } catch (e) {
      console.warn('mod push failed', e);
    }
  };
})();

// Send player state throttled
function netTick(dt) {
  if (!net.enabled) return;
  net.lastSend += dt;
  if (net.lastSend < 0.1) return; // 10 раз/сек
  net.lastSend = 0;

  try {
    dbUpdate(net.refs.me, {
      name: net.name,
      skin: net.skin,
      x: player.pos.x,
      y: player.pos.y,
      z: player.pos.z,
      online: true,
      t: serverTimestamp()
    });
  } catch (e) {
    // ignore
  }
}

// Connect on "ИГРАТЬ"
resumeBtn.addEventListener('click', async () => { 
    playSound('pickup');

  const desiredName = sanitizeName(netUI.nameInput ? netUI.nameInput.value : 'Player');
  if (netUI.nameInput) netUI.nameInput.value = desiredName;
  setNameStatus('');

  if (!net.enabled) {
    setNetStatus('ПРОВЕРКА НИКА...');
    const taken = await isNicknameTaken(desiredName);
    if (taken) {
      setNameStatus('Никнейм занят', true);
      setNetStatus('Никнейм занят');
      menuOverlay.style.display = 'flex';
      return;
    }
  }

  if (net.enabled && desiredName !== net.name) {
    net.name = desiredName;
    if (localAvatarTag) updateNameTag(localAvatarTag, desiredName);
    try { dbUpdate(net.refs.me, { name: desiredName, t: serverTimestamp() }); } catch (_) {}
  }

  menuOverlay.style.display = 'none';
  try { if (!isInventoryOpen) controls.lock(); } catch(e) {}

  if (!net.enabled) {
    setNetStatus('ПОДКЛЮЧЕНИЕ...');
    await connectRoom();
    if (!net.enabled) {
      menuOverlay.style.display = 'flex';
      return;
    }
  }

  setTimeout(() => {
    if (!controls.isLocked && !isTouchDevice) {
      setNetStatus('Нажми по игре, чтобы захватить мышь');
    }
  }, 400);
});
// Одиночная игра (без Firebase)
singleBtn?.addEventListener('click', () => {
  setNameStatus('');
  resetLocalWorldToClean(true);
  setNetStatus('SINGLEPLAYER');
  menuOverlay.style.display = 'none';
  try { controls.lock(); } catch(e) {}
});

// --- CHAT (без <input>, чтобы PointerLock не ломался) ---
function chatOpen() {
  net.chatOpen = true;
  net.chatDraft = '';
  if (netUI.chatBox) netUI.chatBox.style.display = 'block';
  if (netUI.chatInputLine) netUI.chatInputLine.style.display = 'block';
  if (netUI.chatDraftSpan) netUI.chatDraftSpan.textContent = '';
}
function chatClose() {
  net.chatOpen = false;
  net.chatDraft = '';
  if (netUI.chatInputLine) netUI.chatInputLine.style.display = 'none';
  if (netUI.chatDraftSpan) netUI.chatDraftSpan.textContent = '';
}
function chatSend() {
  const text = (net.chatDraft || '').trim();
  if (!text) return;

  // Команда (не отправляется в чат)
  if (text.toLowerCase() === '/gamemodeezes') {
    creativeUnlocked = true;
    flyMode = true;
    // показать/скрыть кнопку "вниз" на мобилках
    const downBtn = document.querySelector('#mobile-controls .m-flydown');
    if (downBtn) downBtn.style.display = 'inline-flex';
    sysChatLine('* Включён творческий режим: полёт');
    return;
  }

    // Любая другая команда (не отправляем и не палим правильную)
  if (text.startsWith('/')) {
    // Можно вообще молчать, но пусть будет нейтрально:
    sysChatLine('* ...');
    return;
  }

if (!net.enabled) {
    // одиночка: показываем локально
    sysChatLine(`[${net.name}] ${text}`);
    return;
  }

  dbPush(net.refs.chat, {
    name: net.name,
    text,
    t: serverTimestamp()
  });
}

function chatAppendChar(ch) {
  if (net.chatDraft.length >= 120) return;
  net.chatDraft += ch;
  if (netUI.chatDraftSpan) netUI.chatDraftSpan.textContent = net.chatDraft;
}
function chatBackspace() {
  net.chatDraft = net.chatDraft.slice(0, -1);
  if (netUI.chatDraftSpan) netUI.chatDraftSpan.textContent = net.chatDraft;
}

// Capture keydown BEFORE game controls
document.addEventListener('keydown', (e) => {
  // Open chat
  if (!net.chatOpen && e.code === 'KeyT' && controls.isLocked && !isInventoryOpen) {
    e.preventDefault();
    e.stopImmediatePropagation();
    chatOpen();
    return;
  }

  if (!net.chatOpen) return;

  // While chat open, block game controls
  e.preventDefault();
  e.stopImmediatePropagation();

  if (e.code === 'Escape') {
    chatClose();
    return;
  }
  if (e.code === 'Enter') {
    chatSend();
    chatClose();
    return;
  }
  if (e.code === 'Backspace') {
    chatBackspace();
    return;
  }
  if (e.code === 'Space') {
    chatAppendChar(' ');
    return;
  }

  // Printable characters
  if (!e.ctrlKey && !e.metaKey && !e.altKey && typeof e.key === 'string' && e.key.length === 1) {
    chatAppendChar(e.key);
  }
}, true);

animate();

window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    viewCamera.aspect = window.innerWidth / window.innerHeight;
    viewCamera.updateProjectionMatrix();
    handCamera.aspect = window.innerWidth / window.innerHeight;
    handCamera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
});

</script>
</body>
</html>

run.bat
@echo off
echo Запуск игры на локальном сервере...
cd /d "%~dp0"
start "" python -m http.server 8000
timeout /t 2 /nobreak
start http://localhost:8000

style.css
body {
    margin: 0;
    overflow: hidden;
    font-family: 'Courier New', Courier, monospace;
    user-select: none;
}

#crosshair {
    position: absolute;
    top: 50%;
    left: 50%;
    width: 20px;
    height: 20px;
    margin-top: -10px;
    margin-left: -10px;
    text-align: center;
    line-height: 20px;
    color: white;
    font-size: 20px;
    pointer-events: none;
    z-index: 10;
    text-shadow: 1px 1px 0 #000;
}

#menu-overlay {
    position: absolute;
    width: 100%;
    height: 100%;
    background-color: rgba(0,0,0,0.7);
    display: flex;
    justify-content: center;
    align-items: center;
    z-index: 20;
    backdrop-filter: blur(5px);
}

#menu-box {
    background: rgba(0, 0, 0, 0.8);
    padding: 40px;
    border: 4px solid #fff;
    border-radius: 10px;
    text-align: center;
    color: white;
    width: 400px;
    box-shadow: 0 0 20px rgba(0,0,0,0.5);
}

#menu-box h1 {
    margin-top: 0;
    text-shadow: 2px 2px 0 #555;
    font-size: 32px;
}

#resume-btn {
    padding: 15px 30px;
    font-size: 20px;
    font-family: inherit;
    cursor: pointer;
    background: #4caf50;
    color: white;
    border: none;
    border-radius: 5px;
    margin-bottom: 20px;
    width: 100%;
    transition: background 0.2s;
}

#resume-btn:hover {
    background: #45a049;
}

.settings-row {
    margin: 20px 0;
    display: flex;
    flex-direction: column;
    gap: 10px;
    align-items: center;
}

.controls-info {
    margin-top: 20px;
    font-size: 14px;
    color: #ccc;
    border-top: 1px solid #555;
    padding-top: 10px;
}

#hotbar {
    position: absolute;
    bottom: 10px;
    left: 50%;
    transform: translateX(-50%);
    display: flex;
    background: rgba(0, 0, 0, 0.5);
    padding: 5px;
    border-radius: 5px;
    gap: 5px;
    z-index: 10;
}

.slot {
    width: 50px;
    height: 50px;
    border: 2px solid #555;
    background: rgba(0,0,0,0.3);
    display: flex;
    justify-content: center;
    align-items: center;
    position: relative;
}

.slot img {
    width: 90%;
    height: 90%;
    image-rendering: pixelated; 
}

.slot.active {
    border: 3px solid white;
    background: rgba(255,255,255,0.2);
    transform: scale(1.1);
}

.item-count {
    position: absolute;
    bottom: 2px;
    right: 2px;
    font-size: 14px;
    color: white;
    text-shadow: 1px 1px 0 #000;
}

#inventory-screen {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: rgba(0,0,0,0.7);
    display: flex;
    justify-content: center;
    align-items: center;
    z-index: 15;
}

#inventory-container {
    background: #c6c6c6;
    padding: 20px;
    border-radius: 5px;
    border: 4px solid #555;
    color: #333;
}

#inventory-grid {
    display: grid;
    grid-template-columns: repeat(9, 1fr);
    gap: 5px;
    margin-top: 10px;
}

#inventory-hotbar-grid {
    display: grid;
    grid-template-columns: repeat(9, 1fr);
    gap: 5px;
}

#inventory-grid .slot, #inventory-hotbar-grid .slot {
    background: #8b8b8b;
    border: 2px solid #fff;
    border-right-color: #373737;
    border-bottom-color: #373737;
}

#inventory-grid .slot:hover, #inventory-hotbar-grid .slot:hover {
    background: #a0a0a0;
}

И Папка Assets где хранятся все фото звуки
