<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1,user-scalable=no">
<meta name="theme-color" content="#07011a">
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-title" content="Stress Reliever">
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
<link rel="manifest" href="data:application/manifest+json,{&quot;name&quot;:&quot;Cheyenne N. Pilling's - Stress Reliever&quot;,&quot;short_name&quot;:&quot;Stress Reliever&quot;,&quot;start_url&quot;:&quot;.&quot;,&quot;display&quot;:&quot;standalone&quot;,&quot;background_color&quot;:&quot;#07011a&quot;,&quot;theme_color&quot;:&quot;#07011a&quot;}">
<title>Cheyenne N. Pilling's - Stress Reliever</title>
<style>
  html,body{margin:0;height:100%;background:#07011a;overflow:hidden;touch-action:none;font-family:system-ui,-apple-system,Roboto,sans-serif;-webkit-user-select:none;user-select:none}
  canvas{display:block;position:fixed;inset:0}
  .glass{background:rgba(12,16,34,.55);backdrop-filter:blur(10px);-webkit-backdrop-filter:blur(10px);border:1px solid #ff2bd6;border-radius:16px;color:#fff;box-shadow:0 0 14px rgba(255,43,214,.55),inset 0 0 10px rgba(0,240,255,.18)}
  #hud{position:fixed;top:12px;left:12px;right:12px;display:flex;align-items:center;gap:14px;pointer-events:none}
  #score{font-weight:800;font-size:22px;padding:8px 14px;color:#00f0ff;text-shadow:0 0 8px #00f0ff,0 0 18px #00f0ff}
  #shots{display:flex;gap:6px;padding:8px 12px}
  .dot{width:14px;height:14px;border-radius:50%;background:linear-gradient(135deg,#caf0f8,#4cc9f0);box-shadow:0 0 8px #4cc9f0}
  #level{margin-left:auto;padding:8px 14px;font-size:15px;opacity:.9}
  #mute{pointer-events:auto;cursor:pointer;font-size:18px;padding:6px 12px;background:rgba(12,16,34,.55);color:#fff;border:1px solid rgba(255,255,255,.12);border-radius:12px}
  #hint{position:fixed;bottom:16px;left:0;right:0;text-align:center;color:#ff7af0;text-shadow:0 0 10px #ff2bd6;font-size:15px;pointer-events:none;transition:opacity .3s}
  #overlay{position:fixed;inset:0;display:flex;align-items:center;justify-content:center;background:rgba(5,8,20,.55);backdrop-filter:blur(4px);-webkit-backdrop-filter:blur(4px)}
  #overlay.hidden{display:none}
  .card{padding:32px 36px;max-width:460px;text-align:center}
  #title{margin:0 0 10px;font-size:clamp(34px,8vw,58px);font-weight:900;background:linear-gradient(90deg,#ff2bd6,#00f0ff,#b8ff00);filter:drop-shadow(0 0 12px #ff2bd6);-webkit-background-clip:text;background-clip:text;color:transparent}
  #text{white-space:pre-line;opacity:.9;font-size:18px;line-height:1.5;margin:0 0 22px}
  #btn{font-size:20px;font-weight:800;padding:12px 44px;border:2px solid #00f0ff;border-radius:14px;background:rgba(0,240,255,.08);color:#00f0ff;cursor:pointer;box-shadow:0 0 14px #00f0ff,inset 0 0 10px rgba(0,240,255,.4);text-shadow:0 0 8px #00f0ff}
  #btn:active{transform:translateY(3px);box-shadow:none}
  .pop{position:fixed;transform:translate(-50%,-50%);font-weight:900;font-size:24px;color:#ffd166;text-shadow:0 2px 8px #000;pointer-events:none;animation:rise .9s ease-out forwards}
  @keyframes rise{to{transform:translate(-50%,-150%);opacity:0}}
</style>
</head>
<body>

<div id="hud" class="hidden">
  <div id="score" class="glass">Score 0</div>
  <div id="shots" class="glass"></div>
  <div id="level" class="glass"></div>
  <button id="mute" title="Mute (M)">🔊</button>
</div>
<div id="hint" class="hidden">Drag Bolt back and release</div>

<div id="overlay">
  <div class="card glass">
    <h1 id="title">Cheyenne N. Pilling's - Stress Reliever</h1>
    <p id="text"></p>
    <button id="btn">Play</button>
  </div>
</div>

<script type="importmap">
{
  "imports": {
    "three": "https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js",
    "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/",
    "cannon-es": "https://cdn.jsdelivr.net/npm/cannon-es@0.20.0/dist/cannon-es.js"
  }
}
</script>

<script type="module">
import * as THREE from 'three';
import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/addons/postprocessing/RenderPass.js';
import { UnrealBloomPass } from 'three/addons/postprocessing/UnrealBloomPass.js';
import { OutputPass } from 'three/addons/postprocessing/OutputPass.js';
import * as CANNON from 'cannon-es';

/* ================= constants & levels ================= */
const ANCHOR = { x: -8, y: 3 };
const MAX_PULL = 2.4;
const DT = 1 / 60;

const MAT = {
  wood:  { hp: 45, color: 0xc8894b, rough: 0.7,  metal: 0,    opacity: 1 },
  glass: { hp: 25, color: 0x9be7f5, rough: 0.05, metal: 0,    opacity: 0.7 },
  stone: { hp: 90, color: 0x9aa0a8, rough: 0.95, metal: 0.05, opacity: 1 },
};

const LEVELS = [
  { name: 'Scrap Yard', shots: 3, build(s) {
      s.tower(2, 2, 'wood'); s.tower(7, 3, 'glass'); s.tower(12, 2, 'wood'); } },
  { name: 'Glass Gardens', shots: 4, build(s) {
      s.tower(1, 3, 'wood'); s.tower(5, 1, 'stone'); s.tower(9, 4, 'glass'); s.goblin(14, 0.45); } },
  { name: 'Iron Citadel', shots: 4, build(s) {
      s.tower(0, 4, 'stone'); s.tower(5, 3, 'wood'); s.tower(10, 3, 'glass'); } },
  { name: 'Twin Peaks', shots: 4, build(s) {
      s.tower(4, 4, 'wood'); s.tower(10, 4, 'wood'); s.tower(16, 2, 'glass'); } },
  { name: 'Goblin Gully', shots: 5, build(s) {
      s.goblin(3, 0.45); s.goblin(6, 0.45); s.goblin(9, 0.45); s.goblin(12, 0.45);
      s.tower(7, 2, 'stone'); s.tower(14, 3, 'glass'); } },
  { name: 'Glass Storm', shots: 4, build(s) {
      s.tower(2, 3, 'glass'); s.tower(6, 3, 'glass'); s.tower(10, 3, 'glass');
      s.tower(14, 3, 'glass'); s.tower(18, 1, 'stone'); } },
  { name: 'Stone Fortress', shots: 5, build(s) {
      s.tower(3, 5, 'stone'); s.tower(9, 5, 'stone'); s.tower(15, 4, 'wood');
      s.goblin(6, 0.45); s.goblin(12, 0.45); } },
  { name: 'The Tallest Tower', shots: 5, build(s) {
      s.tower(8, 5, 'stone'); s.tower(3, 2, 'glass'); s.tower(14, 2, 'glass');
      s.block(11, 0.25, 1.5, 0.5, 'wood'); s.goblin(11, 0.8); } },
];

/* ================= renderer, scene, camera ================= */
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
renderer.setSize(innerWidth, innerHeight);
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 1.1;
renderer.outputColorSpace = THREE.SRGBColorSpace;
document.body.prepend(renderer.domElement);

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x07011a);
scene.fog = new THREE.Fog(0x2a0a4a, 45, 110);

const camera = new THREE.PerspectiveCamera(45, innerWidth / innerHeight, 0.1, 300);
const CAM_BASE = new THREE.Vector3(3, 7, 26);
const CAM_TARGET = new THREE.Vector3(3, 4.5, 0);
camera.position.copy(CAM_BASE);
camera.lookAt(CAM_TARGET);

// lights: sky/ground ambient + warm key light with soft shadows + cool rim
scene.add(new THREE.HemisphereLight(0x9ad7ff, 0x203020, 0.9));
const sun = new THREE.DirectionalLight(0xffe0b0, 2.4);
sun.position.set(10, 22, 18);
sun.castShadow = true;
sun.shadow.mapSize.set(2048, 2048);
Object.assign(sun.shadow.camera, { left: -26, right: 26, top: 22, bottom: -6, near: 1, far: 80 });
sun.shadow.bias = -0.0004;
sun.shadow.radius = 4;
scene.add(sun);
const rim = new THREE.DirectionalLight(0x00f0ff, 1.2);
rim.position.set(-14, 8, -12);
scene.add(rim);

// ground
const ground = new THREE.Mesh(
  new THREE.PlaneGeometry(260, 260),
  new THREE.MeshStandardMaterial({ color: 0x0d0a24, roughness: 0.9 })
);
ground.rotation.x = -Math.PI / 2;
ground.receiveShadow = true;
scene.add(ground);
const neonGrid = new THREE.GridHelper(200, 80, 0xff2bd6, 0x00f0ff);
neonGrid.position.y = 0.02;
neonGrid.material.transparent = true;
neonGrid.material.opacity = 0.35;
scene.add(neonGrid);

// starfield and sun disc
{
  const count = 900, pos = new Float32Array(count * 3);
  for (let i = 0; i < count; i++) {
    pos[i * 3] = (Math.random() - 0.5) * 320;
    pos[i * 3 + 1] = Math.random() * 120 + 6;
    pos[i * 3 + 2] = -60 - Math.random() * 80;
  }
  const geo = new THREE.BufferGeometry();
  geo.setAttribute('position', new THREE.BufferAttribute(pos, 3));
  scene.add(new THREE.Points(geo, new THREE.PointsMaterial({ color: 0xffffff, size: 0.35, sizeAttenuation: true })));
  const disc = new THREE.Mesh(new THREE.SphereGeometry(4, 32, 16),
    new THREE.MeshBasicMaterial({ color: 0xffd166 }));
  disc.position.set(22, 30, -90);
  scene.add(disc);
}

// slingshot
{
  const wood = new THREE.MeshStandardMaterial({ color: 0x6b4423, roughness: 0.8 });
  const post = new THREE.Mesh(new THREE.CylinderGeometry(0.16, 0.2, ANCHOR.y + 0.2, 14), wood);
  post.position.set(ANCHOR.x, (ANCHOR.y + 0.2) / 2, 0);
  post.castShadow = true;
  scene.add(post);
  for (const sx of [-0.45, 0.45]) {
    const arm = new THREE.Mesh(new THREE.CylinderGeometry(0.09, 0.12, 1.2, 10), wood);
    arm.position.set(ANCHOR.x + sx, ANCHOR.y + 0.5, 0);
    arm.rotation.z = -sx * 0.6;
    arm.castShadow = true;
    scene.add(arm);
  }
}

// elastic band
const bandGeo = new THREE.BufferGeometry();
const band = new THREE.Line(bandGeo, new THREE.LineBasicMaterial({ color: 0x3b2412 }));
band.visible = false;
scene.add(band);

/* ================= post-processing (bloom + output tone) ================= */
const composer = new EffectComposer(renderer);
composer.addPass(new RenderPass(scene, camera));
const bloom = new UnrealBloomPass(new THREE.Vector2(innerWidth, innerHeight), 0.55, 0.5, 0.8);
composer.addPass(bloom);
composer.addPass(new OutputPass());

/* ================= physics ================= */
const world = new CANNON.World({ gravity: new CANNON.Vec3(0, -14, 0) });
world.broadphase = new CANNON.SAPBroadphase(world);
world.allowSleep = false;
world.solver.iterations = 12;
world.defaultContactMaterial.friction = 0.45;
world.defaultContactMaterial.restitution = 0.15;

{
  const floor = new CANNON.Body({ mass: 0, shape: new CANNON.Plane() });
  floor.quaternion.setFromEuler(-Math.PI / 2, 0, 0);
  world.addBody(floor);
  const walls = [
    [-22, 12, 0, 0.5, 20, 5],   // left
    [24, 12, 0, 0.5, 20, 5],    // right
    [0, 24, 0, 30, 0.5, 5],     // ceiling
    [0, 10, -1.4, 30, 20, 0.2], // back lane wall
    [0, 10, 1.4, 30, 20, 0.2],  // front lane wall
  ];
  for (const [x, y, z, hx, hy, hz] of walls) {
    const w = new CANNON.Body({ mass: 0, shape: new CANNON.Box(new CANNON.Vec3(hx, hy, hz)) });
    w.position.set(x, y, z);
    world.addBody(w);
  }
}

/* ================= meshes for bodies ================= */
function addBlock(x, y, w, h, d, mat) {
  const m = MAT[mat];
  const mesh = new THREE.Mesh(
    new THREE.BoxGeometry(w, h, d),
    new THREE.MeshStandardMaterial({
      color: m.color, roughness: m.rough, metalness: m.metal,
      transparent: m.opacity < 1, opacity: m.opacity,
    })
  );
  mesh.castShadow = true;
  mesh.receiveShadow = true;
  scene.add(mesh);

  const body = new CANNON.Body({ mass: 1.2, shape: new CANNON.Box(new CANNON.Vec3(w / 2, h / 2, d / 2)) });
  body.position.set(x, y, 0);
  world.addBody(body);
  Object.assign(body, { kind: 'block', mat, hp: m.hp, mesh });
  body.addEventListener('collide', onCollide);
  return body;
}

function makeGoblinMesh() {
  const g = new THREE.Group();
  const skin = new THREE.MeshStandardMaterial({ color: 0x4caf50, roughness: 0.55 });
  const dark = new THREE.MeshStandardMaterial({ color: 0x1b3d1b, roughness: 0.8 });
  const head = new THREE.Mesh(new THREE.SphereGeometry(0.45, 32, 24), skin);
  head.castShadow = true;
  g.add(head);
  for (const sx of [-1, 1]) {
    const ear = new THREE.Mesh(new THREE.ConeGeometry(0.16, 0.5, 12), skin);
    ear.rotation.z = sx * Math.PI / 2;
    ear.position.set(sx * 0.5, 0.05, 0);
    ear.castShadow = true;
    g.add(ear);
    const eye = new THREE.Mesh(new THREE.SphereGeometry(0.14, 20, 14),
      new THREE.MeshStandardMaterial({ color: 0xffffff, emissive: 0xffffff, emissiveIntensity: 0.25 }));
    eye.position.set(sx * 0.17, 0.12, 0.36);
    g.add(eye);
    const pupil = new THREE.Mesh(new THREE.SphereGeometry(0.07, 14, 10), dark);
    pupil.position.set(sx * 0.19, 0.11, 0.46);
    g.add(pupil);
    const brow = new THREE.Mesh(new THREE.BoxGeometry(0.3, 0.05, 0.05), dark);
    brow.position.set(sx * 0.17, 0.27, 0.42);
    brow.rotation.z = -sx * 0.4;
    g.add(brow);
  }
  const mouth = new THREE.Mesh(new THREE.BoxGeometry(0.22, 0.04, 0.05), dark);
  mouth.position.set(0, -0.18, 0.42);
  g.add(mouth);
  return g;
}

function addGoblin(x, y) {
  const mesh = makeGoblinMesh();
  scene.add(mesh);
  const body = new CANNON.Body({ mass: 1, shape: new CANNON.Sphere(0.45) });
  body.position.set(x, y, 0);
  world.addBody(body);
  Object.assign(body, { kind: 'enemy', hp: 60, mesh });
  body.addEventListener('collide', onCollide);
  return body;
}

function makeHeroMesh() {
  const g = new THREE.Group();
  const shell = new THREE.Mesh(new THREE.SphereGeometry(0.45, 32, 24),
    new THREE.MeshStandardMaterial({ color: 0xcaf0f8, metalness: 0.7, roughness: 0.25 }));
  shell.castShadow = true;
  g.add(shell);
  const visor = new THREE.Mesh(new THREE.BoxGeometry(0.62, 0.26, 0.1),
    new THREE.MeshStandardMaterial({ color: 0x023e8a, emissive: 0x0a6fd6, emissiveIntensity: 0.6, roughness: 0.1 }));
  visor.position.set(0, 0.05, 0.4);
  g.add(visor);
  for (const sx of [-1, 1]) {
    const eye = new THREE.Mesh(new THREE.BoxGeometry(0.14, 0.08, 0.02),
      new THREE.MeshStandardMaterial({ color: 0x4cc9f0, emissive: 0x4cc9f0, emissiveIntensity: 2 }));
    eye.position.set(sx * 0.14, 0.05, 0.47);
    g.add(eye);
  }
  const stick = new THREE.Mesh(new THREE.CylinderGeometry(0.02, 0.02, 0.35, 8),
    new THREE.MeshStandardMaterial({ color: 0x8d99ae, metalness: 0.8, roughness: 0.3 }));
  stick.position.set(0, 0.52, 0);
  g.add(stick);
  const tip = new THREE.Mesh(new THREE.SphereGeometry(0.07, 14, 10),
    new THREE.MeshStandardMaterial({ color: 0xef476f, emissive: 0xef476f, emissiveIntensity: 2.5 }));
  tip.position.set(0, 0.72, 0);
  g.add(tip);
  return g;
}

let hero = null;
function makeHero() {
  if (hero) {
    world.removeBody(hero);
    scene.remove(hero.mesh);
  }
  const mesh = makeHeroMesh();
  scene.add(mesh);
  hero = new CANNON.Body({ mass: 0, shape: new CANNON.Sphere(0.45), linearDamping: 0.05, angularDamping: 0.2 });
  hero.position.set(ANCHOR.x, ANCHOR.y, 0);
  world.addBody(hero);
  hero.mesh = mesh;
  hero.kind = 'hero';
}

/* ================= particles & popups ================= */
const partGeo = new THREE.BoxGeometry(0.12, 0.12, 0.12);
let parts = [];
function burst(pos, color, n = 16, speed = 6) {
  for (let i = 0; i < n; i++) {
    const m = new THREE.Mesh(partGeo, new THREE.MeshStandardMaterial({
      color, emissive: color, emissiveIntensity: 0.8, roughness: 0.4,
    }));
    m.position.copy(pos);
    scene.add(m);
    const v = new THREE.Vector3(
      (Math.random() - 0.5) * speed, Math.random() * speed, (Math.random() - 0.5) * speed * 0.5);
    parts.push({ m, v, life: 1 });
  }
  if (parts.length > 400) {
    for (const p of parts.splice(0, parts.length - 400)) { scene.remove(p.m); p.m.material.dispose(); }
  }
}
function updateParticles(dt) {
  for (const p of parts) {
    p.v.y -= 15 * dt;
    p.m.position.addScaledVector(p.v, dt);
    p.m.rotation.x += dt * 6;
    p.m.rotation.z += dt * 4;
    p.life -= dt * 0.9;
    p.m.scale.setScalar(Math.max(p.life, 0.01));
  }
  for (const p of parts.filter(p => p.life <= 0)) { scene.remove(p.m); p.m.material.dispose(); }
  parts = parts.filter(p => p.life > 0);
}
function popup(pos, text) {
  const v = pos.clone().project(camera);
  const el = document.createElement('div');
  el.className = 'pop';
  el.textContent = text;
  el.style.left = ((v.x + 1) / 2 * innerWidth) + 'px';
  el.style.top = ((-v.y + 1) / 2 * innerHeight) + 'px';
  document.body.appendChild(el);
  setTimeout(() => el.remove(), 900);
}

/* ================= audio: Web Audio sfx + procedural music ================= */
let AC = null, master, musicBus, sfxBus, reverb, muted = false;
const BPM = 124, STEP = 60 / BPM / 2; // eighth-note step
const CHORDS = [[57, 60, 64], [53, 57, 60], [60, 64, 67], [55, 59, 62]]; // Am F C G
let step = 0, nextTime = 0, musicOn = false;

function makeImpulse(seconds) {
  const len = Math.floor(AC.sampleRate * seconds);
  const buf = AC.createBuffer(2, len, AC.sampleRate);
  for (let ch = 0; ch < 2; ch++) {
    const d = buf.getChannelData(ch);
    for (let i = 0; i < len; i++) d[i] = (Math.random() * 2 - 1) * Math.pow(1 - i / len, 3);
  }
  return buf;
}

function initAudio() {
  if (AC) { if (AC.state === 'suspended') AC.resume(); return; }
  AC = new (window.AudioContext || window.webkitAudioContext)();
  master = AC.createGain();
  master.gain.value = muted ? 0 : 0.85;
  master.connect(AC.destination);

  reverb = AC.createConvolver();
  reverb.buffer = makeImpulse(2.4);
  const wet = AC.createGain();
  wet.gain.value = 0.3;
  reverb.connect(wet).connect(master);

  musicBus = AC.createGain();
  musicBus.gain.value = 0.28;
  musicBus.connect(master);
  musicBus.connect(reverb);

  sfxBus = AC.createGain();
  sfxBus.gain.value = 0.7;
  sfxBus.connect(master);
  sfxBus.connect(reverb);

  startMusic();
}

function midiHz(m) { return 440 * Math.pow(2, (m - 69) / 12); }

function osc(type, freq, t, dur, dest, vol, filterFreq = 0) {
  const o = AC.createOscillator(), g = AC.createGain();
  o.type = type;
  o.frequency.setValueAtTime(freq, t);
  g.gain.setValueAtTime(0.0001, t);
  g.gain.exponentialRampToValueAtTime(vol, t + 0.01);
  g.gain.exponentialRampToValueAtTime(0.0001, t + dur);
  let node = o;
  if (filterFreq) {
    const f = AC.createBiquadFilter();
    f.type = 'lowpass';
    f.frequency.value = filterFreq;
    o.connect(f);
    node = f;
  }
  node.connect(g).connect(dest);
  o.start(t);
  o.stop(t + dur + 0.05);
}

function kick(t) {
  const o = AC.createOscillator(), g = AC.createGain();
  o.frequency.setValueAtTime(150, t);
  o.frequency.exponentialRampToValueAtTime(42, t + 0.15);
  g.gain.setValueAtTime(0.9, t);
  g.gain.exponentialRampToValueAtTime(0.001, t + 0.25);
  o.connect(g).connect(musicBus);
  o.start(t); o.stop(t + 0.3);
}

function noiseBurst(t, dur, filterType, freq, vol, dest, sweepTo = 0) {
  const len = Math.floor(AC.sampleRate * dur);
  const buf = AC.createBuffer(1, len, AC.sampleRate);
  const d = buf.getChannelData(0);
  for (let i = 0; i < len; i++) d[i] = (Math.random() * 2 - 1);
  const src = AC.createBufferSource();
  src.buffer = buf;
  const f = AC.createBiquadFilter();
  f.type = filterType;
  f.frequency.setValueAtTime(freq, t);
  if (sweepTo) f.frequency.exponentialRampToValueAtTime(sweepTo, t + dur);
  const g = AC.createGain();
  g.gain.setValueAtTime(vol, t);
  g.gain.exponentialRampToValueAtTime(0.0001, t + dur);
  src.connect(f).connect(g).connect(dest);
  src.start(t);
}

function playStep(s, t) {
  const bar = Math.floor(s / 8) % 4;
  const pos = s % 8;
  const chord = CHORDS[bar];

  if (pos === 0) {
    // pad: detuned saws through a slow filter
    for (const n of chord) {
      osc('sawtooth', midiHz(n), t, 8 * STEP, musicBus, 0.035, 900);
      osc('sawtooth', midiHz(n) * 1.004, t, 8 * STEP, musicBus, 0.03, 900);
    }
  }
  if (pos % 4 === 0) kick(t);
  if (pos === 4) noiseBurst(t, 0.14, 'bandpass', 1800, 0.35, musicBus);
  if (pos % 2 === 1) noiseBurst(t, 0.03, 'highpass', 7000, 0.12, musicBus);
  if (pos % 2 === 0) osc('sawtooth', midiHz(chord[0] - 12), t, STEP * 1.8, musicBus, 0.22, 450);

  // arpeggio, 16th-feel with octave shimmer
  const arp = [chord[0], chord[1], chord[2], chord[1]][pos % 4];
  osc('square', midiHz(arp + 12 * (Math.floor(pos / 4) % 2)), t, STEP * 0.9, musicBus, 0.05, 2400);
}

function scheduler() {
  if (!musicOn || !AC) return;
  while (nextTime < AC.currentTime + 0.15) {
    playStep(step, nextTime);
    nextTime += STEP;
    step++;
  }
  setTimeout(scheduler, 25);
}

function startMusic() {
  if (musicOn) return;
  musicOn = true;
  nextTime = AC.currentTime + 0.1;
  scheduler();
}

function sfxLaunch() {
  if (!AC) return;
  noiseBurst(AC.currentTime, 0.35, 'bandpass', 300, 0.5, sfxBus, 2600);
  osc('sawtooth', 220, AC.currentTime, 0.25, sfxBus, 0.08);
}
function sfxHit(p) {
  if (!AC) return;
  osc('triangle', 180 + p * 80, AC.currentTime, 0.08, sfxBus, 0.25);
}

// robot impact: metallic clang with material-specific tone, a click, and a thud
function sfxRobotImpact(kind, p) {
  if (!AC) return;
  const t = AC.currentTime, v = 0.12 + p * 0.3;
  const base = kind === 'glass' ? 1800 : kind === 'stone' ? 520 : kind === 'goblin' ? 340 : 260;
  for (const ratio of [1, 2.76, 5.4]) {
    const o = AC.createOscillator(), g = AC.createGain();
    o.type = kind === 'glass' ? 'sine' : 'triangle';
    o.frequency.setValueAtTime(base * ratio, t);
    g.gain.setValueAtTime(v / ratio, t);
    g.gain.exponentialRampToValueAtTime(0.0001, t + (kind === 'glass' ? 0.6 : 0.25) / ratio);
    o.connect(g).connect(sfxBus);
    o.start(t);
    o.stop(t + 0.7);
  }
  noiseBurst(t, 0.03, 'highpass', 3000, v * 0.8, sfxBus);
  if (kind !== 'glass') osc('sine', 90, t, 0.18, sfxBus, v * 0.6);
}
function sfxSmash() {
  if (!AC) return;
  noiseBurst(AC.currentTime, 0.3, 'lowpass', 2200, 0.5, sfxBus, 300);
  osc('sawtooth', 120, AC.currentTime, 0.2, sfxBus, 0.2);
}
function sfxPop() {
  if (!AC) return;
  const t = AC.currentTime;
  const o = AC.createOscillator(), g = AC.createGain();
  o.type = 'square';
  o.frequency.setValueAtTime(660, t);
  o.frequency.exponentialRampToValueAtTime(960, t + 0.2);
  g.gain.setValueAtTime(0.18, t);
  g.gain.exponentialRampToValueAtTime(0.0001, t + 0.22);
  o.connect(g).connect(sfxBus);
  o.start(t); o.stop(t + 0.25);
}
function sfxWin() {
  if (!AC) return;
  [523, 659, 784, 1047].forEach((f, i) => osc('triangle', f, AC.currentTime + i * 0.12, 0.35, sfxBus, 0.25));
}
function sfxLose() {
  if (!AC) return;
  const t = AC.currentTime;
  const o = AC.createOscillator(), g = AC.createGain();
  o.type = 'sawtooth';
  o.frequency.setValueAtTime(300, t);
  o.frequency.exponentialRampToValueAtTime(50, t + 0.7);
  g.gain.setValueAtTime(0.2, t);
  g.gain.exponentialRampToValueAtTime(0.0001, t + 0.8);
  o.connect(g).connect(sfxBus);
  o.start(t); o.stop(t + 0.9);
}

function setMuted(m) {
  muted = m;
  document.getElementById('mute').textContent = m ? '🔇' : '🔊';
  if (master) master.gain.setTargetAtTime(m ? 0 : 0.85, AC.currentTime, 0.05);
}
document.getElementById('mute').onclick = () => setMuted(!muted);
addEventListener('keydown', e => { if (e.key === 'm' || e.key === 'M') setMuted(!muted); });

/* ================= game state ================= */
const el = {
  overlay: document.getElementById('overlay'),
  title: document.getElementById('title'),
  text: document.getElementById('text'),
  btn: document.getElementById('btn'),
  hud: document.getElementById('hud'),
  hint: document.getElementById('hint'),
  score: document.getElementById('score'),
  shots: document.getElementById('shots'),
  level: document.getElementById('level'),
};

let levelIndex = 0, shotsLeft = 0, score = 0, levelStartScore = 0;
let state = 'over', flyT = 0, endT = 0, dragging = false, overlayFn = () => {};
let toRemove = new Set(), lastHit = 0, shake = 0;

function goblinsLeft() { return world.bodies.filter(b => b.kind === 'enemy').length; }

function buildLevelHelpers() {
  return {
    tower(x, floors, mat) {
      const w = 0.5, h = 2.4, d = 1.0;
      for (let f = 0; f < floors; f++) {
        const y = h / 2 + f * h;
        addBlock(x - 0.9, y, w, h, d, mat);
        addBlock(x + 0.9, y, w, h, d, mat);
      }
      const top = floors * h;
      addBlock(x, top + 0.25, 2.4, 0.5, d, mat);
      addGoblin(x, top + 0.5 + 0.45);
    },
    goblin(x, y) { addGoblin(x, y); },
    block(x, y, w, h, mat) { addBlock(x, y, w, h, 1.0, mat); },
  };
}

function clearLevel() {
  for (const b of [...world.bodies]) {
    if (b.kind && b.kind !== 'hero') {
      world.removeBody(b);
      scene.remove(b.mesh);
    }
  }
  toRemove.clear();
}

function loadLevel(i) {
  levelIndex = i;
  levelStartScore = score;
  clearLevel();
  LEVELS[i].build(buildLevelHelpers());
  shotsLeft = LEVELS[i].shots;
  makeHero();
  state = 'ready';
  hideMenu();
  el.hud.classList.remove('hidden');
  el.hint.classList.remove('hidden');
  renderHUD();
}

function renderHUD() {
  el.score.textContent = `Score ${score}`;
  el.level.textContent = LEVELS[levelIndex].name;
  el.shots.innerHTML = '';
  for (let i = 0; i < shotsLeft; i++) {
    const d = document.createElement('div');
    d.className = 'dot';
    el.shots.appendChild(d);
  }
  el.hint.style.opacity = state === 'ready' && !dragging ? 1 : 0;
}

function showMenu(title, text, btn, fn) {
  state = 'over';
  el.title.textContent = title;
  el.text.textContent = text;
  el.btn.textContent = btn;
  overlayFn = fn;
  el.overlay.classList.remove('hidden');
  el.hint.style.opacity = 0;
}
function hideMenu() { el.overlay.classList.add('hidden'); }
el.btn.onclick = () => { initAudio(); overlayFn(); };

/* ================= collisions & damage ================= */
function onCollide(e) {
  const a = e.target, b = e.body;
  if (!a.kind && !b.kind) return;
  const rel = a.velocity.distanceTo(b.velocity);
  if (rel < 2.5) return;
  if (a.kind && a.kind !== 'hero') hurt(a, rel);
  if (b.kind && b.kind !== 'hero') hurt(b, rel);
  const now = performance.now();
  if (rel > 3 && now - lastHit > 60) {
    lastHit = now;
    const heroHit = a.kind === 'hero' || b.kind === 'hero';
    const other = a.kind === 'hero' ? b : a;
    if (heroHit) sfxRobotImpact(other.kind === 'enemy' ? 'goblin' : (other.mat || 'wood'), Math.min(rel / 14, 1));
    else sfxHit(Math.min(rel / 15, 1));
  }
  if (rel > 7) {
    const mid = new THREE.Vector3(
      (a.position.x + b.position.x) / 2, (a.position.y + b.position.y) / 2, 0);
    burst(mid, 0xffffff, 3, 2);
  }
}
function hurt(b, rel) {
  if (b.dead) return;
  b.hp -= rel * 3;
  if (b.hp <= 0) { b.dead = true; toRemove.add(b); }
}

function processRemovals() {
  for (const b of toRemove) {
    const pos = b.mesh.position.clone();
    if (b.kind === 'enemy') {
      burst(pos, 0x7CFC00, 26, 9);
      popup(pos.clone().add(new THREE.Vector3(0, 0.8, 0)), '+500');
      score += 500;
      sfxPop();
      shake = Math.max(shake, 0.35);
    } else if (b.kind === 'block') {
      burst(pos, MAT[b.mat].color, 12, 6);
      score += 50;
      sfxSmash();
      shake = Math.max(shake, 0.12);
    }
    world.removeBody(b);
    scene.remove(b.mesh);
  }
  toRemove.clear();
  renderHUD();
}

/* ================= input ================= */
const ray = new THREE.Raycaster();
const playPlane = new THREE.Plane(new THREE.Vector3(0, 0, 1), 0);
const hit = new THREE.Vector3();

function pointerWorld(e) {
  const ndc = new THREE.Vector2((e.clientX / innerWidth) * 2 - 1, -(e.clientY / innerHeight) * 2 + 1);
  ray.setFromCamera(ndc, camera);
  ray.ray.intersectPlane(playPlane, hit);
  return hit.clone();
}

renderer.domElement.addEventListener('pointerdown', e => {
  initAudio();
  if (state !== 'ready') return;
  const p = pointerWorld(e);
  const h = new THREE.Vector3(hero.position.x, hero.position.y, 0);
  if (p.distanceTo(h) < 1.3) {
    dragging = true;
    renderer.domElement.setPointerCapture(e.pointerId);
    renderHUD();
  }
});
renderer.domElement.addEventListener('pointermove', e => {
  if (!dragging) return;
  const p = pointerWorld(e);
  let dx = p.x - ANCHOR.x, dy = p.y - ANCHOR.y;
  const d = Math.hypot(dx, dy);
  if (d > MAX_PULL) { dx *= MAX_PULL / d; dy *= MAX_PULL / d; }
  hero.position.set(ANCHOR.x + dx, ANCHOR.y + dy, 0);
  hero.velocity.set(0, 0, 0);
});
function release() {
  if (!dragging) return;
  dragging = false;
  const d = Math.hypot(ANCHOR.x - hero.position.x, ANCHOR.y - hero.position.y);
  if (d > 0.3) launch();
  else hero.position.set(ANCHOR.x, ANCHOR.y, 0);
  renderHUD();
}
addEventListener('pointerup', release);
addEventListener('pointercancel', release);

function launch() {
  const dx = ANCHOR.x - hero.position.x, dy = ANCHOR.y - hero.position.y;
  hero.type = CANNON.Body.DYNAMIC;
  hero.mass = 1.5;
  hero.updateMassProperties();
  hero.wakeUp();
  hero.velocity.set(dx * 9, dy * 9, 0);
  shotsLeft--;
  state = 'flying';
  flyT = 0;
  sfxLaunch();
}

function endShot() {
  if (shotsLeft === 0 && goblinsLeft() > 0) {
    sfxLose();
    showMenu('Out of robots!', `Score ${score}. The goblins survived.`, 'Retry', () => {
      score = levelStartScore;
      loadLevel(levelIndex);
    });
  } else {
    makeHero();
    state = 'ready';
  }
  renderHUD();
}

function updateFlying(dt) {
  flyT += dt;
  if (goblinsLeft() === 0) { state = 'cleared'; endT = 0; sfxWin(); return; }
  const p = hero.position, sp = hero.velocity.length();
  const gone = p.x > 24 || p.x < -24 || p.y < -6 || p.y > 30;
  if (gone || (flyT > 1.2 && sp < 0.4) || flyT > 8) endShot();
}

function onCleared() {
  const earned = shotsLeft >= 2 ? 3 : shotsLeft === 1 ? 2 : 1;
  const last = levelIndex === LEVELS.length - 1;
  showMenu('Level cleared!',
    `${'★'.repeat(earned)}${'☆'.repeat(3 - earned)}\nScore ${score}`,
    last ? 'Play again' : 'Next level',
    () => {
      if (last) { score = 0; loadLevel(0); }
      else loadLevel(levelIndex + 1);
    });
}

/* ================= resize & loop ================= */
function resize() {
  camera.aspect = innerWidth / innerHeight;
  // pull the camera back on narrow (portrait) screens so the whole lane stays visible
  CAM_BASE.z = 26 * Math.max(1, 1.78 / camera.aspect);
  camera.updateProjectionMatrix();
  renderer.setSize(innerWidth, innerHeight);
  composer.setSize(innerWidth, innerHeight);
  bloom.setSize(innerWidth, innerHeight);
}
addEventListener('resize', resize);
resize();

function syncMeshes() {
  for (const b of world.bodies) {
    if (!b.mesh) continue;
    b.mesh.position.set(b.position.x, b.position.y, b.position.z);
    b.mesh.quaternion.set(b.quaternion.x, b.quaternion.y, b.quaternion.z, b.quaternion.w);
  }
}

function updateBand() {
  if (!(state === 'ready' || dragging)) { band.visible = false; return; }
  band.visible = true;
  const hp = hero.position;
  const pts = new Float32Array([
    ANCHOR.x - 0.45, ANCHOR.y + 0.5, 0,
    hp.x, hp.y, hp.z,
    ANCHOR.x + 0.45, ANCHOR.y + 0.5, 0,
  ]);
  bandGeo.setAttribute('position', new THREE.BufferAttribute(pts, 3));
}

function updateCamera(dt) {
  shake *= Math.pow(0.001, dt);
  const s = shake * 0.6;
  camera.position.set(
    CAM_BASE.x + (Math.random() - 0.5) * s,
    CAM_BASE.y + (Math.random() - 0.5) * s,
    CAM_BASE.z);
  camera.lookAt(CAM_TARGET);
}

let last = performance.now(), acc = 0, trailTick = 0;
function frame(now) {
  const dt = Math.min((now - last) / 1000, 0.05);
  last = now;
  acc += dt;
  while (acc >= DT) {
    world.step(DT);
    acc -= DT;
    if (state === 'flying') updateFlying(DT);
    if (state === 'cleared') {
      endT += DT;
      if (endT > 1.2) { state = 'over'; onCleared(); }
    }
  }
  processRemovals();
  if (state === 'flying' && ++trailTick % 2 === 0) {
    burst(hero.position.clone(), 0x4cc9f0, 1, 0.3);
  }
  syncMeshes();
  updateBand();
  updateParticles(dt);
  updateCamera(dt);
  composer.render();
  requestAnimationFrame(frame);
}

// start with the title card; hero and scene are ready behind it
makeHero();
showMenu('Cheyenne N. Pilling\'s - Stress Reliever',
  'Pull Bolt back on the slingshot and release.\nSmash every goblin tower.',
  'Play', () => { score = 0; loadLevel(0); });
requestAnimationFrame(frame);
</script>
</body>
</html>
