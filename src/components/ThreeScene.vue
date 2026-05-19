<script setup lang="ts">
import { onBeforeUnmount, onMounted, ref } from "vue";
import * as THREE from "three";
import roseReferenceUrl from "../assets/mg.png";
import { messageCards, type CardTone } from "../data/messages";

const containerRef = ref<HTMLDivElement | null>(null);
const selectedCardText = ref<string | null>(null);
const showHint = ref(false);
const showRoseHint = ref(false);
const showRose = ref(false);
const showProposal = ref(false);
const disableRoseReplay = ref(false);
const proposalLocked = ref(false);
const proposalRejecting = ref(false);

let renderer: THREE.WebGLRenderer | null = null;
let scene: THREE.Scene | null = null;
let camera: THREE.PerspectiveCamera | null = null;
let animationId = 0;
let cardMeshes: THREE.Mesh[] = [];
let glowSprites: THREE.Sprite[] = [];
let hoveredCard: THREE.Mesh | null = null;
let heartGroup: THREE.Group | null = null;

// Animation state
let animState: "scattered" | "text" | "morphing" | "cards" | "rose" = "scattered";
let morphStartTime = 0;
let roseStartTime = 0;
let roseMesh: THREE.Mesh | null = null;
let roseGlow: THREE.Sprite | null = null;
let bouquetGroup: THREE.Group | null = null;
let bouquetAura: THREE.Sprite | null = null;
let bouquetReady = false;

// Particle systems
let heartPoints: THREE.Points | null = null;
let sparklePoints: THREE.Points | null = null;
let glowOrbs: THREE.Points | null = null;
let shootingStars: THREE.Points | null = null;

const raycaster = new THREE.Raycaster();
const pointer = new THREE.Vector2(-10, -10);

const CARD_W = 0.22;
const CARD_H = 0.12;
const CANVAS_W = 512;
const CANVAS_H = 280;

const toneColors: Record<CardTone, { bg1: string; bg2: string; glow: string }> = {
  peach: { bg1: "#FFE9D4", bg2: "#FFD4CA", glow: "#ff9e7a" },
  mint: { bg1: "#DEFFE8", bg2: "#BEF1D9", glow: "#6affb0" },
  sky: { bg1: "#DCF3FF", bg2: "#C1E2FF", glow: "#6ab8ff" },
  lavender: { bg1: "#EFE4FF", bg2: "#DDD1FF", glow: "#b388ff" },
  butter: { bg1: "#FFF5C6", bg2: "#FFE4AD", glow: "#ffd54f" },
  rose: { bg1: "#FFE3F0", bg2: "#FFCDE1", glow: "#ff6eb4" },
};

const tones: CardTone[] = ["peach", "mint", "sky", "lavender", "butter", "rose"];
const proposalFlowers = [
  { x: "6%", y: "16%", size: 110, delay: "0.08s", hue: "peach" },
  { x: "14%", y: "72%", size: 142, delay: "0.16s", hue: "rose" },
  { x: "24%", y: "20%", size: 92, delay: "0.22s", hue: "cream" },
  { x: "34%", y: "84%", size: 132, delay: "0.3s", hue: "pink" },
  { x: "46%", y: "10%", size: 98, delay: "0.36s", hue: "rose" },
  { x: "56%", y: "82%", size: 138, delay: "0.42s", hue: "peach" },
  { x: "68%", y: "16%", size: 108, delay: "0.5s", hue: "cream" },
  { x: "80%", y: "72%", size: 148, delay: "0.58s", hue: "rose" },
  { x: "92%", y: "24%", size: 118, delay: "0.64s", hue: "pink" },
  { x: "8%", y: "48%", size: 100, delay: "0.2s", hue: "pink" },
  { x: "92%", y: "52%", size: 102, delay: "0.54s", hue: "cream" },
  { x: "50%", y: "92%", size: 118, delay: "0.68s", hue: "rose" },
  { x: "20%", y: "8%", size: 86, delay: "0.26s", hue: "rose" },
  { x: "74%", y: "8%", size: 90, delay: "0.46s", hue: "peach" },
  { x: "22%", y: "92%", size: 108, delay: "0.62s", hue: "cream" },
  { x: "78%", y: "90%", size: 112, delay: "0.72s", hue: "pink" },
];
let witherStartTime = 0;

// ─── Heart parametric ───
function heartXY(t: number) {
  const x = 16 * Math.sin(t) ** 3;
  const y =
    13 * Math.cos(t) -
    5 * Math.cos(2 * t) -
    2 * Math.cos(3 * t) -
    Math.cos(4 * t);
  return { x: x / 17, y: y / 17 };
}

// ─── Canvas texture for cards ───
function roundRect(
  ctx: CanvasRenderingContext2D,
  x: number,
  y: number,
  w: number,
  h: number,
  r: number,
) {
  ctx.beginPath();
  ctx.moveTo(x + r, y);
  ctx.lineTo(x + w - r, y);
  ctx.quadraticCurveTo(x + w, y, x + w, y + r);
  ctx.lineTo(x + w, y + h - r);
  ctx.quadraticCurveTo(x + w, y + h, x + w - r, y + h);
  ctx.lineTo(x + r, y + h);
  ctx.quadraticCurveTo(x, y + h, x, y + h - r);
  ctx.lineTo(x, y + r);
  ctx.quadraticCurveTo(x, y, x + r, y);
  ctx.closePath();
}

function wrapText(
  ctx: CanvasRenderingContext2D,
  text: string,
  x: number,
  y: number,
  maxW: number,
  lineH: number,
) {
  const chars = text.split("");
  let line = "";
  const lines: string[] = [];
  for (const ch of chars) {
    const test = line + ch;
    if (ctx.measureText(test).width > maxW && line) {
      lines.push(line);
      line = ch;
    } else {
      line = test;
    }
  }
  lines.push(line);

  const startY = y - ((lines.length - 1) * lineH) / 2;
  for (let i = 0; i < lines.length; i++) {
    ctx.fillText(lines[i], x, startY + i * lineH);
  }
}

function createCardTexture(text: string, tone: CardTone): THREE.CanvasTexture {
  const canvas = document.createElement("canvas");
  canvas.width = CANVAS_W;
  canvas.height = CANVAS_H;
  const ctx = canvas.getContext("2d")!;
  const c = toneColors[tone];

  // Rounded rect background
  roundRect(ctx, 0, 0, CANVAS_W, CANVAS_H, 32);
  const grad = ctx.createLinearGradient(0, 0, CANVAS_W, CANVAS_H);
  grad.addColorStop(0, c.bg1);
  grad.addColorStop(1, c.bg2);
  ctx.fillStyle = grad;
  ctx.fill();

  // Subtle border
  ctx.strokeStyle = "rgba(255,255,255,0.45)";
  ctx.lineWidth = 2;
  ctx.stroke();

  // Text
  ctx.fillStyle = "rgba(41,44,59,0.82)";
  ctx.font = "bold 44px 'PingFang SC','Microsoft YaHei',sans-serif";
  ctx.textAlign = "center";
  ctx.textBaseline = "middle";
  wrapText(ctx, text, CANVAS_W / 2, CANVAS_H / 2, CANVAS_W - 64, 52);

  const tex = new THREE.CanvasTexture(canvas);
  tex.needsUpdate = true;
  return tex;
}

// ─── Glow sprite texture ───
function createGlowTexture(color: string): THREE.CanvasTexture {
  const size = 128;
  const canvas = document.createElement("canvas");
  canvas.width = size;
  canvas.height = size;
  const ctx = canvas.getContext("2d")!;
  const grad = ctx.createRadialGradient(
    size / 2,
    size / 2,
    0,
    size / 2,
    size / 2,
    size / 2,
  );
  grad.addColorStop(0, color + "55");
  grad.addColorStop(0.4, color + "22");
  grad.addColorStop(1, color + "00");
  ctx.fillStyle = grad;
  ctx.fillRect(0, 0, size, size);
  const tex = new THREE.CanvasTexture(canvas);
  tex.needsUpdate = true;
  return tex;
}

function createPetalTexture(image: CanvasImageSource): THREE.CanvasTexture {
  const canvas = document.createElement("canvas");
  canvas.width = 360;
  canvas.height = 420;
  const ctx = canvas.getContext("2d")!;
  ctx.clearRect(0, 0, canvas.width, canvas.height);

  ctx.beginPath();
  ctx.moveTo(180, 14);
  ctx.bezierCurveTo(280, 44, 338, 150, 298, 264);
  ctx.bezierCurveTo(262, 364, 214, 402, 180, 410);
  ctx.bezierCurveTo(146, 402, 98, 364, 62, 264);
  ctx.bezierCurveTo(22, 150, 80, 44, 180, 14);
  ctx.closePath();
  ctx.clip();

  const goldGradient = ctx.createLinearGradient(0, 0, 0, canvas.height);
  goldGradient.addColorStop(0, "rgba(255, 249, 215, 1)");
  goldGradient.addColorStop(0.34, "rgba(255, 220, 138, 0.98)");
  goldGradient.addColorStop(0.7, "rgba(221, 157, 52, 0.98)");
  goldGradient.addColorStop(1, "rgba(148, 89, 18, 0.98)");
  ctx.fillStyle = goldGradient;
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  ctx.globalAlpha = 0.22;
  ctx.filter = "saturate(0.7) contrast(1.15) brightness(1.1)";
  ctx.drawImage(image, 120, 10, 780, 780, 18, 12, 324, 386);
  ctx.filter = "none";
  ctx.globalAlpha = 1;

  const edgeGlow = ctx.createRadialGradient(180, 138, 20, 180, 138, 210);
  edgeGlow.addColorStop(0, "rgba(255,255,255,0.45)");
  edgeGlow.addColorStop(0.46, "rgba(255,244,212,0.12)");
  edgeGlow.addColorStop(1, "rgba(255,244,212,0)");
  ctx.fillStyle = edgeGlow;
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  ctx.globalCompositeOperation = "screen";
  ctx.strokeStyle = "rgba(255, 246, 223, 0.7)";
  ctx.lineWidth = 5;
  ctx.beginPath();
  ctx.moveTo(180, 24);
  ctx.bezierCurveTo(257, 82, 273, 180, 226, 332);
  ctx.stroke();
  ctx.beginPath();
  ctx.moveTo(180, 24);
  ctx.bezierCurveTo(118, 88, 105, 186, 145, 332);
  ctx.stroke();
  ctx.globalCompositeOperation = "source-over";

  ctx.strokeStyle = "rgba(255, 243, 215, 0.85)";
  ctx.lineWidth = 4;
  ctx.beginPath();
  ctx.moveTo(180, 14);
  ctx.bezierCurveTo(280, 44, 338, 150, 298, 264);
  ctx.bezierCurveTo(262, 364, 214, 402, 180, 410);
  ctx.bezierCurveTo(146, 402, 98, 364, 62, 264);
  ctx.bezierCurveTo(22, 150, 80, 44, 180, 14);
  ctx.closePath();
  ctx.stroke();

  const texture = new THREE.CanvasTexture(canvas);
  texture.colorSpace = THREE.SRGBColorSpace;
  texture.needsUpdate = true;
  return texture;
}

function createLeafGeometry() {
  const shape = new THREE.Shape();
  shape.moveTo(0, 0);
  shape.bezierCurveTo(0.07, 0.02, 0.13, 0.1, 0, 0.24);
  shape.bezierCurveTo(-0.13, 0.1, -0.07, 0.02, 0, 0);

  const geometry = new THREE.ShapeGeometry(shape, 24);
  geometry.translate(0, -0.02, 0);
  return geometry;
}

function createPetalGeometry(width: number, height: number) {
  const shape = new THREE.Shape();
  shape.moveTo(0, 0);
  shape.bezierCurveTo(width * 0.34, height * 0.1, width * 0.5, height * 0.62, 0, height);
  shape.bezierCurveTo(-width * 0.5, height * 0.62, -width * 0.34, height * 0.1, 0, 0);
  const geometry = new THREE.ShapeGeometry(shape, 26);
  geometry.translate(0, -height * 0.08, 0);
  return geometry;
}

function getResponsiveMetrics(width: number, height: number) {
  const narrow = width <= 768;
  const tiny = width <= 480;
  return {
    cameraZ: tiny ? 2.45 : narrow ? 2.2 : 1.8,
    heartScale: tiny ? 0.72 : narrow ? 0.84 : 1,
    bouquetScale: tiny ? 0.74 : narrow ? 0.84 : 1,
  };
}

function applyResponsiveLayout(width: number, height: number) {
  const metrics = getResponsiveMetrics(width, height);
  if (camera) {
    camera.position.z = metrics.cameraZ;
  }
  if (heartGroup) {
    heartGroup.scale.setScalar(metrics.heartScale);
  }
  if (heartPoints) {
    heartPoints.scale.setScalar(metrics.heartScale);
  }
  if (bouquetGroup) {
    bouquetGroup.scale.setScalar(metrics.bouquetScale);
  }
}

function createRoseBloomGroup(
  texture: THREE.Texture,
  glowTexture: THREE.Texture,
  bloomScale: number,
  phase: number,
) {
  const bloomGroup = new THREE.Group();
  const petals: THREE.Mesh[] = [];
  const layers = [
    { count: 13, radius: 0.122, y: -0.034, width: 0.27, height: 0.43, tilt: 1.58, open: 0.62, colorA: "#f5ca73", colorB: "#ebbe5d" },
    { count: 12, radius: 0.084, y: 0.03, width: 0.22, height: 0.36, tilt: 1.3, open: 0.42, colorA: "#ffe1a0", colorB: "#f7cf78" },
    { count: 10, radius: 0.048, y: 0.096, width: 0.17, height: 0.28, tilt: 1.04, open: 0.26, colorA: "#fff0c6", colorB: "#ffdfa4" },
    { count: 7, radius: 0.018, y: 0.148, width: 0.126, height: 0.206, tilt: 0.78, open: 0.12, colorA: "#fff8de", colorB: "#ffe7bf" },
  ];

  for (const layer of layers) {
    const petalGeometry = createPetalGeometry(layer.width, layer.height);
    for (let i = 0; i < layer.count; i++) {
      const angle =
        (i / layer.count) * Math.PI * 2 +
        (layer.count % 2 === 0 ? Math.PI / layer.count : 0);
      const petal = new THREE.Mesh(
        petalGeometry,
        new THREE.MeshBasicMaterial({
          map: texture,
          transparent: true,
          alphaTest: 0.08,
          depthWrite: false,
          side: THREE.DoubleSide,
          color: new THREE.Color(i % 2 === 0 ? layer.colorA : layer.colorB),
          opacity: 0,
        }),
      );
      petal.position.set(
        Math.sin(angle) * layer.radius,
        layer.y + Math.cos(angle * 1.6) * 0.01,
        Math.cos(angle) * layer.radius * 0.55,
      );
      petal.userData = {
        startRotY: angle + Math.PI * 0.5,
        endRotY: angle,
        startRotX: -0.14,
        endRotX: layer.tilt,
        startRotZ: 0,
        endRotZ: -Math.sin(angle) * layer.open,
      };
      bloomGroup.add(petal);
      petals.push(petal);
    }
  }

  for (let i = 0; i < 6; i++) {
    const angle = (i / 6) * Math.PI * 2 + Math.PI / 6;
    const core = new THREE.Mesh(
      createPetalGeometry(0.118, 0.21),
      new THREE.MeshBasicMaterial({
        map: texture,
        transparent: true,
        alphaTest: 0.08,
        depthWrite: false,
        side: THREE.DoubleSide,
        color: new THREE.Color("#fff9ea"),
        opacity: 0,
      }),
    );
    core.position.set(Math.sin(angle) * 0.012, 0.162, Math.cos(angle) * 0.012);
    core.userData = {
      startRotY: angle + Math.PI * 0.5,
      endRotY: angle,
      startRotX: 0.08,
      endRotX: 0.52,
      startRotZ: 0,
      endRotZ: Math.sin(angle) * 0.1,
    };
    bloomGroup.add(core);
    petals.push(core);
  }

  const centerBud = new THREE.Mesh(
    createPetalGeometry(0.104, 0.176),
    new THREE.MeshBasicMaterial({
      map: texture,
      transparent: true,
      alphaTest: 0.08,
      depthWrite: false,
      side: THREE.DoubleSide,
      color: new THREE.Color("#fff6de"),
      opacity: 0,
    }),
  );
  centerBud.position.set(0, 0.16, 0);
  centerBud.userData = {
    startRotY: 0,
    endRotY: 0,
    startRotX: 0.06,
      endRotX: 0.28,
    startRotZ: 0,
    endRotZ: 0,
  };
  bloomGroup.add(centerBud);
  petals.push(centerBud);

  for (let i = 0; i < 5; i++) {
    const angle = (i / 5) * Math.PI * 2;
    const sepal = new THREE.Mesh(
      createPetalGeometry(0.11, 0.16),
      new THREE.MeshBasicMaterial({
        color: new THREE.Color("#c39235"),
        transparent: true,
        opacity: 0,
        depthWrite: false,
        side: THREE.DoubleSide,
      }),
    );
    sepal.position.set(Math.sin(angle) * 0.072, -0.028, Math.cos(angle) * 0.072);
    sepal.rotation.y = angle;
    sepal.rotation.x = 1.42;
    sepal.userData = {
      startRotY: angle,
      endRotY: angle,
      startRotX: 1.02,
      endRotX: 1.42,
      startRotZ: 0,
      endRotZ: 0,
    };
    bloomGroup.add(sepal);
    petals.push(sepal);
  }

  const glow = new THREE.Sprite(
    new THREE.SpriteMaterial({
      map: glowTexture,
      transparent: true,
      opacity: 0,
      blending: THREE.AdditiveBlending,
      depthWrite: false,
      color: new THREE.Color("#ffd985"),
    }),
  );
  glow.scale.set(0.85, 0.85, 1);
  glow.position.set(0, 0.04, -0.06);
  bloomGroup.add(glow);

  bloomGroup.userData = {
    petals,
    glow,
    finalScale: bloomScale,
    phase,
  };

  return bloomGroup;
}

function buildBouquet(targetScene: THREE.Scene) {
  bouquetGroup = new THREE.Group();
  bouquetGroup.visible = false;
  bouquetGroup.position.set(0, -0.18, -0.22);
  targetScene.add(bouquetGroup);

  const image = new Image();
  image.decoding = "async";
  image.onload = () => {
    if (!bouquetGroup) return;

    const petalTexture = createPetalTexture(image);
    const glowTexture = createGlowTexture("#ffd88a");
    const leafGeometry = createLeafGeometry();
    const specs = [
      { x: -0.52, z: -0.08, stemHeight: 0.9, lean: 0.66, bloomScale: 1.18, stemStart: 0.68, leafStart: 1.24, bloomStart: 1.92, phase: 0.18 },
      { x: -0.4, z: 0.01, stemHeight: 0.98, lean: 0.52, bloomScale: 1.08, stemStart: 0.78, leafStart: 1.34, bloomStart: 2.02, phase: 0.52 },
      { x: -0.28, z: 0.09, stemHeight: 1.02, lean: 0.36, bloomScale: 0.98, stemStart: 0.88, leafStart: 1.48, bloomStart: 2.16, phase: 0.92 },
      { x: -0.12, z: 0.18, stemHeight: 1.06, lean: 0.18, bloomScale: 0.92, stemStart: 1.0, leafStart: 1.6, bloomStart: 2.3, phase: 1.34 },
      { x: -0.06, z: -0.05, stemHeight: 0.74, lean: 0.08, bloomScale: 0.7, stemStart: 1.12, leafStart: 1.72, bloomStart: 2.44, phase: 1.72 },
      { x: 0.06, z: -0.05, stemHeight: 0.74, lean: -0.08, bloomScale: 0.7, stemStart: 1.18, leafStart: 1.78, bloomStart: 2.52, phase: 2.12 },
      { x: 0.12, z: 0.18, stemHeight: 1.06, lean: -0.18, bloomScale: 0.92, stemStart: 1.0, leafStart: 1.6, bloomStart: 2.3, phase: 2.5 },
      { x: 0.28, z: 0.09, stemHeight: 1.02, lean: -0.36, bloomScale: 0.98, stemStart: 0.88, leafStart: 1.48, bloomStart: 2.16, phase: 2.94 },
      { x: 0.4, z: 0.01, stemHeight: 0.98, lean: -0.52, bloomScale: 1.08, stemStart: 0.78, leafStart: 1.34, bloomStart: 2.02, phase: 3.36 },
      { x: 0.52, z: -0.08, stemHeight: 0.9, lean: -0.66, bloomScale: 1.18, stemStart: 0.68, leafStart: 1.24, bloomStart: 1.92, phase: 3.88 },
      { x: -0.18, z: -0.16, stemHeight: 0.84, lean: 0.22, bloomScale: 0.82, stemStart: 1.3, leafStart: 1.9, bloomStart: 2.64, phase: 4.2 },
      { x: 0.18, z: -0.16, stemHeight: 0.84, lean: -0.22, bloomScale: 0.82, stemStart: 1.3, leafStart: 1.9, bloomStart: 2.64, phase: 4.66 },
    ];
    const ornamentGroup = new THREE.Group();
    ornamentGroup.position.set(0, 0.08, -0.3);
    bouquetGroup.add(ornamentGroup);

    const ornamentSprites: THREE.Sprite[] = [];
    for (let i = 0; i < 18; i++) {
      const angle = (i / 18) * Math.PI * 2;
      const radius = 0.32 + Math.sin(i * 1.8) * 0.06;
      const sprite = new THREE.Sprite(
        new THREE.SpriteMaterial({
          map: glowTexture,
          transparent: true,
          opacity: 0,
          blending: THREE.AdditiveBlending,
          depthWrite: false,
          color: new THREE.Color(i % 3 === 0 ? "#ffe7a5" : i % 3 === 1 ? "#fff4cf" : "#f0c76f"),
        }),
      );
      sprite.position.set(
        Math.cos(angle) * radius,
        Math.sin(angle) * 0.2 - 0.02 + Math.cos(angle * 1.6) * 0.03,
        -0.18 + Math.sin(angle * 2.1) * 0.04,
      );
      const scale = 0.08 + (i % 4) * 0.018;
      sprite.scale.set(scale, scale, 1);
      sprite.userData = {
        baseScale: scale,
        baseY: sprite.position.y,
        delay: 1.95 + i * 0.05,
        phase: i * 0.4,
      };
      ornamentGroup.add(sprite);
      ornamentSprites.push(sprite);
    }

    bouquetGroup.userData.ornamentSprites = ornamentSprites;

    for (const spec of specs) {
      const branch = new THREE.Group();
      branch.position.set(spec.x, -1.06, spec.z);
      branch.rotation.z = spec.lean;
      bouquetGroup.add(branch);

      const stemGeometry = new THREE.CylinderGeometry(0.018, 0.009, spec.stemHeight, 10, 18);
      stemGeometry.translate(0, spec.stemHeight / 2, 0);
      const stem = new THREE.Mesh(
        stemGeometry,
        new THREE.MeshStandardMaterial({
          color: new THREE.Color("#f1d48a"),
          emissive: new THREE.Color("#9c7430"),
          emissiveIntensity: 0.45,
          metalness: 0.72,
          roughness: 0.22,
          transparent: true,
          opacity: 0,
        }),
      );
      stem.scale.y = 0.001;
      branch.add(stem);

      const leftLeaf = new THREE.Mesh(
        leafGeometry,
        new THREE.MeshStandardMaterial({
          color: new THREE.Color("#f9e6b7"),
          emissive: new THREE.Color("#8e6d2d"),
          emissiveIntensity: 0.35,
          metalness: 0.58,
          roughness: 0.38,
          transparent: true,
          opacity: 0,
          side: THREE.DoubleSide,
        }),
      );
      leftLeaf.position.set(-0.06, spec.stemHeight * 0.34, 0.03);
      leftLeaf.rotation.z = 0.15;
      leftLeaf.rotation.y = -0.25;
      leftLeaf.scale.setScalar(0.001);
      leftLeaf.userData = {
        endScale: 0.62,
        startRotZ: 0.15,
        endRotZ: 1.02,
        startRotY: -0.25,
        endRotY: 0.42,
      };
      branch.add(leftLeaf);

      const rightLeaf = new THREE.Mesh(
        leafGeometry,
        new THREE.MeshStandardMaterial({
          color: new THREE.Color("#fff0c4"),
          emissive: new THREE.Color("#8e6d2d"),
          emissiveIntensity: 0.35,
          metalness: 0.58,
          roughness: 0.38,
          transparent: true,
          opacity: 0,
          side: THREE.DoubleSide,
        }),
      );
      rightLeaf.position.set(0.05, spec.stemHeight * 0.46, -0.02);
      rightLeaf.rotation.z = -0.18;
      rightLeaf.rotation.y = 0.22;
      rightLeaf.scale.setScalar(0.001);
      rightLeaf.userData = {
        endScale: 0.56,
        startRotZ: -0.18,
        endRotZ: -0.94,
        startRotY: 0.22,
        endRotY: -0.38,
      };
      branch.add(rightLeaf);

      const bloom = createRoseBloomGroup(petalTexture, glowTexture, spec.bloomScale, spec.phase);
      bloom.position.set(0, spec.stemHeight + 0.04, 0);
      bloom.scale.setScalar(0.001);
      branch.add(bloom);

      branch.userData = {
        stem,
        leaves: [leftLeaf, rightLeaf],
        bloom,
        stemStart: spec.stemStart,
        leafStart: spec.leafStart,
        bloomStart: spec.bloomStart,
        stemHeight: spec.stemHeight,
        phase: spec.phase,
        baseLean: spec.lean,
      };
    }

    bouquetAura = new THREE.Sprite(
      new THREE.SpriteMaterial({
        map: createGlowTexture("#ffe7af"),
        transparent: true,
        opacity: 0,
        blending: THREE.AdditiveBlending,
        depthWrite: false,
      }),
    );
    bouquetAura.scale.set(2.2, 2.2, 1);
    bouquetAura.position.set(0, 0.16, -0.42);
    bouquetGroup.add(bouquetAura);

    bouquetReady = true;
    const container = containerRef.value;
    if (container) {
      applyResponsiveLayout(container.clientWidth, container.clientHeight);
    }
    if (showRose.value) {
      bouquetGroup.visible = true;
    }
  };
  image.src = roseReferenceUrl;
}

// ─── Build 3D card meshes: scattered start → heart shape ───
function buildCards(group: THREE.Group) {
  const count = messageCards.length;

  for (let i = 0; i < count; i++) {
    const card = messageCards[i];
    const tone = tones[i % tones.length];
    const t = (i / count) * Math.PI * 2;
    const h = heartXY(t);

    // Target position on heart
    const targetZ = Math.sin(i * 0.7) * 0.08;
    const targetX = h.x;
    const targetY = h.y;

    // Start position: scattered from edges of a large sphere
    const phi = Math.acos(2 * Math.random() - 1); // uniform sphere distribution
    const theta = Math.random() * Math.PI * 2;
    const r = 2.5 + Math.random() * 1.5; // 2.5 ~ 4.0 world units out
    const startX = r * Math.sin(phi) * Math.cos(theta);
    const startY = r * Math.sin(phi) * Math.sin(theta);
    const startZ = r * Math.cos(phi) * 0.5; // flatten Z a bit

    // Card mesh
    const tex = createCardTexture(card.text, tone);
    const geo = new THREE.PlaneGeometry(CARD_W, CARD_H);
    const mat = new THREE.MeshBasicMaterial({
      map: tex,
      transparent: true,
      opacity: 0,
      side: THREE.DoubleSide,
      depthWrite: false,
    });
    const mesh = new THREE.Mesh(geo, mat);

    // Start at scattered position
    mesh.position.set(startX, startY, startZ);

    // Random initial rotation (tumbling in from space)
    mesh.rotation.set(
      (Math.random() - 0.5) * Math.PI * 2,
      (Math.random() - 0.5) * Math.PI * 2,
      (Math.random() - 0.5) * Math.PI * 2,
    );

    // Store animation data
    (mesh as any)._cardIndex = i;
    (mesh as any)._phase = Math.random() * Math.PI * 2;
    (mesh as any)._depth = 0.5 + Math.abs(h.x) * 0.5;
    (mesh as any)._startPos = new THREE.Vector3(startX, startY, startZ);
    (mesh as any)._targetPos = new THREE.Vector3(targetX, targetY, targetZ);
    (mesh as any)._startRot = new THREE.Euler(
      (Math.random() - 0.5) * Math.PI * 2,
      (Math.random() - 0.5) * Math.PI * 2,
      (Math.random() - 0.5) * Math.PI * 2,
    );
    // Final rotation: face slightly outward
    const faceAngle = Math.atan2(h.y, h.x);
    (mesh as any)._targetRot = new THREE.Euler(0, Math.sin(i * 0.3) * 0.1, faceAngle * 0.08);
    // Stagger: each card has a slightly different start time
    (mesh as any)._stagger = (i / count) * 0.5 + Math.random() * 0.15;

    group.add(mesh);
    cardMeshes.push(mesh);

    // Glow halo sprite
    const glowTex = createGlowTexture(toneColors[tone].glow);
    const spriteMat = new THREE.SpriteMaterial({
      map: glowTex,
      transparent: true,
      opacity: 0,
      blending: THREE.AdditiveBlending,
      depthWrite: false,
    });
    const sprite = new THREE.Sprite(spriteMat);
    sprite.scale.set(CARD_W * 2.2, CARD_H * 2.2, 1);
    sprite.position.set(startX, startY, startZ - 0.02);
    (sprite as any)._phase = Math.random() * Math.PI * 2;
    (sprite as any)._startPos = new THREE.Vector3(startX, startY, startZ - 0.02);
    (sprite as any)._targetPos = new THREE.Vector3(targetX, targetY, targetZ - 0.02);
    (sprite as any)._stagger = (mesh as any)._stagger;
    group.add(sprite);
    glowSprites.push(sprite);
  }
}

// ─── Easing helpers ───
function easeOutQuart(x: number): number {
  return 1 - Math.pow(1 - x, 4);
}

function easeInOutCubic(x: number): number {
  return x < 0.5 ? 4 * x * x * x : 1 - Math.pow(-2 * x + 2, 3) / 2;
}

function easeOutBack(x: number): number {
  const c1 = 1.70158;
  const c3 = c1 + 1;
  return 1 + c3 * Math.pow(x - 1, 3) + c1 * Math.pow(x - 1, 2);
}

function lerpAngle(a: number, b: number, t: number): number {
  return a + (b - a) * t;
}

// ─── Golden rose geometry ───
function createRoseGeometry(): THREE.BufferGeometry {
  const vertices: number[] = [];
  const normals: number[] = [];
  const normalizedYs: number[] = [];
  const Y_MIN = -0.5; // bottom of stem
  const Y_MAX = 0.22; // top of tallest petal

  function pushVertex(x: number, y: number, z: number, nx: number, ny: number, nz: number) {
    vertices.push(x, y, z);
    normals.push(nx, ny, nz);
    normalizedYs.push((y - Y_MIN) / (Y_MAX - Y_MIN));
  }

  // Petals: 3 layers, each layer rotated
  const petalLayers = [
    { count: 7, rBase: 0.04, rTip: 0.22, height: 0.18, curve: 0.7, tilt: 0.55, twist: 0.0 },
    { count: 8, rBase: 0.03, rTip: 0.18, height: 0.14, curve: 0.6, tilt: 0.9, twist: 0.2 },
    { count: 9, rBase: 0.02, rTip: 0.14, height: 0.1, curve: 0.5, tilt: 1.2, twist: 0.4 },
  ];

  for (const layer of petalLayers) {
    for (let i = 0; i < layer.count; i++) {
      const angle = (i / layer.count) * Math.PI * 2 + layer.twist;
      const stepsR = 8;
      const stepsH = 12;

      for (let ri = 0; ri < stepsR; ri++) {
        for (let hi = 0; hi < stepsH; hi++) {
          const u0 = ri / stepsR;
          const u1 = (ri + 1) / stepsR;
          const v0 = hi / stepsH;
          const v1 = (hi + 1) / stepsH;

          const p00 = petalPoint(angle, u0, v0, layer);
          const p10 = petalPoint(angle, u1, v0, layer);
          const p01 = petalPoint(angle, u0, v1, layer);
          const p11 = petalPoint(angle, u1, v1, layer);

          const n00 = petalNormal(angle, u0, v0, layer);
          const n10 = petalNormal(angle, u1, v0, layer);
          const n01 = petalNormal(angle, u0, v1, layer);
          const n11 = petalNormal(angle, u1, v1, layer);

          pushVertex(p00[0], p00[1], p00[2], n00[0], n00[1], n00[2]);
          pushVertex(p10[0], p10[1], p10[2], n10[0], n10[1], n10[2]);
          pushVertex(p01[0], p01[1], p01[2], n01[0], n01[1], n01[2]);

          pushVertex(p10[0], p10[1], p10[2], n10[0], n10[1], n10[2]);
          pushVertex(p11[0], p11[1], p11[2], n11[0], n11[1], n11[2]);
          pushVertex(p01[0], p01[1], p01[2], n01[0], n01[1], n01[2]);
        }
      }
    }
  }

  // Stem
  const stemSegs = 16;
  const stemRings = 8;
  const stemR = 0.012;
  for (let i = 0; i < stemSegs; i++) {
    for (let j = 0; j < stemRings; j++) {
      const a0 = (i / stemSegs) * Math.PI * 2;
      const a1 = ((i + 1) / stemSegs) * Math.PI * 2;
      const y0 = -0.05 - (j / stemRings) * 0.45;
      const y1 = -0.05 - ((j + 1) / stemRings) * 0.45;

      pushVertex(Math.cos(a0) * stemR, y0, Math.sin(a0) * stemR, Math.cos(a0), 0, Math.sin(a0));
      pushVertex(Math.cos(a1) * stemR, y0, Math.sin(a1) * stemR, Math.cos(a1), 0, Math.sin(a1));
      pushVertex(Math.cos(a0) * stemR, y1, Math.sin(a0) * stemR, Math.cos(a0), 0, Math.sin(a0));

      pushVertex(Math.cos(a1) * stemR, y0, Math.sin(a1) * stemR, Math.cos(a1), 0, Math.sin(a1));
      pushVertex(Math.cos(a1) * stemR, y1, Math.sin(a1) * stemR, Math.cos(a1), 0, Math.sin(a1));
      pushVertex(Math.cos(a0) * stemR, y1, Math.sin(a0) * stemR, Math.cos(a0), 0, Math.sin(a0));
    }
  }

  const geo = new THREE.BufferGeometry();
  geo.setAttribute("position", new THREE.Float32BufferAttribute(vertices, 3));
  geo.setAttribute("normal", new THREE.Float32BufferAttribute(normals, 3));
  geo.setAttribute("normalizedY", new THREE.Float32BufferAttribute(normalizedYs, 1));
  return geo;
}

function petalPoint(
  angle: number,
  u: number,
  v: number,
  layer: { rBase: number; rTip: number; height: number; curve: number; tilt: number },
): number[] {
  const r = layer.rBase + (layer.rTip - layer.rBase) * u;
  const localX = r * Math.sin(v * Math.PI);
  const localY = r * Math.cos(v * Math.PI) * layer.curve * (1 - u * 0.5);
  const localZ = u * layer.height;

  const cosA = Math.cos(angle);
  const sinA = Math.sin(angle);
  const cosT = Math.cos(layer.tilt * u);
  const sinT = Math.sin(layer.tilt * u);

  const y2 = localY * cosT - localZ * sinT;
  const z2 = localY * sinT + localZ * cosT;

  return [cosA * localX, y2 + 0.02, sinA * localX + z2 * 0.3];
}

function petalNormal(
  angle: number,
  u: number,
  v: number,
  layer: { curve: number; tilt: number },
): number[] {
  const nx = Math.sin(v * Math.PI);
  const ny = Math.cos(v * Math.PI);
  const nz = 0.3;

  const cosT = Math.cos(layer.tilt * u);
  const sinT = Math.sin(layer.tilt * u);
  const cosA = Math.cos(angle);
  const sinA = Math.sin(angle);

  const ny2 = ny * cosT - nz * sinT;
  const nz2 = ny * sinT + nz * cosT;

  const ox = cosA * nx;
  const oy = ny2;
  const oz = sinA * nx + nz2 * 0.3;
  const len = Math.sqrt(ox * ox + oy * oy + oz * oz) || 1;
  return [ox / len, oy / len, oz / len];
}

// ─── Rose shaders ───
const roseVertexShader = `
  attribute float normalizedY;
  uniform float uTime;
  uniform float uGrowth;
  varying vec3 vNormal;
  varying vec3 vViewPos;
  varying float vY;

  void main() {
    vNormal = normalize(normalMatrix * normal);
    vY = position.y;

    float g = clamp(uGrowth, 0.0, 1.0);
    // Growth: vertices appear from bottom to top
    float growEdge = smoothstep(g - 0.15, g, normalizedY);
    float visible = 1.0 - growEdge;

    vec3 pos = position;

    // Slight upward stretch at the growing tip
    float tipGlow = smoothstep(g - 0.08, g, normalizedY) * (1.0 - smoothstep(g, g + 0.02, normalizedY));
    pos.y += tipGlow * 0.02;

    // Scale: hidden vertices collapse to center
    pos *= visible;

    // Gentle sway after fully grown
    if (g >= 0.98) {
      pos.x += sin(uTime * 0.8) * 0.003;
      pos.z += cos(uTime * 0.6) * 0.003;
    }

    vec4 mvPos = modelViewMatrix * vec4(pos, 1.0);
    vViewPos = -mvPos.xyz;
    gl_Position = projectionMatrix * mvPos;
  }
`;

const roseFragmentShader = `
  uniform float uTime;
  uniform float uGrowth;
  varying vec3 vNormal;
  varying vec3 vViewPos;
  varying float vY;

  void main() {
    vec3 N = normalize(vNormal);
    vec3 V = normalize(vViewPos);
    vec3 L = normalize(vec3(0.5, 1.0, 0.8));

    float diff = max(dot(N, L), 0.0);
    float spec1 = pow(max(dot(reflect(-L, N), V), 0.0), 32.0);
    float spec2 = pow(max(dot(reflect(-L, N), V), 0.0), 128.0);
    float fresnel = pow(1.0 - max(dot(N, V), 0.0), 3.0);

    vec3 deepGold = vec3(0.72, 0.45, 0.05);
    vec3 brightGold = vec3(1.0, 0.88, 0.4);
    vec3 highlight = vec3(1.0, 0.95, 0.8);

    float yFactor = smoothstep(-0.1, 0.2, vY);
    vec3 baseColor = mix(deepGold, brightGold, yFactor);

    vec3 color = baseColor * (0.3 + diff * 0.7);
    color += highlight * spec1 * 0.6;
    color += highlight * spec2 * 0.8;
    color += brightGold * fresnel * 0.4;

    // Sparkle on grown parts
    float sparkle = sin(vY * 40.0 + uTime * 2.0) * 0.5 + 0.5;
    sparkle *= sin(dot(N, vec3(1.0)) * 30.0 + uTime * 3.0) * 0.5 + 0.5;
    color += highlight * sparkle * 0.15;

    gl_FragColor = vec4(color, 1.0);
  }
`;

// ─── Text particle extraction ───
function extractTextParticles(text: string, count: number): Float32Array {
  const canvas = document.createElement("canvas");
  const cw = 2048;
  const ch = 512;
  canvas.width = cw;
  canvas.height = ch;
  const ctx = canvas.getContext("2d")!;
  ctx.clearRect(0, 0, cw, ch);
  ctx.fillStyle = "#fff";
  ctx.font = "bold 200px sans-serif";
  ctx.textAlign = "center";
  ctx.textBaseline = "middle";
  ctx.fillText(text, cw / 2, ch / 2);

  const imageData = ctx.getImageData(0, 0, cw, ch).data;
  const points: { x: number; y: number }[] = [];
  const step = 4;
  for (let y = 0; y < ch; y += step) {
    for (let x = 0; x < cw; x += step) {
      if (imageData[(y * cw + x) * 4 + 3] > 64) {
        points.push({ x, y });
      }
    }
  }

  const positions = new Float32Array(count * 3);
  const scale = 1.6; // text spread in world units
  for (let i = 0; i < count; i++) {
    const pt = points[i % points.length];
    const nx = (pt.x / cw - 0.5) * scale;
    const ny = -(pt.y / ch - 0.5) * scale * (ch / cw);
    positions[i * 3] = nx + (Math.random() - 0.5) * 0.012;
    positions[i * 3 + 1] = ny + (Math.random() - 0.5) * 0.012;
    positions[i * 3 + 2] = (Math.random() - 0.5) * 0.05;
  }
  return positions;
}

// ─── Background particle effects ───

function heartShape(t: number) {
  const x = 16 * Math.sin(t) ** 3;
  const y =
    13 * Math.cos(t) -
    5 * Math.cos(2 * t) -
    2 * Math.cos(3 * t) -
    Math.cos(4 * t);
  return { x: x / 16, y: y / 16 };
}

function generateHeartParticles(count: number) {
  const positions = new Float32Array(count * 3); // target heart positions
  const scattered = new Float32Array(count * 3); // random start positions
  const textPos = extractTextParticles("嗨 杨雨馨", count); // text positions
  const colors = new Float32Array(count * 3);
  const sizes = new Float32Array(count);
  const phases = new Float32Array(count);

  for (let i = 0; i < count; i++) {
    const t = (i / count) * Math.PI * 2;
    const heart = heartShape(t);
    const spread = 0.06 + Math.random() * 0.12;
    const angle = Math.random() * Math.PI * 2;
    const depth = (Math.random() - 0.5) * 0.3;

    // Target: on heart curve
    positions[i * 3] = heart.x + Math.cos(angle) * spread;
    positions[i * 3 + 1] = heart.y + Math.sin(angle) * spread;
    positions[i * 3 + 2] = depth;

    // Scattered: random sphere
    const phi = Math.acos(2 * Math.random() - 1);
    const theta = Math.random() * Math.PI * 2;
    const r = 2.0 + Math.random() * 2.5;
    scattered[i * 3] = r * Math.sin(phi) * Math.cos(theta);
    scattered[i * 3 + 1] = r * Math.sin(phi) * Math.sin(theta);
    scattered[i * 3 + 2] = r * Math.cos(phi) * 0.6;

    const hue = 0.85 + Math.random() * 0.15;
    const sat = 0.6 + Math.random() * 0.3;
    const light = 0.55 + Math.random() * 0.35;
    const color = new THREE.Color().setHSL(hue % 1, sat, light);
    colors[i * 3] = color.r;
    colors[i * 3 + 1] = color.g;
    colors[i * 3 + 2] = color.b;

    sizes[i] = 2.0 + Math.random() * 4.0;
    phases[i] = Math.random() * Math.PI * 2;
  }

  return { positions, scattered, textPos, colors, sizes, phases };
}

function generateSparkles(count: number) {
  const positions = new Float32Array(count * 3);
  const colors = new Float32Array(count * 3);
  const sizes = new Float32Array(count);
  const phases = new Float32Array(count);

  for (let i = 0; i < count; i++) {
    positions[i * 3] = (Math.random() - 0.5) * 6;
    positions[i * 3 + 1] = (Math.random() - 0.5) * 6;
    positions[i * 3 + 2] = (Math.random() - 0.5) * 4;

    const palette = [
      [1.0, 0.95, 0.98],
      [0.9, 0.92, 1.0],
      [1.0, 0.88, 0.95],
      [0.92, 0.88, 1.0],
      [1.0, 0.97, 0.85],
    ];
    const c = palette[i % palette.length];
    colors[i * 3] = c[0];
    colors[i * 3 + 1] = c[1];
    colors[i * 3 + 2] = c[2];

    sizes[i] = 1.0 + Math.random() * 3.0;
    phases[i] = Math.random() * Math.PI * 2;
  }

  return { positions, colors, sizes, phases };
}

function generateGlowOrbs(count: number) {
  const positions = new Float32Array(count * 3);
  const colors = new Float32Array(count * 3);
  const sizes = new Float32Array(count);
  const phases = new Float32Array(count);

  for (let i = 0; i < count; i++) {
    positions[i * 3] = (Math.random() - 0.5) * 8;
    positions[i * 3 + 1] = (Math.random() - 0.5) * 6;
    positions[i * 3 + 2] = -1 - Math.random() * 3;

    const palette = [
      [0.98, 0.55, 0.75],
      [0.7, 0.55, 1.0],
      [0.55, 0.75, 1.0],
      [1.0, 0.7, 0.5],
      [0.6, 0.9, 0.8],
    ];
    const c = palette[i % palette.length];
    colors[i * 3] = c[0];
    colors[i * 3 + 1] = c[1];
    colors[i * 3 + 2] = c[2];

    sizes[i] = 15.0 + Math.random() * 30.0;
    phases[i] = Math.random() * Math.PI * 2;
  }

  return { positions, colors, sizes, phases };
}

function generateShootingStars(count: number) {
  const positions = new Float32Array(count * 3);
  const colors = new Float32Array(count * 3);
  const sizes = new Float32Array(count);
  const velocities = new Float32Array(count * 3);
  const lifetimes = new Float32Array(count);

  for (let i = 0; i < count; i++) {
    resetShootingStar(positions, colors, sizes, velocities, lifetimes, i);
  }

  return { positions, colors, sizes, velocities, lifetimes };
}

function resetShootingStar(
  positions: Float32Array,
  colors: Float32Array,
  sizes: Float32Array,
  velocities: Float32Array,
  lifetimes: Float32Array,
  i: number,
) {
  positions[i * 3] = (Math.random() - 0.5) * 8;
  positions[i * 3 + 1] = 3 + Math.random() * 2;
  positions[i * 3 + 2] = -1 - Math.random() * 2;

  const speed = 0.01 + Math.random() * 0.015;
  velocities[i * 3] = (Math.random() - 0.5) * 0.01;
  velocities[i * 3 + 1] = -speed;
  velocities[i * 3 + 2] = 0;

  colors[i * 3] = 0.9 + Math.random() * 0.1;
  colors[i * 3 + 1] = 0.92 + Math.random() * 0.08;
  colors[i * 3 + 2] = 1.0;

  sizes[i] = 2.0 + Math.random() * 3.0;
  lifetimes[i] = 0;
}

// ─── Shaders ───
const heartVertexShader = `
  attribute float size;
  attribute float phase;
  attribute vec3 scattered;
  attribute vec3 textPosition;
  uniform float uTime;
  uniform float uScatteredProgress;
  uniform float uMorphProgress;
  varying vec3 vColor;
  varying float vAlpha;

  void main() {
    vColor = color;
    float pulse = sin(uTime * 0.8 + phase) * 0.5 + 0.5;
    vAlpha = 0.3 + pulse * 0.4;

    // Phase 1: scattered → text
    float sp = clamp(uScatteredProgress, 0.0, 1.0);
    float ep1 = 1.0 - pow(1.0 - sp, 3.0);
    vec3 pos = mix(scattered, textPosition, ep1);

    // Phase 2: text → heart
    float mp = clamp(uMorphProgress, 0.0, 1.0);
    float ep2 = 1.0 - pow(1.0 - mp, 3.0);
    pos = mix(pos, position, ep2);

    // Swirl during convergence
    float swirl = (1.0 - ep1) * 0.3 + (1.0 - ep2) * sp * 0.2;
    pos.x += cos(uTime * 1.5 + phase * 6.28) * swirl;
    pos.y += sin(uTime * 1.5 + phase * 6.28) * swirl;

    // Fade in
    vAlpha *= smoothstep(0.0, 0.3, sp);

    vec4 mvPosition = modelViewMatrix * vec4(pos, 1.0);
    gl_PointSize = size * (2.5 / -mvPosition.z) * (0.8 + pulse * 0.3);
    gl_Position = projectionMatrix * mvPosition;
  }
`;

const heartFragmentShader = `
  varying vec3 vColor;
  varying float vAlpha;

  void main() {
    float dist = length(gl_PointCoord - vec2(0.5));
    if (dist > 0.5) discard;

    float glow = 1.0 - smoothstep(0.0, 0.5, dist);
    glow = pow(glow, 1.5);

    gl_FragColor = vec4(vColor, glow * vAlpha);
  }
`;

const sparkleVertexShader = `
  attribute float size;
  attribute float phase;
  uniform float uTime;
  varying vec3 vColor;
  varying float vAlpha;

  void main() {
    vColor = color;
    float twinkle = sin(uTime * 2.0 + phase * 6.28) * 0.5 + 0.5;
    vAlpha = 0.3 + twinkle * 0.7;

    vec4 mvPosition = modelViewMatrix * vec4(position, 1.0);
    gl_PointSize = size * (2.0 / -mvPosition.z) * twinkle;
    gl_Position = projectionMatrix * mvPosition;
  }
`;

const sparkleFragmentShader = `
  varying vec3 vColor;
  varying float vAlpha;

  void main() {
    float dist = length(gl_PointCoord - vec2(0.5));
    if (dist > 0.5) discard;

    vec2 uv = gl_PointCoord - vec2(0.5);
    float cross = exp(-abs(uv.x) * 12.0) + exp(-abs(uv.y) * 12.0);
    float glow = exp(-dist * 6.0) + cross * 0.3;

    gl_FragColor = vec4(vColor, glow * vAlpha);
  }
`;

const glowVertexShader = `
  attribute float size;
  attribute float phase;
  uniform float uTime;
  varying vec3 vColor;
  varying float vAlpha;

  void main() {
    vColor = color;
    float breathe = sin(uTime * 0.3 + phase) * 0.5 + 0.5;
    vAlpha = 0.08 + breathe * 0.12;

    vec4 mvPosition = modelViewMatrix * vec4(position, 1.0);
    gl_PointSize = size * (3.0 / -mvPosition.z) * (0.7 + breathe * 0.3);
    gl_Position = projectionMatrix * mvPosition;
  }
`;

const glowFragmentShader = `
  varying vec3 vColor;
  varying float vAlpha;

  void main() {
    float dist = length(gl_PointCoord - vec2(0.5));
    float glow = exp(-dist * dist * 8.0);

    gl_FragColor = vec4(vColor, glow * vAlpha);
  }
`;

const shootingVertexShader = `
  attribute float size;
  varying vec3 vColor;

  void main() {
    vColor = color;

    vec4 mvPosition = modelViewMatrix * vec4(position, 1.0);
    gl_PointSize = size * (2.5 / -mvPosition.z);
    gl_Position = projectionMatrix * mvPosition;
  }
`;

const shootingFragmentShader = `
  varying vec3 vColor;

  void main() {
    float dist = length(gl_PointCoord - vec2(0.5));
    if (dist > 0.5) discard;

    float glow = exp(-dist * 5.0);
    gl_FragColor = vec4(vColor, glow);
  }
`;

// ─── Init ───
function init() {
  const container = containerRef.value;
  if (!container) return;

  const width = container.clientWidth;
  const height = container.clientHeight;

  renderer = new THREE.WebGLRenderer({ alpha: true, antialias: true });
  renderer.setSize(width, height);
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
  renderer.sortObjects = true;
  container.appendChild(renderer.domElement);

  scene = new THREE.Scene();
  camera = new THREE.PerspectiveCamera(60, width / height, 0.1, 100);
  camera.position.z = 1.8;

  const ambientLight = new THREE.AmbientLight(0xffffff, 1.2);
  scene.add(ambientLight);

  const keyLight = new THREE.DirectionalLight(0xfff2d8, 2.6);
  keyLight.position.set(1.8, 2.8, 2.2);
  scene.add(keyLight);

  const rimLight = new THREE.PointLight(0xffd995, 6, 5, 2);
  rimLight.position.set(0, 1.2, 1.6);
  scene.add(rimLight);

  // ── Heart group (cards + glows) ──
  heartGroup = new THREE.Group();
  scene.add(heartGroup);
  buildCards(heartGroup);

  // ── Heart particles ──
  const heartCount = 3000;
  const hd = generateHeartParticles(heartCount);
  const heartGeo = new THREE.BufferGeometry();
  heartGeo.setAttribute("position", new THREE.BufferAttribute(hd.positions, 3));
  heartGeo.setAttribute("scattered", new THREE.BufferAttribute(hd.scattered, 3));
  heartGeo.setAttribute("textPosition", new THREE.BufferAttribute(hd.textPos, 3));
  heartGeo.setAttribute("color", new THREE.BufferAttribute(hd.colors, 3));
  heartGeo.setAttribute("size", new THREE.BufferAttribute(hd.sizes, 1));
  heartGeo.setAttribute("phase", new THREE.BufferAttribute(hd.phases, 1));

  heartPoints = new THREE.Points(
    heartGeo,
    new THREE.ShaderMaterial({
      vertexShader: heartVertexShader,
      fragmentShader: heartFragmentShader,
      vertexColors: true,
      transparent: true,
      blending: THREE.AdditiveBlending,
      depthWrite: false,
      uniforms: {
        uTime: { value: 0 },
        uScatteredProgress: { value: 0 },
        uMorphProgress: { value: 0 },
      },
    }),
  );
  scene.add(heartPoints);

  // ── Sparkles ──
  const sparkleCount = 400;
  const sd = generateSparkles(sparkleCount);
  const sparkleGeo = new THREE.BufferGeometry();
  sparkleGeo.setAttribute("position", new THREE.BufferAttribute(sd.positions, 3));
  sparkleGeo.setAttribute("color", new THREE.BufferAttribute(sd.colors, 3));
  sparkleGeo.setAttribute("size", new THREE.BufferAttribute(sd.sizes, 1));
  sparkleGeo.setAttribute("phase", new THREE.BufferAttribute(sd.phases, 1));

  sparklePoints = new THREE.Points(
    sparkleGeo,
    new THREE.ShaderMaterial({
      vertexShader: sparkleVertexShader,
      fragmentShader: sparkleFragmentShader,
      vertexColors: true,
      transparent: true,
      blending: THREE.AdditiveBlending,
      depthWrite: false,
      uniforms: { uTime: { value: 0 } },
    }),
  );
  scene.add(sparklePoints);

  // ── Glow orbs ──
  const orbCount = 20;
  const od = generateGlowOrbs(orbCount);
  const orbGeo = new THREE.BufferGeometry();
  orbGeo.setAttribute("position", new THREE.BufferAttribute(od.positions, 3));
  orbGeo.setAttribute("color", new THREE.BufferAttribute(od.colors, 3));
  orbGeo.setAttribute("size", new THREE.BufferAttribute(od.sizes, 1));
  orbGeo.setAttribute("phase", new THREE.BufferAttribute(od.phases, 1));

  glowOrbs = new THREE.Points(
    orbGeo,
    new THREE.ShaderMaterial({
      vertexShader: glowVertexShader,
      fragmentShader: glowFragmentShader,
      vertexColors: true,
      transparent: true,
      blending: THREE.AdditiveBlending,
      depthWrite: false,
      uniforms: { uTime: { value: 0 } },
    }),
  );
  scene.add(glowOrbs);

  // ── Shooting stars ──
  const shootCount = 6;
  const std = generateShootingStars(shootCount);
  const shootGeo = new THREE.BufferGeometry();
  shootGeo.setAttribute("position", new THREE.BufferAttribute(std.positions, 3));
  shootGeo.setAttribute("color", new THREE.BufferAttribute(std.colors, 3));
  shootGeo.setAttribute("size", new THREE.BufferAttribute(std.sizes, 1));

  shootingStars = new THREE.Points(
    shootGeo,
    new THREE.ShaderMaterial({
      vertexShader: shootingVertexShader,
      fragmentShader: shootingFragmentShader,
      vertexColors: true,
      transparent: true,
      blending: THREE.AdditiveBlending,
      depthWrite: false,
    }),
  );
  (shootingStars as any)._velocities = std.velocities;
  (shootingStars as any)._lifetimes = std.lifetimes;
  scene.add(shootingStars);

  // ── Golden rose (hidden initially) ──
  const roseGeo = createRoseGeometry();
  const roseMat = new THREE.ShaderMaterial({
    vertexShader: roseVertexShader,
    fragmentShader: roseFragmentShader,
    side: THREE.DoubleSide,
    uniforms: {
      uTime: { value: 0 },
      uGrowth: { value: 0 },
    },
  });
  roseMesh = new THREE.Mesh(roseGeo, roseMat);
  roseMesh.visible = false;
  scene.add(roseMesh);

  // Rose glow
  const roseGlowTex = createGlowTexture("#ffd700");
  const roseGlowMat = new THREE.SpriteMaterial({
    map: roseGlowTex,
    transparent: true,
    opacity: 0,
    blending: THREE.AdditiveBlending,
    depthWrite: false,
  });
  roseGlow = new THREE.Sprite(roseGlowMat);
  roseGlow.scale.set(0.8, 0.8, 1);
  roseGlow.visible = false;
  scene.add(roseGlow);

  buildBouquet(scene);
  applyResponsiveLayout(width, height);

  // Events
  container.addEventListener("pointermove", onPointerMove);
  container.addEventListener("click", onClick);

  animate(0);
}

// ─── Pointer events ───
function onPointerMove(e: PointerEvent) {
  const container = containerRef.value;
  if (!container) return;
  const rect = container.getBoundingClientRect();
  pointer.x = ((e.clientX - rect.left) / rect.width) * 2 - 1;
  pointer.y = -((e.clientY - rect.top) / rect.height) * 2 + 1;
}

function onClick() {
  if (!camera || !scene) return;

  // During "text" phase, any click starts the morph
  if (animState === "text") {
    animState = "morphing";
    showHint.value = false;
    morphStartTime = performance.now() * 0.001;
    containerRef.value && (containerRef.value.style.cursor = "default");
    return;
  }

  // During "cards" phase, check for card clicks
  if (animState === "cards") {
    raycaster.setFromCamera(pointer, camera);
    const hits = raycaster.intersectObjects(cardMeshes);
    if (hits.length > 0) {
      const mesh = hits[0].object as THREE.Mesh;
      const idx = (mesh as any)._cardIndex as number;
      selectedCardText.value = messageCards[idx].text;
    }
  }
}

function closeOverlay() {
  selectedCardText.value = null;
}

function resetToCardsView() {
  const now = performance.now() * 0.001;
  animState = "cards";
  morphStartTime = now - CARD_DELAY - CARD_DURATION - 0.05;
  showRose.value = false;
  showRoseHint.value = false;
  showProposal.value = false;
  proposalLocked.value = false;
  proposalRejecting.value = false;
  selectedCardText.value = null;
  hoveredCard = null;

  if (heartPoints) {
    const mat = heartPoints.material as THREE.ShaderMaterial;
    mat.opacity = 1;
    mat.uniforms.uScatteredProgress.value = 1;
    mat.uniforms.uMorphProgress.value = 1;
  }
  if (sparklePoints) {
    (sparklePoints.material as THREE.ShaderMaterial).opacity = 1;
  }
  if (glowOrbs) {
    (glowOrbs.material as THREE.ShaderMaterial).opacity = 1;
  }
  if (shootingStars) {
    (shootingStars.material as THREE.ShaderMaterial).opacity = 1;
  }
  if (bouquetGroup) {
    bouquetGroup.visible = false;
  }
  if (bouquetAura) {
    (bouquetAura.material as THREE.SpriteMaterial).opacity = 0;
  }
  if (roseMesh) {
    roseMesh.visible = false;
  }
  if (roseGlow) {
    roseGlow.visible = false;
  }

  containerRef.value && (containerRef.value.style.cursor = "default");
}

function handleAccept() {
  if (proposalLocked.value) return;
  proposalLocked.value = true;
  disableRoseReplay.value = true;
  window.setTimeout(() => {
    resetToCardsView();
  }, 360);
}

function handleReject() {
  if (proposalLocked.value) return;
  proposalLocked.value = true;
  proposalRejecting.value = true;
  witherStartTime = performance.now() * 0.001;
  window.setTimeout(() => {
    showProposal.value = false;
  }, 2400);
}

function triggerRose() {
  if (animState !== "cards") return;
  animState = "rose";
  showRoseHint.value = false;
  showRose.value = true;
  showProposal.value = false;
  proposalLocked.value = false;
  proposalRejecting.value = false;
  selectedCardText.value = null;
  roseStartTime = performance.now() * 0.001;
  if (bouquetGroup && bouquetReady) bouquetGroup.visible = true;
  if (roseMesh) roseMesh.visible = false;
  if (roseGlow) roseGlow.visible = false;
}

// ─── Animation loop ───
const SCATTERED_DURATION = 3.0;  // scattered → text
const MORPH_DURATION = 2.5;      // text → heart
const CARD_DELAY = 1.5;          // after morph, wait before cards fly
const CARD_DURATION = 5.5;       // cards fly in

function animate(time: number) {
  animationId = requestAnimationFrame(animate);
  const t = time * 0.001;

  // ── State machine for particle phases ──
  if (animState === "scattered") {
    const p = Math.min(t / SCATTERED_DURATION, 1);
    if (heartPoints) {
      (heartPoints.material as THREE.ShaderMaterial).uniforms.uScatteredProgress.value = p;
    }
    if (p >= 1) {
      animState = "text";
      showHint.value = true;
      containerRef.value && (containerRef.value.style.cursor = "pointer");
    }
  }

  if (animState === "text") {
    // Particles at text position, gentle pulse
    if (heartPoints) {
      (heartPoints.material as THREE.ShaderMaterial).uniforms.uScatteredProgress.value = 1;
    }
  }

  if (animState === "morphing") {
    const morphTime = t - morphStartTime;
    const p = Math.min(morphTime / MORPH_DURATION, 1);
    if (heartPoints) {
      (heartPoints.material as THREE.ShaderMaterial).uniforms.uMorphProgress.value = p;
    }
    if (p >= 1) {
      animState = "cards";
      morphStartTime = t; // reuse as cards start time
    }
  }

  if (animState === "cards") {
    if (heartPoints) {
      (heartPoints.material as THREE.ShaderMaterial).uniforms.uMorphProgress.value = 1;
    }
  }

  // ── Card flying entrance (only in "cards" phase) ──
  const cardTime = (animState === "cards" || animState === "rose") ? Math.max(0, t - morphStartTime - CARD_DELAY) : 0;
  const globalProgress = Math.min(cardTime / CARD_DURATION, 1);
  const allArrived = globalProgress >= 1;

  // Show rose hint once cards arrive
  if (allArrived && animState === "cards" && !showRoseHint.value && !disableRoseReplay.value) {
    showRoseHint.value = true;
  }

  for (let i = 0; i < cardMeshes.length; i++) {
    const mesh = cardMeshes[i];
    const mat = mesh.material as THREE.MeshBasicMaterial;
    const cphase = (mesh as any)._phase as number;
    const depth = (mesh as any)._depth as number;
    const stagger = (mesh as any)._stagger as number;
    const startPos = (mesh as any)._startPos as THREE.Vector3;
    const targetPos = (mesh as any)._targetPos as THREE.Vector3;
    const startRot = (mesh as any)._startRot as THREE.Euler;
    const targetRot = (mesh as any)._targetRot as THREE.Euler;

    // Per-card progress with stagger
    const rawProgress = (cardTime / CARD_DURATION - stagger) / (1 - stagger);
    const progress = Math.max(0, Math.min(1, rawProgress));
    const easedPos = easeOutQuart(progress);
    const easedRot = easeInOutCubic(progress);

    if (progress < 1) {
      // ── Flying phase: interpolate from scattered → heart ──
      mesh.position.lerpVectors(startPos, targetPos, easedPos);

      // Add a spiral twist during flight
      const spiralAngle = (1 - progress) * Math.PI * 3;
      const spiralRadius = (1 - progress) * 0.15;
      mesh.position.x += Math.cos(spiralAngle + cphase) * spiralRadius;
      mesh.position.y += Math.sin(spiralAngle + cphase) * spiralRadius;

      // Rotation: tumble → settle
      mesh.rotation.x = lerpAngle(startRot.x, targetRot.x, easedRot);
      mesh.rotation.y = lerpAngle(startRot.y, targetRot.y, easedRot);
      mesh.rotation.z = lerpAngle(startRot.z, targetRot.z, easedRot);

      // Opacity: fade in quickly, then full
      const opacityProgress = Math.min(progress * 3, 1);
      const baseOp = opacityProgress * (0.6 + depth * 0.12);
      mat.opacity = Math.min(baseOp * 1.15, 1);

      // Scale: small → full with slight overshoot
      const scaleProgress = progress < 0.8
        ? easeOutQuart(progress / 0.8) * 1.08
        : 1.08 - (progress - 0.8) / 0.2 * 0.08;
      mesh.scale.setScalar(scaleProgress);

      // Mark arrival time for smooth transition
      if (progress >= 0.999) {
        (mesh as any)._arriveTime = t;
      }
    } else {
      // ── Arrived: rhythmic heartbeat + ripple wave ──
      mesh.position.set(targetPos.x, targetPos.y, targetPos.z);

      // Heartbeat: double-bump pulse (lub-dub)
      const heartbeat = Math.sin(t * 1.2) ** 2 * 0.5 + Math.sin(t * 2.4) ** 2 * 0.25;
      const beatScale = 1 + heartbeat * 0.06;
      mesh.scale.setScalar(beatScale);

      // Ripple wave: travels outward from center of heart
      const distFromCenter = Math.sqrt(targetPos.x * targetPos.x + targetPos.y * targetPos.y);
      const ripple = Math.sin(t * 2.0 - distFromCenter * 6.0) * 0.5 + 0.5;
      mesh.position.z = targetPos.z + ripple * 0.025;
      mesh.position.y += Math.sin(t * 0.8 + cphase + distFromCenter * 3.0) * 0.002;

      // Gentle rotation breathing
      const rotBreath = Math.sin(t * 0.5 + cphase) * 0.03 + ripple * 0.015;
      mesh.rotation.set(targetRot.x, targetRot.y + rotBreath * 0.5, targetRot.z + rotBreath);

      // Smooth brightness transition: flight glow fades out over 0.6s
      const arriveTime = (mesh as any)._arriveTime ?? t;
      const fadeT = Math.min((t - arriveTime) / 0.6, 1);
      const flightBoost = 1.15 * (1 - fadeT) + 1.0 * fadeT;
      const baseOp = (0.6 + depth * 0.12 + heartbeat * 0.06) * flightBoost;
      mat.opacity = Math.min(baseOp, 1);
    }
  }

  // ── Glow sprites: follow their cards ──
  for (let i = 0; i < glowSprites.length; i++) {
    const sprite = glowSprites[i];
    const mat = sprite.material as THREE.SpriteMaterial;
    const gphase = (sprite as any)._phase as number;
    const stagger = (sprite as any)._stagger as number;
    const startPos = (sprite as any)._startPos as THREE.Vector3;
    const targetPos = (sprite as any)._targetPos as THREE.Vector3;

    const rawProgress = (cardTime / CARD_DURATION - stagger) / (1 - stagger);
    const progress = Math.max(0, Math.min(1, rawProgress));
    const easedPos = easeOutQuart(progress);

    if (progress < 1) {
      sprite.position.lerpVectors(startPos, targetPos, easedPos);
      mat.opacity = Math.min(progress * 2, 1) * 0.3;
    } else {
      sprite.position.copy(targetPos);
      const sd = Math.sqrt(targetPos.x * targetPos.x + targetPos.y * targetPos.y);
      sprite.position.y += Math.sin(t * 0.8 + gphase + sd * 3.0) * 0.002;
      const heartbeatGlow = Math.sin(t * 1.2) ** 2 * 0.5 + Math.sin(t * 2.4) ** 2 * 0.25;
      mat.opacity = 0.15 + heartbeatGlow * 0.2;
    }

    const breathe = Math.sin(t * 0.5 + gphase) * 0.5 + 0.5;
    const s = 1.8 + breathe * 0.4;
    sprite.scale.set(CARD_W * s * 2.2, CARD_H * s * 2.2, 1);
  }

  // ── Hover raycasting (only after cards arrived) ──
  if (camera && allArrived) {
    raycaster.setFromCamera(pointer, camera);
    const hits = raycaster.intersectObjects(cardMeshes);
    const newHovered = hits.length > 0 ? (hits[0].object as THREE.Mesh) : null;

    if (hoveredCard && hoveredCard !== newHovered) {
      (hoveredCard.material as THREE.MeshBasicMaterial).opacity =
        0.6 + (hoveredCard as any)._depth * 0.12;
    }

    if (newHovered) {
      (newHovered.material as THREE.MeshBasicMaterial).opacity = 1;
      containerRef.value && (containerRef.value.style.cursor = "pointer");
    } else {
      containerRef.value && (containerRef.value.style.cursor = "default");
    }

    hoveredCard = newHovered;
  }

  // ── Heart particles rotation ──
  if (heartPoints) {
    // Smoothly ramp rotation intensity after morph completes
    const morphP = (heartPoints.material as THREE.ShaderMaterial).uniforms.uMorphProgress.value;
    const rotIntensity = 0.03 + morphP * 0.12;
    heartPoints.rotation.y = Math.sin(t * 0.2) * rotIntensity;
    heartPoints.rotation.z = Math.sin(t * 0.15) * 0.05;
    heartPoints.position.y = Math.sin(t * 0.4) * 0.05;
    (heartPoints.material as THREE.ShaderMaterial).uniforms.uTime.value = t;
  }

  // ── Sparkles drift ──
  if (sparklePoints) {
    const pos = sparklePoints.geometry.attributes.position.array as Float32Array;
    const vel = (sparklePoints as any)._velocities as Float32Array | undefined;
    if (!vel) {
      const count = pos.length / 3;
      const vels = new Float32Array(count * 3);
      for (let i = 0; i < count; i++) {
        vels[i * 3] = (Math.random() - 0.5) * 0.002;
        vels[i * 3 + 1] = (Math.random() - 0.5) * 0.002;
        vels[i * 3 + 2] = (Math.random() - 0.5) * 0.001;
      }
      (sparklePoints as any)._velocities = vels;
    } else {
      for (let i = 0; i < pos.length / 3; i++) {
        pos[i * 3] += vel[i * 3];
        pos[i * 3 + 1] += vel[i * 3 + 1];
        pos[i * 3 + 2] += vel[i * 3 + 2];
        if (pos[i * 3] > 3) pos[i * 3] = -3;
        if (pos[i * 3] < -3) pos[i * 3] = 3;
        if (pos[i * 3 + 1] > 3) pos[i * 3 + 1] = -3;
        if (pos[i * 3 + 1] < -3) pos[i * 3 + 1] = 3;
      }
      sparklePoints.geometry.attributes.position.needsUpdate = true;
    }
    (sparklePoints.material as THREE.ShaderMaterial).uniforms.uTime.value = t;
  }

  // ── Glow orbs ──
  if (glowOrbs) {
    const pos = glowOrbs.geometry.attributes.position.array as Float32Array;
    for (let i = 0; i < pos.length / 3; i++) {
      pos[i * 3] += Math.sin(t * 0.1 + i) * 0.0005;
      pos[i * 3 + 1] += Math.cos(t * 0.08 + i * 1.3) * 0.0005;
    }
    glowOrbs.geometry.attributes.position.needsUpdate = true;
    (glowOrbs.material as THREE.ShaderMaterial).uniforms.uTime.value = t;
  }

  // ── Shooting stars ──
  if (shootingStars) {
    const pos = shootingStars.geometry.attributes.position.array as Float32Array;
    const vel = (shootingStars as any)._velocities as Float32Array;
    const life = (shootingStars as any)._lifetimes as Float32Array;
    if (vel && life) {
      for (let i = 0; i < pos.length / 3; i++) {
        life[i] += 0.016;
        pos[i * 3] += vel[i * 3];
        pos[i * 3 + 1] += vel[i * 3 + 1];

        if (pos[i * 3 + 1] < -3.5 || life[i] > 3) {
          pos[i * 3] = (Math.random() - 0.5) * 8;
          pos[i * 3 + 1] = 3 + Math.random() * 2;
          vel[i * 3] = (Math.random() - 0.5) * 0.01;
          vel[i * 3 + 1] = -(0.01 + Math.random() * 0.015);
          life[i] = 0;
        }
      }
      shootingStars.geometry.attributes.position.needsUpdate = true;
    }
  }

  // ── Bouquet animation ──
  if (animState === "rose") {
    const roseTime = t - roseStartTime;
    const particleFade = Math.min(roseTime / 2.0, 1);
    if (heartPoints) {
      (heartPoints.material as THREE.ShaderMaterial).opacity = 1 - particleFade;
      (heartPoints.material as THREE.ShaderMaterial).uniforms.uMorphProgress.value = 1;
    }
    if (sparklePoints) {
      (sparklePoints.material as THREE.ShaderMaterial).opacity = 1 - particleFade * 0.6;
    }
    if (glowOrbs) {
      (glowOrbs.material as THREE.ShaderMaterial).opacity = 1 - particleFade * 0.72;
    }
    if (shootingStars) {
      (shootingStars.material as THREE.ShaderMaterial).opacity = 1 - particleFade * 0.85;
    }

    // Cards + glows fade out: 0.4s → 3s
    if (roseTime > 0.5) {
      const cardFade = Math.min((roseTime - 0.4) / 2.6, 1);
      const easedFade = 1 - (1 - cardFade) * (1 - cardFade); // ease-out quad
      for (const mesh of cardMeshes) {
        const baseOp = 0.6 + (mesh as any)._depth * 0.12;
        (mesh.material as THREE.MeshBasicMaterial).opacity = baseOp * (1 - easedFade);
      }
      for (const sprite of glowSprites) {
        (sprite.material as THREE.SpriteMaterial).opacity *= (1 - easedFade);
      }
    }

    if (bouquetGroup && bouquetReady) {
      bouquetGroup.visible = true;
      if (roseTime > 4.4 && !showProposal.value) {
        showProposal.value = true;
      }
      const witherProgress = proposalRejecting.value
        ? Math.min(Math.max((t - witherStartTime) / 2.8, 0), 1)
        : 0;

      const settle = Math.min(Math.max((roseTime - 3.4) / 1.2, 0), 1);
      bouquetGroup.rotation.y = Math.sin(t * 0.32) * 0.06 * settle;
      bouquetGroup.position.y =
        -0.2 +
        Math.sin(t * 0.8) * 0.01 * settle +
        witherProgress * 0.18;

      const ornamentSprites = bouquetGroup.userData.ornamentSprites as THREE.Sprite[] | undefined;
      if (ornamentSprites) {
        for (const sprite of ornamentSprites) {
          const delay = sprite.userData.delay as number;
          const phase = sprite.userData.phase as number;
          const baseScale = sprite.userData.baseScale as number;
          const baseY = sprite.userData.baseY as number;
          const progress = Math.min(Math.max((roseTime - delay) / 0.9, 0), 1);
          const breathe = Math.sin(t * 1.1 + phase) * 0.5 + 0.5;
          const scale = Math.max(0.001, baseScale * (0.8 + progress * 0.45 + breathe * 0.16));
          sprite.scale.set(scale, scale, 1);
          sprite.position.y = baseY + Math.sin(t * 0.9 + phase) * 0.016;
          (sprite.material as THREE.SpriteMaterial).opacity =
            progress * (0.16 + breathe * 0.18) * (1 - witherProgress);
        }
      }

      for (const child of bouquetGroup.children) {
        if (!(child instanceof THREE.Group)) continue;
        const branch = child as THREE.Group;
        const stem = branch.userData.stem as THREE.Mesh | undefined;
        const leaves = branch.userData.leaves as THREE.Mesh[] | undefined;
        const bloom = branch.userData.bloom as THREE.Group | undefined;
        const stemStart = branch.userData.stemStart as number | undefined;
        const leafStart = branch.userData.leafStart as number | undefined;
        const bloomStart = branch.userData.bloomStart as number | undefined;
        const stemHeight = branch.userData.stemHeight as number | undefined;
        const phase = branch.userData.phase as number | undefined;
        const baseLean = branch.userData.baseLean as number | undefined;

        if (stem && stemStart !== undefined) {
          const stemProgress = Math.min(Math.max((roseTime - stemStart) / 1.15, 0), 1);
          const stemEase = easeOutQuart(stemProgress);
          stem.scale.y = Math.max(0.001, stemEase);
          const stemMat = stem.material as THREE.MeshStandardMaterial;
          stemMat.opacity = stemProgress * (0.95 - witherProgress * 0.42);
          stemMat.emissiveIntensity = 0.45 * (1 - witherProgress * 0.85);
        }

        if (leaves && leafStart !== undefined) {
          const leafProgress = Math.min(Math.max((roseTime - leafStart) / 0.9, 0), 1);
          const leafEase = leafProgress > 0 ? easeOutBack(leafProgress) : 0;
          for (const leaf of leaves) {
            leaf.scale.setScalar(Math.max(0.001, (leaf.userData.endScale as number) * leafEase));
            leaf.rotation.z =
              (leaf.userData.startRotZ as number) +
              ((leaf.userData.endRotZ as number) - (leaf.userData.startRotZ as number)) * leafEase +
              witherProgress * 0.55 * Math.sign(leaf.userData.endRotZ as number);
            leaf.rotation.y =
              (leaf.userData.startRotY as number) +
              ((leaf.userData.endRotY as number) - (leaf.userData.startRotY as number)) * leafEase;
            const leafMat = leaf.material as THREE.MeshStandardMaterial;
            leafMat.opacity = leafProgress * (0.88 - witherProgress * 0.58);
            leafMat.emissiveIntensity = 0.35 * (1 - witherProgress * 0.8);
          }
        }

        if (bloom && bloomStart !== undefined && stemHeight !== undefined) {
          const bloomProgress = Math.min(Math.max((roseTime - bloomStart) / 1.15, 0), 1);
          const bloomEase = bloomProgress > 0 ? easeOutBack(bloomProgress) : 0;
          const finalScale = bloom.userData.finalScale as number;
          bloom.scale.setScalar(Math.max(0.001, finalScale * bloomEase * (1 - witherProgress * 0.58)));
          bloom.position.y =
            stemHeight +
            0.03 +
            (1 - bloomProgress) * 0.08 +
            Math.sin((phase ?? 0) + t * 0.7) * 0.004 * settle -
            witherProgress * 0.22;

          const petals = bloom.userData.petals as THREE.Mesh[];
          for (const petal of petals) {
            petal.rotation.x =
              (petal.userData.startRotX as number) +
              ((petal.userData.endRotX as number) - (petal.userData.startRotX as number)) * bloomEase -
              witherProgress * 0.5;
            petal.rotation.y =
              (petal.userData.startRotY as number) +
              ((petal.userData.endRotY as number) - (petal.userData.startRotY as number)) * bloomEase;
            petal.rotation.z =
              (petal.userData.startRotZ as number) +
              ((petal.userData.endRotZ as number) - (petal.userData.startRotZ as number)) * bloomEase;
            (petal.material as THREE.MeshBasicMaterial).opacity =
              bloomProgress * (0.96 - witherProgress * 0.76);
          }

          const bloomGlow = bloom.userData.glow as THREE.Sprite;
          if (bloomGlow) {
            const glowBreath = Math.sin(t * 1.1 + (phase ?? 0)) * 0.5 + 0.5;
            (bloomGlow.material as THREE.SpriteMaterial).opacity =
              bloomProgress * (0.14 + glowBreath * 0.16) * (1 - witherProgress);
            const glowScale = 0.7 + bloomProgress * 0.34 + glowBreath * 0.12;
            bloomGlow.scale.set(glowScale, glowScale, 1);
          }
        }

        if (baseLean !== undefined) {
          branch.rotation.z =
            baseLean +
            Math.sin(t * 0.7 + (phase ?? 0)) * 0.0008 * settle +
            Math.sign(baseLean || 1) * witherProgress * 0.34;
          branch.position.y = -1.06 + witherProgress * 0.12;
        }
      }

      if (bouquetAura) {
        const auraRise = Math.min(Math.max((roseTime - 2.0) / 1.5, 0), 1);
        const auraBreath = Math.sin(t * 0.95) * 0.5 + 0.5;
        bouquetAura.position.y = 0.12 + Math.sin(t * 0.5) * 0.01 * settle;
        bouquetAura.scale.set(
          1.7 + auraRise * 0.95 + auraBreath * 0.12,
          1.7 + auraRise * 0.95 + auraBreath * 0.12,
          1,
        );
        (bouquetAura.material as THREE.SpriteMaterial).opacity =
          auraRise * (0.14 + auraBreath * 0.12) * (1 - witherProgress);
      }
    }
  }

  // ── Heart group gentle rotation ──
  if (heartGroup) {
    heartGroup.rotation.y = Math.sin(t * 0.15) * 0.06;
    heartGroup.position.y = Math.sin(t * 0.3) * 0.02;
  }

  if (renderer && scene && camera) {
    renderer.render(scene, camera);
  }
}

function onResize() {
  const container = containerRef.value;
  if (!container || !renderer || !camera) return;

  const width = container.clientWidth;
  const height = container.clientHeight;
  renderer.setSize(width, height);
  camera.aspect = width / height;
  camera.updateProjectionMatrix();
  applyResponsiveLayout(width, height);
}

onMounted(() => {
  init();
  window.addEventListener("resize", onResize);
});

onBeforeUnmount(() => {
  cancelAnimationFrame(animationId);
  window.removeEventListener("resize", onResize);

  const container = containerRef.value;
  if (container) {
    container.removeEventListener("pointermove", onPointerMove);
    container.removeEventListener("click", onClick);
  }

  renderer?.dispose();
  heartPoints?.geometry.dispose();
  (heartPoints?.material as THREE.ShaderMaterial)?.dispose();
  sparklePoints?.geometry.dispose();
  (sparklePoints?.material as THREE.ShaderMaterial)?.dispose();
  glowOrbs?.geometry.dispose();
  (glowOrbs?.material as THREE.ShaderMaterial)?.dispose();
  shootingStars?.geometry.dispose();
  (shootingStars?.material as THREE.ShaderMaterial)?.dispose();

  for (const m of cardMeshes) {
    m.geometry.dispose();
    (m.material as THREE.MeshBasicMaterial).map?.dispose();
    (m.material as THREE.MeshBasicMaterial).dispose();
  }
  for (const s of glowSprites) {
    s.material.map?.dispose();
    s.material.dispose();
  }

  bouquetGroup?.traverse((child) => {
    const mesh = child as THREE.Mesh;
    if (mesh.geometry) {
      mesh.geometry.dispose?.();
    }

    const material = (mesh as any).material as THREE.Material | THREE.Material[] | undefined;
    if (Array.isArray(material)) {
      for (const item of material) {
        (item as any).map?.dispose?.();
        item.dispose?.();
      }
    } else if (material) {
      (material as any).map?.dispose?.();
      material.dispose?.();
    }
  });
});
</script>

<template>
  <div ref="containerRef" class="three-scene" />

  <transition name="hint-fade">
    <div v-if="showHint" class="click-hint">
      点击任意处继续
    </div>
  </transition>

  <transition name="hint-fade">
    <button
      v-if="showRoseHint && !showRose && !disableRoseReplay"
      class="rose-btn"
      @click="triggerRose"
    >
      点我
    </button>
  </transition>

  <transition name="proposal-fade">
    <div
      v-if="showProposal"
      class="proposal-stage"
      :class="{ 'proposal-stage--reject': proposalRejecting }"
    >
      <div class="proposal-flowers">
        <div
          v-for="(flower, index) in proposalFlowers"
          :key="`${flower.x}-${flower.y}-${index}`"
          class="proposal-flowers__item"
          :class="`proposal-flowers__item--${flower.hue}`"
          :style="{
            left: flower.x,
            top: flower.y,
            width: `${flower.size}px`,
            height: `${flower.size}px`,
            animationDelay: flower.delay,
            '--drop-delay': flower.delay,
          }"
        />
      </div>
      <div class="proposal-copy">
        <div class="proposal-copy__halo" />
        <div class="proposal-copy__sub">520 快乐</div>
        <div class="proposal-copy__main">你愿意做我女朋友吗</div>
        <div v-if="!proposalRejecting" class="proposal-actions">
          <button class="proposal-actions__btn proposal-actions__btn--yes" @click="handleAccept">
            愿意
          </button>
          <button class="proposal-actions__btn proposal-actions__btn--no" @click="handleReject">
            不愿意
          </button>
        </div>
      </div>
    </div>
  </transition>

  <transition name="overlay-fade">
    <div v-if="selectedCardText" class="card-overlay" @click.self="closeOverlay">
      <div class="card-overlay__inner">
        <button class="card-overlay__close" @click="closeOverlay">&times;</button>
        <div class="card-overlay__text">{{ selectedCardText }}</div>
      </div>
    </div>
  </transition>
</template>

<style scoped>
.three-scene {
  position: fixed;
  inset: 0;
  z-index: 0;
  pointer-events: auto;
}

.three-scene :deep(canvas) {
  display: block;
}

/* ─── Click hint ─── */
.click-hint {
  position: fixed;
  left: 50%;
  bottom: 60px;
  transform: translateX(-50%);
  padding: 12px 28px;
  border-radius: 999px;
  background: rgba(10, 16, 42, 0.45);
  border: 1px solid rgba(255, 255, 255, 0.18);
  color: rgba(235, 241, 255, 0.85);
  font-size: 15px;
  letter-spacing: 0.1em;
  backdrop-filter: blur(10px);
  animation: hint-pulse 2s ease-in-out infinite;
  z-index: 100;
  pointer-events: none;
}

@keyframes hint-pulse {
  0%, 100% { opacity: 0.7; }
  50% { opacity: 1; }
}

.hint-fade-enter-active,
.hint-fade-leave-active {
  transition: opacity 0.6s ease;
}

.hint-fade-enter-from,
.hint-fade-leave-to {
  opacity: 0;
}

/* ─── Rose button ─── */
.rose-btn {
  position: fixed;
  right: 40px;
  bottom: 40px;
  z-index: 100;
  padding: 14px 32px;
  border: 1px solid rgba(255, 215, 0, 0.5);
  border-radius: 999px;
  background: linear-gradient(135deg, rgba(180, 130, 0, 0.35), rgba(255, 215, 0, 0.2));
  color: #ffd700;
  font-size: 16px;
  font-weight: 700;
  letter-spacing: 0.15em;
  cursor: pointer;
  backdrop-filter: blur(10px);
  box-shadow:
    0 0 20px rgba(255, 215, 0, 0.2),
    inset 0 0 12px rgba(255, 215, 0, 0.1);
  animation: rose-btn-pulse 2s ease-in-out infinite;
  transition: transform 0.2s ease;
}

.rose-btn:hover {
  transform: scale(1.08);
  box-shadow:
    0 0 30px rgba(255, 215, 0, 0.35),
    inset 0 0 16px rgba(255, 215, 0, 0.15);
}

@keyframes rose-btn-pulse {
  0%, 100% { opacity: 0.8; box-shadow: 0 0 20px rgba(255, 215, 0, 0.2); }
  50% { opacity: 1; box-shadow: 0 0 30px rgba(255, 215, 0, 0.35); }
}

.proposal-stage {
  position: fixed;
  inset: 0;
  z-index: 180;
  display: grid;
  place-items: center;
  pointer-events: none;
  overflow: hidden;
  background:
    radial-gradient(circle at center, rgba(255, 238, 214, 0.06), transparent 24%),
    linear-gradient(180deg, rgba(26, 12, 18, 0.02), rgba(26, 12, 18, 0.12));
}

.proposal-stage::before {
  content: "";
  position: absolute;
  inset: -10%;
  background:
    radial-gradient(circle at 50% 50%, rgba(255, 210, 160, 0.12), transparent 16%),
    radial-gradient(circle at 50% 50%, rgba(255, 120, 150, 0.08), transparent 40%);
  filter: blur(16px);
  animation: proposal-glow 3.8s ease-in-out infinite;
}

.proposal-stage::after {
  content: "";
  position: absolute;
  left: 50%;
  top: 50%;
  width: min(72vw, 880px);
  height: min(34vh, 260px);
  transform: translate(-50%, -50%);
  border-radius: 999px;
  background:
    radial-gradient(circle at center, rgba(23, 8, 12, 0.74), rgba(23, 8, 12, 0.28) 52%, transparent 72%);
  filter: blur(10px);
}

.proposal-flowers {
  position: absolute;
  inset: 0;
}

.proposal-flowers__item {
  position: absolute;
  border-radius: 50%;
  background:
    radial-gradient(circle at 50% 50%, rgba(255, 250, 245, 0.98) 0 16%, transparent 17%),
    radial-gradient(circle at 25% 30%, rgba(255, 255, 255, 0.92) 0 10%, transparent 11%),
    radial-gradient(circle at 50% 12%, currentColor 0 19%, transparent 20%),
    radial-gradient(circle at 84% 32%, currentColor 0 20%, transparent 21%),
    radial-gradient(circle at 84% 72%, currentColor 0 20%, transparent 21%),
    radial-gradient(circle at 50% 90%, currentColor 0 21%, transparent 22%),
    radial-gradient(circle at 14% 72%, currentColor 0 20%, transparent 21%),
    radial-gradient(circle at 14% 32%, currentColor 0 20%, transparent 21%);
  box-shadow:
    0 18px 40px rgba(55, 12, 28, 0.18),
    0 0 24px rgba(255, 255, 255, 0.08);
  opacity: 0;
  transform: translate(-50%, -50%) scale(0.2) rotate(-18deg);
  filter: saturate(1.16);
  animation: bloom-burst 1.1s cubic-bezier(0.2, 0.9, 0.18, 1) forwards, flower-float 5.6s ease-in-out infinite;
}

.proposal-stage--reject .proposal-flowers__item {
  animation: flower-drop 1.9s cubic-bezier(0.2, 0.88, 0.28, 1) forwards;
  animation-delay: var(--drop-delay);
}

.proposal-flowers__item::before,
.proposal-flowers__item::after {
  content: "";
  position: absolute;
  inset: 50%;
  border-radius: 999px;
  background: linear-gradient(180deg, rgba(93, 119, 56, 0.56), rgba(60, 78, 35, 0.18));
  transform-origin: top center;
}

.proposal-flowers__item::before {
  width: 8%;
  height: 54%;
  transform: translate(-50%, -6%) rotate(8deg);
}

.proposal-flowers__item::after {
  width: 36%;
  height: 18%;
  transform: translate(-24%, 150%) rotate(-26deg);
  border-radius: 100% 0;
}

.proposal-flowers__item--rose {
  color: rgba(219, 60, 96, 0.92);
}

.proposal-flowers__item--pink {
  color: rgba(255, 162, 186, 0.94);
}

.proposal-flowers__item--peach {
  color: rgba(255, 196, 165, 0.94);
}

.proposal-flowers__item--cream {
  color: rgba(255, 246, 226, 0.96);
}

.proposal-copy {
  position: relative;
  z-index: 1;
  display: grid;
  justify-items: center;
  gap: 16px;
  padding: 28px 36px;
  pointer-events: auto;
  animation: proposal-rise 1.15s cubic-bezier(0.2, 0.9, 0.18, 1);
}

.proposal-stage--reject .proposal-copy {
  animation: proposal-fall 1.9s cubic-bezier(0.2, 0.88, 0.28, 1) forwards 0.35s;
}

.proposal-copy__halo {
  position: absolute;
  inset: 50% auto auto 50%;
  width: min(62vw, 760px);
  height: min(24vh, 190px);
  transform: translate(-50%, -50%);
  border-radius: 999px;
  background:
    radial-gradient(circle at center, rgba(255, 232, 214, 0.18), rgba(255, 232, 214, 0.05) 44%, transparent 70%);
  filter: blur(6px);
  z-index: -1;
}

.proposal-copy__sub {
  font-size: 18px;
  font-weight: 700;
  letter-spacing: 0.5em;
  color: rgba(255, 236, 220, 0.96);
  text-transform: uppercase;
  text-shadow:
    0 0 18px rgba(255, 206, 177, 0.28),
    0 2px 8px rgba(0, 0, 0, 0.34);
}

.proposal-copy__main {
  font-size: clamp(50px, 6vw, 92px);
  font-weight: 800;
  line-height: 1.14;
  letter-spacing: 0.1em;
  text-align: center;
  color: #fffaf6;
  -webkit-text-stroke: 1.2px rgba(118, 23, 45, 0.68);
  text-shadow:
    0 4px 0 rgba(111, 18, 39, 0.32),
    0 14px 34px rgba(122, 20, 52, 0.42),
    0 0 38px rgba(255, 208, 215, 0.36),
    0 0 80px rgba(255, 238, 223, 0.22);
  animation: proposal-text-pop 1.2s cubic-bezier(0.16, 0.84, 0.24, 1) both 0.18s;
}

.proposal-actions {
  display: flex;
  align-items: center;
  gap: 16px;
  margin-top: 12px;
}

.proposal-actions__btn {
  min-width: 120px;
  padding: 13px 24px;
  border-radius: 999px;
  border: 0;
  font-size: 17px;
  font-weight: 700;
  letter-spacing: 0.12em;
  cursor: pointer;
  transition:
    transform 0.22s ease,
    box-shadow 0.22s ease,
    background 0.22s ease;
}

.proposal-actions__btn:hover {
  transform: translateY(-3px) scale(1.04);
}

.proposal-actions__btn--yes {
  color: #fffaf6;
  background: linear-gradient(135deg, rgba(224, 58, 111, 0.96), rgba(255, 132, 164, 0.94));
  box-shadow:
    0 16px 34px rgba(162, 24, 72, 0.34),
    0 0 24px rgba(255, 196, 208, 0.24);
}

.proposal-actions__btn--no {
  color: rgba(255, 242, 235, 0.94);
  background: linear-gradient(135deg, rgba(88, 55, 66, 0.92), rgba(54, 30, 42, 0.94));
  box-shadow:
    0 14px 28px rgba(19, 8, 14, 0.28),
    inset 0 0 0 1px rgba(255, 255, 255, 0.08);
}

.proposal-stage--reject .proposal-actions {
  opacity: 0;
  transform: translateY(18px);
  transition: all 0.25s ease;
}

@keyframes proposal-rise {
  from {
    opacity: 0;
    transform: translateY(36px) scale(0.92);
  }
  to {
    opacity: 1;
    transform: translateY(0) scale(1);
  }
}

.proposal-fade-enter-active,
.proposal-fade-leave-active {
  transition: opacity 0.8s ease;
}

.proposal-fade-enter-from,
.proposal-fade-leave-to {
  opacity: 0;
}

@keyframes bloom-burst {
  0% {
    opacity: 0;
    transform: translate(-50%, -50%) scale(0.2) rotate(-18deg);
  }
  70% {
    opacity: 1;
    transform: translate(-50%, -50%) scale(1.08) rotate(6deg);
  }
  100% {
    opacity: 0.96;
    transform: translate(-50%, -50%) scale(1) rotate(0deg);
  }
}

@keyframes flower-float {
  0%, 100% {
    margin-top: 0;
  }
  50% {
    margin-top: -8px;
  }
}

@keyframes flower-drop {
  0% {
    opacity: 0.96;
    transform: translate(-50%, -50%) scale(1) rotate(0deg);
  }
  100% {
    opacity: 0;
    transform: translate(-50%, calc(-50% + 110vh)) scale(0.88) rotate(18deg);
  }
}

@keyframes proposal-glow {
  0%, 100% {
    transform: scale(0.96);
    opacity: 0.82;
  }
  50% {
    transform: scale(1.04);
    opacity: 1;
  }
}

@keyframes proposal-text-pop {
  0% {
    opacity: 0;
    transform: translateY(24px) scale(0.9);
    letter-spacing: 0.22em;
  }
  60% {
    opacity: 1;
    transform: translateY(-4px) scale(1.03);
  }
  100% {
    opacity: 1;
    transform: translateY(0) scale(1);
    letter-spacing: 0.1em;
  }
}

@keyframes proposal-fall {
  0% {
    opacity: 1;
    transform: translateY(0) scale(1);
  }
  100% {
    opacity: 0;
    transform: translateY(105vh) scale(0.92);
  }
}

/* ─── Selected card overlay ─── */
.card-overlay {
  position: fixed;
  inset: 0;
  z-index: 300;
  display: grid;
  place-items: center;
  background: rgba(8, 12, 32, 0.55);
  backdrop-filter: blur(16px);
  cursor: pointer;
}

.card-overlay__inner {
  position: relative;
  max-width: 420px;
  padding: 48px 40px;
  border-radius: 28px;
  border: 1px solid rgba(255, 255, 255, 0.22);
  background: linear-gradient(
    135deg,
    rgba(255, 233, 212, 0.92),
    rgba(239, 228, 255, 0.92),
    rgba(220, 243, 255, 0.92)
  );
  box-shadow:
    0 60px 160px rgba(6, 14, 45, 0.5),
    0 0 80px rgba(255, 255, 255, 0.25);
  text-align: center;
  cursor: default;
  animation: overlay-pop 0.4s cubic-bezier(0.2, 0.9, 0.18, 1);
}

.card-overlay__close {
  position: absolute;
  top: -14px;
  right: -14px;
  width: 40px;
  height: 40px;
  border: 0;
  border-radius: 999px;
  background: rgba(10, 16, 42, 0.88);
  box-shadow:
    0 10px 30px rgba(6, 14, 45, 0.45),
    0 0 24px rgba(255, 255, 255, 0.2);
  color: rgba(245, 247, 255, 0.95);
  cursor: pointer;
  font-size: 28px;
  line-height: 1;
}

.card-overlay__close:hover {
  background: rgba(31, 41, 84, 0.96);
}

.card-overlay__text {
  font-size: 36px;
  font-weight: 700;
  line-height: 1.35;
  color: rgba(41, 44, 59, 0.85);
  letter-spacing: 0.04em;
}

@keyframes overlay-pop {
  from {
    transform: scale(0.7);
    opacity: 0;
  }
  to {
    transform: scale(1);
    opacity: 1;
  }
}

.overlay-fade-enter-active,
.overlay-fade-leave-active {
  transition: opacity 0.25s ease;
}

.overlay-fade-enter-from,
.overlay-fade-leave-to {
  opacity: 0;
}

@media (max-width: 900px) {
  .click-hint {
    bottom: 34px;
    padding: 10px 20px;
    font-size: 13px;
  }

  .rose-btn {
    right: 24px;
    bottom: 24px;
    padding: 12px 24px;
    font-size: 15px;
  }

  .proposal-stage::after {
    width: min(84vw, 760px);
    height: min(28vh, 220px);
  }

  .proposal-copy {
    width: min(84vw, 760px);
    padding: 24px 24px;
  }

  .proposal-copy__main {
    font-size: clamp(42px, 6.8vw, 72px);
  }
}

@media (max-width: 640px) {
  .rose-btn {
    right: 20px;
    bottom: 24px;
  }

  .proposal-stage::after {
    width: min(92vw, 560px);
    height: min(24vh, 180px);
  }

  .proposal-copy {
    width: min(90vw, 520px);
    gap: 12px;
    padding: 18px 16px;
  }

  .proposal-actions {
    gap: 10px;
    width: 100%;
  }

  .proposal-actions__btn {
    min-width: 0;
    flex: 1;
    font-size: 15px;
    padding: 12px 16px;
  }

  .proposal-copy__sub {
    font-size: 14px;
    letter-spacing: 0.28em;
  }

  .proposal-copy__main {
    font-size: clamp(32px, 9.2vw, 50px);
    -webkit-text-stroke: 0.9px rgba(118, 23, 45, 0.62);
  }
}

@media (max-width: 480px) {
  .click-hint {
    bottom: 22px;
    width: calc(100vw - 32px);
    text-align: center;
    letter-spacing: 0.08em;
  }

  .proposal-stage::before {
    inset: -18%;
    filter: blur(14px);
  }

  .proposal-stage::after {
    width: min(96vw, 420px);
    height: 160px;
  }

  .proposal-copy {
    width: calc(100vw - 24px);
    gap: 10px;
    padding: 18px 14px 16px;
  }

  .proposal-copy__halo {
    width: 92vw;
    height: 150px;
  }

  .proposal-copy__sub {
    font-size: 12px;
    letter-spacing: 0.22em;
  }

  .proposal-copy__main {
    font-size: clamp(28px, 8.6vw, 40px);
    line-height: 1.2;
    letter-spacing: 0.06em;
  }

  .proposal-actions {
    flex-direction: column;
    width: min(280px, 100%);
    gap: 10px;
  }

  .proposal-actions__btn {
    width: 100%;
    padding: 12px 14px;
    font-size: 14px;
  }

  .card-overlay__inner {
    width: calc(100vw - 28px);
    padding: 36px 24px;
  }

  .card-overlay__text {
    font-size: 28px;
  }
}
</style>
