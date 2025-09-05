import React, { useEffect, useRef, useState } from "react";

// --- Mini Minecraft-like (2D) ---------------------------------------------
// Features: air + land blocks, mountains via noise, scattered trees,
//           controllable player with gravity & collisions, camera follow.
// Controls: A/D or ◀/▶ to move, W/Space/▲ to jump, R to regenerate world.
// ---------------------------------------------------------------------------

// Tailwind is available globally in the canvas preview.

// Tile and world settings
const TILE = 32; // pixels per tile
const WORLD_WIDTH = 256; // tiles horizontally
const WORLD_HEIGHT = 96; // tiles vertically
const SKY_COLOR = "#87ceeb"; // light sky blue

// Tile IDs
const T = {
  AIR: 0,
  GRASS: 1,
  DIRT: 2,
  STONE: 3,
  WOOD: 4,
  LEAF: 5,
  WATER: 6,
};

// Tile palette (simple flat colors for performance)
const PALETTE = {
  [T.AIR]: SKY_COLOR,
  [T.GRASS]: "#4caf50",
  [T.DIRT]: "#8d6e63",
  [T.STONE]: "#607d8b",
  [T.WOOD]: "#795548",
  [T.LEAF]: "#2e7d32",
  [T.WATER]: "#4fc3f7",
};

// Seedable PRNG (Mulberry32)
function mulberry32(seed) {
  return function () {
    let t = (seed += 0x6d2b79f5);
    t = Math.imul(t ^ (t >>> 15), t | 1);
    t ^= t + Math.imul(t ^ (t >>> 7), t | 61);
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296;
  };
}

// Smooth interpolation
function smoothstep(t) {
  return t * t * (3 - 2 * t);
}

// 1D value noise with octaves
function makeNoise1D(seed, frequency = 0.01, octaves = 4, persistence = 0.5) {
  const rand = mulberry32(seed | 0);
  const gradients = new Map();
  const grad = (x) => {
    const xi = Math.floor(x);
    if (!gradients.has(xi)) gradients.set(xi, rand() * 2 - 1); // [-1,1]
    return gradients.get(xi);
  };
  return (x) => {
    let amp = 1;
    let freq = frequency;
    let sum = 0;
    let norm = 0;
    for (let o = 0; o < octaves; o++) {
      const xi = Math.floor(x * freq);
      const xf = x * freq - xi;
      const g0 = grad(xi);
      const g1 = grad(xi + 1);
      const v0 = g0 * xf;
      const v1 = g1 * (xf - 1);
      const t = smoothstep(xf);
      const n = (1 - t) * v0 + t * v1; // lerp
      sum += n * amp;
      norm += amp;
      amp *= persistence;
      freq *= 2;
    }
    return sum / (norm || 1);
  };
}

// World generation -----------------------------------------------------------
function generateWorld(seed) {
  const rand = mulberry32(seed);
  const noise = makeNoise1D(seed ^ 0x9e3779b9, 0.006, 5, 0.55);
  const world = new Uint8Array(WORLD_WIDTH * WORLD_HEIGHT);

  // Heightmap + mountains
  const base = Math.floor(WORLD_HEIGHT * 0.45);
  for (let x = 0; x < WORLD_WIDTH; x++) {
    const n = noise(x);
    const mountain = Math.floor((n * 0.5 + 0.5) * (WORLD_HEIGHT * 0.25));
    const h = base + mountain; // ground level (0 at top)
    for (let y = 0; y < WORLD_HEIGHT; y++) {
      const idx = y * WORLD_WIDTH + x;
      if (y < h) {
        world[idx] = T.AIR; // sky
      } else if (y === h) {
        world[idx] = T.GRASS; // surface
      } else if (y > h && y < h + 4) {
        world[idx] = T.DIRT; // dirt layer
      } else {
        world[idx] = T.STONE; // stone deeper
      }
    }
  }

  // Carve lakes in low areas
  for (let x = 2; x < WORLD_WIDTH - 2; x++) {
    // find ground height
    let groundY = -1;
    for (let y = 0; y < WORLD_HEIGHT; y++) {
      if (get(world, x, y) !== T.AIR) {
        groundY = y;
        break;
      }
    }
    if (groundY > 0 && groundY < WORLD_HEIGHT * 0.55 && rand() < 0.06) {
      const width = 3 + Math.floor(rand() * 6);
      for (let dx = -width; dx <= width; dx++) {
        const gx = x + dx;
        if (gx < 1 || gx >= WORLD_WIDTH - 1) continue;
        const depth = Math.floor((1 - Math.abs(dx) / (width + 0.001)) * 3 + 1);
        for (let d = 0; d < depth; d++) {
          const gy = groundY + d;
          set(world, gx, gy, T.WATER);
        }
      }
    }
  }

  // Trees on gentle slopes
  for (let x = 2; x < WORLD_WIDTH - 2; x++) {
    const gy = surfaceY(world, x);
    const gyL = surfaceY(world, x - 1);
    const gyR = surfaceY(world, x + 1);
    const slope = Math.abs(gyL - gy) + Math.abs(gyR - gy);
    if (
      gy > 0 &&
      slope <= 2 &&
      get(world, x, gy) === T.GRASS &&
      rand() < 0.08
    ) {
      placeTree(world, x, gy - 1, rand);
    }
  }

  return world;
}

function get(world, x, y) {
  if (x < 0 || y < 0 || x >= WORLD_WIDTH || y >= WORLD_HEIGHT) return T.STONE; // treat out of bounds as solid
  return world[y * WORLD_WIDTH + x];
}
function set(world, x, y, v) {
  if (x < 0 || y < 0 || x >= WORLD_WIDTH || y >= WORLD_HEIGHT) return;
  world[y * WORLD_WIDTH + x] = v;
}
function surfaceY(world, x) {
  for (let y = 0; y < WORLD_HEIGHT; y++) {
    if (get(world, x, y) !== T.AIR) return y;
  }
  return WORLD_HEIGHT - 1;
}

function placeTree(world, x, y, rand) {
  // trunk
  const h = 4 + Math.floor(rand() * 3);
  for (let i = 0; i < h; i++) set(world, x, y - i, T.WOOD);
  const topY = y - h;
  // leaves (3x3 blob + cap)
  for (let dx = -2; dx <= 2; dx++) {
    for (let dy = -1; dy <= 2; dy++) {
      if (Math.abs(dx) + Math.abs(dy) <= 3) set(world, x + dx, topY + dy, T.LEAF);
    }
  }
  set(world, x, topY - 1, T.LEAF);
}

// Player --------------------------------------------------------------------
class Player {
  constructor(x, y) {
    this.x = x; // in pixels
    this.y = y;
    this.vx = 0;
    this.vy = 0;
    this.w = TILE * 0.8;
    this.h = TILE * 1.6;
    this.onGround = false;
    this.facing = 1;
  }
}

function rectVsTilemapCollide(world, rx, ry, rw, rh) {
  // returns resolution offsets {dx, dy} to push rect out of solids
  const minX = Math.floor((rx) / TILE) - 1;
  const maxX = Math.floor((rx + rw) / TILE) + 1;
  const minY = Math.floor((ry) / TILE) - 1;
  const maxY = Math.floor((ry + rh) / TILE) + 1;
  let dxAccum = 0, dyAccum = 0;
  let onGround = false;

  const isSolid = (t) => t !== T.AIR && t !== T.WATER && t !== T.LEAF; // leaves are passable; water is non-solid this simple demo

  // Sweep on X then Y for stability
  // X axis
  let newRx = rx;
  if (dxAccum !== 0) {}
  for (let ty = minY; ty <= maxY; ty++) {
    for (let tx = minX; tx <= maxX; tx++) {
      const t = get(world, tx, ty);
      if (!isSolid(t)) continue;
      const bx = tx * TILE;
      const by = ty * TILE;
      const bw = TILE;
      const bh = TILE;
      if (
        newRx < bx + bw &&
        newRx + rw > bx &&
        ry < by + bh &&
        ry + rh > by
      ) {
        const left = bx - (newRx + rw);
        const right = (bx + bw) - newRx;
        const resolve = Math.abs(left) < Math.abs(right) ? left : right;
        newRx += resolve;
        dxAccum += resolve;
      }
    }
  }

  // Y axis
  let newRy = ry;
  for (let ty = minY; ty <= maxY; ty++) {
    for (let tx = minX; tx <= maxX; tx++) {
      const t = get(world, tx, ty);
      if (!isSolid(t)) continue;
      const bx = tx * TILE;
      const by = ty * TILE;
      const bw = TILE;
      const bh = TILE;
      if (
        newRx < bx + bw &&
        newRx + rw > bx &&
        newRy < by + bh &&
        newRy + rh > by
      ) {
        const top = by - (newRy + rh);
        const bottom = (by + bh) - newRy;
        const resolve = Math.abs(top) < Math.abs(bottom) ? top : bottom;
        newRy += resolve;
        dyAccum += resolve;
        if (resolve < 0) onGround = true; // collided from above
      }
    }
  }

  return { dx: dxAccum, dy: dyAccum, onGround };
}

// Rendering -----------------------------------------------------------------
function drawWorld(ctx, world, camX, camY, viewW, viewH) {
  const startX = Math.floor(camX / TILE);
  const startY = Math.floor(camY / TILE);
  const endX = Math.ceil((camX + viewW) / TILE);
  const endY = Math.ceil((camY + viewH) / TILE);

  // Sky background
  ctx.fillStyle = SKY_COLOR;
  ctx.fillRect(0, 0, viewW, viewH);

  for (let y = startY; y <= endY; y++) {
    for (let x = startX; x <= endX; x++) {
      const t = get(world, x, y);
      if (t === T.AIR) continue;
      const sx = x * TILE - camX;
      const sy = y * TILE - camY;
      ctx.fillStyle = PALETTE[t] || "#000";
      ctx.fillRect(sx, sy, TILE, TILE);
      // simple shading lines on stone/dirt/wood
      if (t === T.STONE || t === T.DIRT || t === T.WOOD) {
        ctx.globalAlpha = 0.15;
        ctx.fillStyle = "#000";
        ctx.fillRect(sx, sy, TILE, TILE * 0.1);
        ctx.globalAlpha = 1;
      }
    }
  }
}

function drawPlayer(ctx, p, camX, camY) {
  const px = Math.floor(p.x - camX);
  const py = Math.floor(p.y - camY);
  // body
  ctx.fillStyle = "#222";
  ctx.fillRect(px, py, p.w, p.h);
  // face visor
  ctx.fillStyle = "#cfd8dc";
  const visorW = p.w * 0.6;
  const visorH = p.h * 0.25;
  const vx = px + (p.facing > 0 ? p.w * 0.3 : p.w * 0.1);
  const vy = py + p.h * 0.2;
  ctx.fillRect(vx, vy, visorW, visorH);
}

// Input ---------------------------------------------------------------------
function useControls() {
  const keys = useRef(new Set());
  useEffect(() => {
    const down = (e) => keys.current.add(e.key.toLowerCase());
    const up = (e) => keys.current.delete(e.key.toLowerCase());
    window.addEventListener("keydown", down);
    window.addEventListener("keyup", up);
    return () => {
      window.removeEventListener("keydown", down);
      window.removeEventListener("keyup", up);
    };
  }, []);
  return keys;
}

// Component -----------------------------------------------------------------
export default function MiniMinecraft() {
  const canvasRef = useRef(null);
  const [seed, setSeed] = useState(() => Math.floor(Math.random() * 1e9));
  const [world, setWorld] = useState(() => generateWorld(seed));
  const [debug, setDebug] = useState({ fps: 0, pos: [0, 0] });
  const keys = useControls();

  // regenerate world when seed changes
  useEffect(() => {
    setWorld(generateWorld(seed));
  }, [seed]);

  useEffect(() => {
    const canvas = canvasRef.current;
    const ctx = canvas.getContext("2d");

    // Responsive sizing
    const resize = () => {
      const rect = canvas.parentElement.getBoundingClientRect();
      const dpr = Math.min(window.devicePixelRatio || 1, 2);
      canvas.width = Math.floor(rect.width * dpr);
      canvas.height = Math.floor((rect.height || 600) * dpr);
      canvas.style.width = rect.width + "px";
      canvas.style.height = (rect.height || 600) + "px";
      ctx.setTransform(dpr, 0, 0, dpr, 0, 0); // draw in CSS pixels
    };
    resize();
    window.addEventListener("resize", resize);

    const player = new Player(10 * TILE, 10 * TILE);
    const camera = { x: 0, y: 0 };

    let last = performance.now();
    let acc = 0;
    let frames = 0;
    let fpsTimer = 0;

    const maxVel = 12; // px/frame cap to avoid tunneling

    function step(now) {
      const dt = Math.min(1 / 30, (now - last) / 1000); // cap dt
      last = now;

      // Controls
      const k = keys.current;
      const left = k.has("a") || k.has("arrowleft");
      const right = k.has("d") || k.has("arrowright");
      const jump = k.has("w") || k.has(" ") || k.has("arrowup");
      const regen = k.has("r");
      if (regen) {
        setSeed(Math.floor(Math.random() * 1e9));
        keys.current.delete("r");
      }

      const moveSpeed = 220; // px/s
      const gravity = 1400; // px/s^2
      const jumpVel = 530; // px/s
      const friction = 0.85;

      // Horizontal
      if (left && !right) {
        player.vx = -moveSpeed;
        player.facing = -1;
      } else if (right && !left) {
        player.vx = moveSpeed;
        player.facing = 1;
      } else {
        player.vx *= friction;
        if (Math.abs(player.vx) < 5) player.vx = 0;
      }

      // Vertical physics
      player.vy += gravity * dt;
      if (jump && player.onGround) {
        player.vy = -jumpVel;
      }

      // Integrate with clamping
      const dx = Math.max(-maxVel, Math.min(maxVel, player.vx * dt));
      const dy = Math.max(-maxVel, Math.min(maxVel, player.vy * dt));

      let rx = player.x + dx;
      let ry = player.y + dy;
      const res = rectVsTilemapCollide(world, rx, ry, player.w, player.h);
      rx += res.dx;
      ry += res.dy;
      player.onGround = res.onGround && player.vy >= 0;
      if (res.dy !== 0) player.vy = 0;

      player.x = rx;
      player.y = ry;

      // Keep player within world bounds
      player.x = Math.max(0, Math.min(player.x, WORLD_WIDTH * TILE - player.w));
      player.y = Math.max(0, Math.min(player.y, WORLD_HEIGHT * TILE - player.h));

      // Camera follow with deadzone
      const viewW = canvas.width / (window.devicePixelRatio || 1);
      const viewH = canvas.height / (window.devicePixelRatio || 1);
      const deadX = viewW * 0.3;
      const deadY = viewH * 0.35;
      const targetX = player.x + player.w / 2 - viewW / 2;
      const targetY = player.y + player.h / 2 - viewH / 2;
      camera.x += (targetX - camera.x) * 0.15;
      camera.y += (targetY - camera.y) * 0.15;
      camera.x = Math.max(0, Math.min(camera.x, WORLD_WIDTH * TILE - viewW));
      camera.y = Math.max(0, Math.min(camera.y, WORLD_HEIGHT * TILE - viewH));

      // Render
      drawWorld(ctx, world, camera.x, camera.y, viewW, viewH);
      drawPlayer(ctx, player, camera.x, camera.y);

      // UI overlay
      ctx.fillStyle = "rgba(0,0,0,0.4)";
      ctx.fillRect(12, 12, 290, 92);
      ctx.fillStyle = "#fff";
      ctx.font = "14px ui-sans-serif, system-ui, -apple-system, Segoe UI";
      ctx.fillText("Mini Minecraft-like", 24, 34);
      ctx.fillText("A/D or ◀/▶ move  •  W/Space/▲ jump  •  R regenerate", 24, 54);
      ctx.fillText(
        `Pos: ${(player.x / TILE).toFixed(1)}, ${(player.y / TILE).toFixed(1)}  •  FPS: ${debug.fps.toFixed(
          0
        )}`,
        24,
        74
      );

      // FPS calc
      frames++;
      fpsTimer += dt;
      if (fpsTimer >= 0.5) {
        setDebug({ fps: (frames / fpsTimer), pos: [player.x, player.y] });
        frames = 0;
        fpsTimer = 0;
      }

      requestAnimationFrame(step);
    }

    const raf = requestAnimationFrame(step);
    return () => {
      window.removeEventListener("resize", resize);
      cancelAnimationFrame(raf);
    };
  }, [world]);

  return (
    <div className="w-full h-[70vh] md:h-[80vh] bg-slate-900 relative rounded-2xl overflow-hidden shadow-xl">
      <canvas ref={canvasRef} className="block w-full h-full"/>

      {/* Top bar */}
      <div className="absolute top-3 left-3 right-3 flex items-center gap-3">
        <div className="px-3 py-1.5 rounded-2xl bg-white/80 text-slate-800 text-sm shadow">
          Seed: <span className="font-mono">{seed}</span>
        </div>
        <button
          onClick={() => setSeed(Math.floor(Math.random() * 1e9))}
          className="px-3 py-1.5 rounded-2xl bg-emerald-500 text-white text-sm shadow hover:bg-emerald-600"
        >
          Regenerate World (R)
        </button>
        <div className="ml-auto px-3 py-1.5 rounded-2xl bg-white/80 text-slate-800 text-sm shadow">
          Blocks: Air, Grass, Dirt, Stone, Wood, Leaf, Water
        </div>
      </div>

      {/* Footer help */}
      <div className="absolute bottom-3 left-3 right-3 text-white/90 text-sm flex flex-wrap gap-2">
        <span className="px-2 py-1 rounded-lg bg-black/40">Move: A/D or ◀/▶</span>
        <span className="px-2 py-1 rounded-lg bg-black/40">Jump: W / Space / ▲</span>
        <span className="px-2 py-1 rounded-lg bg-black/40">Regenerate:
