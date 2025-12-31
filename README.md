<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Shadow Chase WebGL</title>
<style>body { margin:0; overflow:hidden; }</style>
</head>
<body>
<script src="js/three.min.js"></script>
<script src="js/STLLoader.js"></script>

<script>
// =======================
// Scene Setup
// =======================
let scene = new THREE.Scene();
let camera = new THREE.PerspectiveCamera(75, window.innerWidth/window.innerHeight, 0.1, 1000);
let renderer = new THREE.WebGLRenderer({antialias:true});
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

// Lighting
let ambientLight = new THREE.AmbientLight(0xffffff, 0.6);
scene.add(ambientLight);
let pointLight = new THREE.PointLight(0xffffff, 0.8);
pointLight.position.set(0,10,10);
scene.add(pointLight);

// =======================
// Load STL Models
// =======================
let loader = new THREE.STLLoader();
let player, shadow, orb;
let orbs = [];

// Player variables
let playerSpeed = 0.15;
let dashSpeed = 0.3;
let jumpSpeed = 0.3;
let velocityY = 0;
let gravity = -0.01;
let isGrounded = true;

// Shadow AI
let shadowBaseSpeed = 0.02;
let shadowState = "stalking"; // stalking, flanking, crawling, vanished, hunting

// Game state
let score = 0;
let gameOver = false;

// Controls
let keys = {};
window.addEventListener('keydown', e=>keys[e.key]=true);
window.addEventListener('keyup', e=>keys[e.key]=false);

// =======================
// Load Player
// =======================
loader.load('models/player.stl', g=>{
    let mat = new THREE.MeshNormalMaterial();
    player = new THREE.Mesh(g, mat);
    player.position.set(0,0,0);
    scene.add(player);
});

// Load Shadow
loader.load('models/shadow.stl', g=>{
    let mat = new THREE.MeshBasicMaterial({color:0x000000});
    shadow = new THREE.Mesh(g, mat);
    shadow.position.set(0,0,-5);
    scene.add(shadow);
});

// Load Initial Orb
loader.load('models/orb.stl', g=>{
    let mat = new THREE.MeshBasicMaterial({color:0xffff00});
    orb = new THREE.Mesh(g, mat);
    orb.position.set(2,0,-2);
    scene.add(orb);
    orbs.push(orb);
});

// Camera
camera.position.set(0,5,10);
camera.lookAt(0,0,0);

// =======================
// Orb Spawning
// =======================
setInterval(()=>{
    if(gameOver || !player) return;
    loader.load('models/orb.stl', g=>{
        let mat = new THREE.MeshBasicMaterial({color:0xffff00});
        let newOrb = new THREE.Mesh(g, mat);
        newOrb.position.set(Math.random()*8-4,0,Math.random()*-8);
        scene.add(newOrb);
        orbs.push(newOrb);
    });
}, 5000);

// =======================
// Shadow AI Decision Maker
// =======================
setInterval(()=>{
    if(!shadow || !player || gameOver) return;
    let dist = shadow.position.distanceTo(player.position);
    if(dist > 15) shadowState = "hunting";
    else if(dist < 5) shadowState = "vanished";
    else if(dist < 10) shadowState = "crawling";
    else shadowState = Math.random()>0.6 ? "stalking" : "flanking";
}, 2000 + Math.random()*3000);

// =======================
// Animation Loop
// =======================
function animate(){
    requestAnimationFrame(animate);
    if(!player) return;

    // --- Player Movement ---
    let moveX=0, moveZ=0;
    if(keys['a']) moveX -= playerSpeed;
    if(keys['d']) moveX += playerSpeed;
    if(keys['w']) moveZ -= playerSpeed;
    if(keys['s']) moveZ += playerSpeed;

    // Jump
    if(keys[' '] && isGrounded){
        velocityY = jumpSpeed;
        isGrounded = false;
    }

    // Dash
    if(keys['Shift']){
        moveX *= dashSpeed/playerSpeed;
        moveZ *= dashSpeed/playerSpeed;
    }

    // Gravity
    if(!isGrounded){
        velocityY += gravity;
        player.position.y += velocityY;
        if(player.position.y <= 0){
            player.position.y = 0;
            isGrounded = true;
            velocityY = 0;
        }
    }

    player.position.x += moveX;
    player.position.z += moveZ;

    // --- Shadow AI Movement ---
    if(shadow){
        let targetPos = player.position.clone();
        switch(shadowState){
            case "stalking":
                targetPos.add(player.getWorldDirection(new THREE.Vector3()).multiplyScalar(-3));
                break;
            case "flanking":
                targetPos.add(new THREE.Vector3(Math.random()>0.5?3:-3,0,0));
                break;
            case "crawling":
                targetPos.add(new THREE.Vector3(0,0,0));
                break;
            case "vanished":
                shadow.position.set(0,-100,0);
                break;
            case "hunting":
                break;
        }
        if(shadowState!="vanished"){
            let dir = new THREE.Vector3().subVectors(targetPos, shadow.position).normalize();
            shadow.position.add(dir.multiplyScalar(shadowBaseSpeed + score*0.0005));
        }
    }

    // --- Collision Detection ---
    if(shadow && player.position.distanceTo(shadow.position)<0.5){
        gameOver = true;
        alert("GAME OVER! Score: "+Math.floor(score));
    }

    orbs.forEach((o,i)=>{
        if(player.position.distanceTo(o.position)<0.5){
            scene.remove(o);
            orbs.splice(i,1);
            score += 5;
        }
    });

    score += 0.02; // survival points

    renderer.render(scene, camera);
}
animate();
</script>
</body>
</html>
