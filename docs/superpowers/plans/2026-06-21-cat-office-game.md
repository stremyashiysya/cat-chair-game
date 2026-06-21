# Cat Office Chaos — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Single-file HTML5 game — кот из стикера ходит по офису, игрок зажимает предметы, кот дубасит их, они летят с физикой.

**Architecture:** Pixi.js v7 WebGL canvas, 5 layered Containers, custom physics loop via Ticker, all assets embedded as base64.

**Tech Stack:** Pixi.js v7 (CDN), Web Audio API, vanilla JS, single `index.html`.

## Global Constraints

- Один файл `index.html`, все ассеты встроены base64
- Pixi.js v7: `https://cdn.jsdelivr.net/npm/pixi.js@7/dist/pixi.min.js`
- Canvas 800×600 внутри, масштаб CSS fit-to-screen
- Мобайл: touch события идентичны mouse
- Фон: frame_016.png (400×400, base64)
- Кот: 16 спрайтов frame_001..016.png (400×400, base64)

---

### Task 1: Скелет HTML + Pixi.Application + слои

**Files:**
- Create: `index.html`

- [ ] Создать `index.html` с базовой структурой:

```html
<!DOCTYPE html>
<html lang="ru">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
<title>Котик в офисе</title>
<style>
* { margin:0; padding:0; box-sizing:border-box; }
body { background:#111; display:flex; align-items:center; justify-content:center;
       min-height:100vh; overflow:hidden; touch-action:none; }
#game-wrap { position:relative; }
#game-wrap canvas { display:block; }
#ui-overlay { position:absolute; top:0; left:0; width:100%; height:100%;
              pointer-events:none; }
#hit-counter { position:absolute; top:12px; left:16px; color:#fff;
               font:bold 20px/1 Arial; text-shadow:0 2px 6px #000; pointer-events:none; }
#btn-reset { position:absolute; bottom:12px; right:12px; padding:8px 16px;
             background:rgba(0,0,0,0.6); color:#fff; border:2px solid rgba(255,255,255,0.4);
             border-radius:12px; font-size:14px; cursor:pointer; pointer-events:all; }
</style>
</head>
<body>
<div id="game-wrap">
  <div id="ui-overlay">
    <div id="hit-counter">💥 0</div>
    <button id="btn-reset">🔄 Привести в порядок</button>
  </div>
</div>
<script src="https://cdn.jsdelivr.net/npm/pixi.js@7/dist/pixi.min.js"></script>
<script>
// ── ASSETS (base64 вставляются в Task 2) ──────────────────────────────────
// const BG_FRAME = "data:image/png;base64,...";
// const CAT_FRAMES = { f01:"...", f02:"...", ... f16:"..." };

// ── CONSTANTS ──────────────────────────────────────────────────────────────
const W = 800, H = 600;
const GRAVITY = 980;
const BOUNCE  = 0.55;
const FRICTION = 0.82;

// ── PIXI SETUP ─────────────────────────────────────────────────────────────
const app = new PIXI.Application({
  width: W, height: H, backgroundColor: 0x1a1a2e, antialias: true
});
document.getElementById('game-wrap').prepend(app.view);

// Слои
const layerBg      = new PIXI.Container();
const layerObjects = new PIXI.Container();
const layerCat     = new PIXI.Container();
const layerFlying  = new PIXI.Container();
app.stage.addChild(layerBg, layerObjects, layerCat, layerFlying);

// Масштабирование под экран
function fitCanvas() {
  const scaleX = window.innerWidth  / W;
  const scaleY = window.innerHeight / H;
  const scale  = Math.min(scaleX, scaleY);
  app.view.style.width  = (W * scale) + 'px';
  app.view.style.height = (H * scale) + 'px';
  const wrap = document.getElementById('game-wrap');
  wrap.style.width  = (W * scale) + 'px';
  wrap.style.height = (H * scale) + 'px';
  // масштабируем ui-overlay тоже
  const overlay = document.getElementById('ui-overlay');
  overlay.style.transform = `scale(${scale})`;
  overlay.style.transformOrigin = '0 0';
  overlay.style.width  = W + 'px';
  overlay.style.height = H + 'px';
}
fitCanvas();
window.addEventListener('resize', fitCanvas);
</script>
</body>
</html>
```

- [ ] Открыть в браузере — должен быть тёмный прямоугольник 800×600, адаптированный под экран.
- [ ] Commit: `git add index.html && git commit -m "feat: pixi canvas skeleton with layers"`

---

### Task 2: Ассеты — embed background + все 16 спрайтов

**Files:**
- Modify: `index.html` (добавить base64 данные перед `// ── CONSTANTS`)

- [ ] Запустить Python-скрипт для генерации JS-блока с base64:

```python
import base64, os

lines = []

# Фон (frame_016)
with open('/tmp/sticker_frames/frame_016.png','rb') as f:
    b64 = base64.b64encode(f.read()).decode()
lines.append(f'const BG_FRAME = "data:image/png;base64,{b64}";')

# 16 спрайтов кота
lines.append('const CAT_FRAMES = {')
for i in range(1, 17):
    path = f'/tmp/sticker_frames/frame_{i:03d}.png'
    with open(path,'rb') as f:
        b64 = base64.b64encode(f.read()).decode()
    lines.append(f'  f{i:02d}: "data:image/png;base64,{b64}",')
lines.append('};')

with open('/tmp/assets_block.js','w') as f:
    f.write('\n'.join(lines))
print("Done:", sum(len(l) for l in lines)//1024, "KB")
```

- [ ] Вставить содержимое `/tmp/assets_block.js` в `index.html` перед строкой `// ── CONSTANTS`.
- [ ] Commit: `git commit -m "feat: embed sticker sprites as base64"`

---

### Task 3: Фон офиса

**Files:**
- Modify: `index.html` (после блока `// Слои`)

- [ ] Добавить рендер фона:

```js
// ── BACKGROUND ─────────────────────────────────────────────────────────────
function initBackground() {
  const tex = PIXI.Texture.from(BG_FRAME);
  const bg  = new PIXI.Sprite(tex);
  // Растянуть на весь canvas (оригинал 400×400 → 800×600)
  bg.width  = W;
  bg.height = H;
  layerBg.addChild(bg);
}
initBackground();
```

- [ ] Проверить в браузере — офис должен занимать весь canvas.
- [ ] Commit: `git commit -m "feat: office background from sticker frame"`

---

### Task 4: Система спрайтов кота

**Files:**
- Modify: `index.html`

- [ ] Добавить класс `CatSprite` после блока Background:

```js
// ── CAT SPRITE SYSTEM ──────────────────────────────────────────────────────
const CAT_STATES = {
  idle:       { frames: [16],                fps: 1   },
  walk:       { frames: [4,5,6,7,4,5,6,7],  fps: 8   },
  charge:     { frames: [1,2,3],             fps: 6   },
  beat:       { frames: [8,9,10,11,12],      fps: 12  },
  throw_anim: { frames: [10,11,12,13],       fps: 10  },
  aftermath:  { frames: [13,14,15],          fps: 6   },
  return_walk:{ frames: [4,5,6,7],           fps: 8   },
};

class CatSprite {
  constructor() {
    // Загружаем все текстуры
    this.textures = {};
    for (let i = 1; i <= 16; i++) {
      const key = `f${String(i).padStart(2,'0')}`;
      this.textures[i] = PIXI.Texture.from(CAT_FRAMES[key]);
    }

    // Стартовый кадр
    this.anim = new PIXI.AnimatedSprite([this.textures[16]]);
    this.anim.anchor.set(0.5, 1.0); // якорь — низ центр
    this.anim.animationSpeed = 0.15;
    this.anim.loop = true;
    this.anim.play();

    // Позиция покоя (за столом, совпадает с котом на фоне)
    this.homeX = 310;
    this.homeY = 420;
    this.anim.x = this.homeX;
    this.anim.y = this.homeY;
    this.anim.scale.set(0.55); // 400px → ~220px в сцене

    layerCat.addChild(this.anim);

    this.state    = 'idle';
    this.targetX  = this.homeX;
    this.targetY  = this.homeY;
    this.speed    = 180; // px/s
    this.bounceT  = 0;   // для чиби-bounce при ходьбе
    this.onArrival = null;
  }

  setState(name) {
    if (this.state === name) return;
    this.state = name;
    const cfg = CAT_STATES[name];
    if (!cfg) return;
    const texArr = cfg.frames.map(i => this.textures[i]);
    this.anim.textures = texArr;
    this.anim.animationSpeed = cfg.fps / 60;
    this.anim.loop = true;
    this.anim.gotoAndPlay(0);
    // Отражаем кота по X если идёт влево
    if (name === 'walk' || name === 'return_walk') {
      this.anim.scale.x = this.targetX < this.anim.x
        ? -Math.abs(this.anim.scale.x)
        :  Math.abs(this.anim.scale.x);
    }
  }

  walkTo(x, y, cb) {
    this.targetX   = x;
    this.targetY   = y;
    this.onArrival = cb;
    this.setState('walk');
  }

  goHome(cb) {
    this.walkTo(this.homeX, this.homeY, () => {
      this.setState('idle');
      if (cb) cb();
    });
  }

  update(dt) {
    if (this.state === 'walk' || this.state === 'return_walk') {
      const dx = this.targetX - this.anim.x;
      const dy = this.targetY - this.anim.y;
      const dist = Math.sqrt(dx*dx + dy*dy);

      if (dist < 4) {
        this.anim.x = this.targetX;
        this.anim.y = this.targetY;
        const cb = this.onArrival;
        this.onArrival = null;
        if (cb) cb();
      } else {
        const step = Math.min(this.speed * dt, dist);
        this.anim.x += (dx / dist) * step;
        // Чиби-bounce по Y
        this.bounceT += dt * 8;
        this.anim.y += (dy / dist) * step + Math.sin(this.bounceT) * 3;
      }
    }
  }
}

const cat = new CatSprite();
```

- [ ] Проверить в браузере — кот должен быть виден на фоне (поверх офиса).
- [ ] Commit: `git commit -m "feat: cat sprite system with state machine"`

---

### Task 5: Офисные объекты — отрисовка и расположение

**Files:**
- Modify: `index.html`

- [ ] Добавить определения объектов и их отрисовку:

```js
// ── OFFICE OBJECTS ─────────────────────────────────────────────────────────
// Позиции подобраны под перспективу офиса из стикера (800×600 canvas)
const OBJECT_DEFS = [
  { id:'chair',   label:'Офисное кресло', x:520, y:430, mass:5.0, maxCharge:2.0,
    w:70, h:80, color:0x222233, draw:'chair' },
  { id:'monitor', label:'Монитор',        x:350, y:310, mass:3.0, maxCharge:1.5,
    w:60, h:55, color:0x1a1a2e, draw:'monitor' },
  { id:'keyboard',label:'Клавиатура',     x:310, y:370, mass:1.5, maxCharge:0.8,
    w:90, h:20, color:0xcccccc, draw:'keyboard' },
  { id:'mouse',   label:'Мышка',          x:420, y:368, mass:0.5, maxCharge:0.3,
    w:22, h:32, color:0x333333, draw:'mouse' },
  { id:'papers',  label:'Папки',          x:385, y:330, mass:1.0, maxCharge:0.5,
    w:50, h:40, color:0xf5f0e0, draw:'papers' },
  { id:'pc',      label:'Системный блок', x:255, y:445, mass:8.0, maxCharge:3.0,
    w:45, h:60, color:0x3a3a3a, draw:'pc' },
  { id:'cabinet', label:'Шкаф',           x:640, y:330, mass:12.0, maxCharge:4.0,
    w:65, h:110, color:0x8a8070, draw:'cabinet' },
  { id:'mug',     label:'Кружка',         x:455, y:348, mass:0.3, maxCharge:0.2,
    w:22, h:28, color:0xff6644, draw:'mug' },
  { id:'stapler', label:'Степлер',        x:340, y:350, mass:0.8, maxCharge:0.4,
    w:35, h:16, color:0xcc2222, draw:'stapler' },
  { id:'lamp',    label:'Лампа',          x:240, y:330, mass:2.0, maxCharge:1.0,
    w:20, h:60, color:0xddcc88, draw:'lamp' },
];

// Текущие состояния объектов
const objects = new Map(); // id → { def, sprite, gfx, x, y, homeX, homeY, resting }

function drawObjectGfx(g, def) {
  g.clear();
  const c  = def.color;
  const dc = darken(c, 0.6);
  const lc = lighten(c, 1.5);
  const w  = def.w, h = def.h;

  switch(def.draw) {
    case 'chair': {
      // Сиденье
      g.beginFill(c); g.drawRoundedRect(-w/2, -h*0.35, w, h*0.15, 4); g.endFill();
      g.beginFill(lc,0.4); g.drawRoundedRect(-w/2, -h*0.35, w, 4, 2); g.endFill();
      // Спинка
      g.beginFill(dc); g.drawRoundedRect(-w/2, -h, w*0.9, h*0.6, 4); g.endFill();
      g.beginFill(lc,0.3); g.drawRoundedRect(-w/2, -h, w*0.9, 5, 2); g.endFill();
      // Ножки и крестовина
      g.beginFill(dc);
      g.drawRect(-3, -h*0.2, 6, h*0.55);
      g.drawEllipse(0, h*0.3, w*0.45, 8);
      [-w*0.45, w*0.45].forEach(ox => {
        g.drawCircle(ox, h*0.3, 5);
        g.drawRect(ox-2, h*0.35, 4, 10);
      });
      g.endFill();
      break;
    }
    case 'monitor': {
      // Экран
      g.beginFill(dc); g.drawRoundedRect(-w/2, -h, w, h*0.75, 4); g.endFill();
      g.beginFill(0x224488); g.drawRect(-w/2+4, -h+4, w-8, h*0.75-8); g.endFill();
      g.beginFill(0x4488ff, 0.3); g.drawRect(-w/2+4, -h+4, w-8, 8); g.endFill();
      // Подставка
      g.beginFill(dc); g.drawRect(-4, -h*0.25, 8, h*0.25); g.endFill();
      g.beginFill(dc); g.drawRoundedRect(-w*0.3, -2, w*0.6, 6, 3); g.endFill();
      break;
    }
    case 'keyboard': {
      g.beginFill(c); g.drawRoundedRect(-w/2, -h, w, h, 3); g.endFill();
      g.beginFill(lc, 0.5); g.drawRoundedRect(-w/2, -h, w, 4, 2); g.endFill();
      // Клавиши
      for(let row=0; row<3; row++) {
        for(let col=0; col<10; col++) {
          g.beginFill(0xeeeeee);
          g.drawRoundedRect(-w/2+5+col*8.5, -h+6+row*5, 7, 4, 1);
          g.endFill();
        }
      }
      break;
    }
    case 'mouse': {
      g.beginFill(c); g.drawEllipse(0, -h/2, w/2, h/2); g.endFill();
      g.beginFill(lc, 0.4); g.drawEllipse(0, -h*0.7, w/2, h*0.15); g.endFill();
      g.lineStyle(1, 0x555555); g.moveTo(0,-h); g.lineTo(0,-2); g.lineStyle(0);
      break;
    }
    case 'papers': {
      // Стопка папок
      [0xf5dca0, 0xeeccaa, c].forEach((col,i) => {
        g.beginFill(col);
        g.drawRoundedRect(-w/2+i*2, -h+i*4, w-i*4, h*0.8, 2);
        g.endFill();
      });
      g.lineStyle(1, 0xaaaaaa, 0.5);
      for(let i=0; i<4; i++) g.moveTo(-w/2+6, -h+10+i*8), g.lineTo(w/2-6, -h+10+i*8);
      g.lineStyle(0);
      break;
    }
    case 'pc': {
      g.beginFill(c); g.drawRoundedRect(-w/2, -h, w, h, 4); g.endFill();
      g.beginFill(lc,0.2); g.drawRoundedRect(-w/2+4, -h+6, w-8, 10, 2); g.endFill();
      // Кнопка
      g.beginFill(0x66aaff); g.drawCircle(-w/2+12, -h+28, 5); g.endFill();
      g.beginFill(0x333333); g.drawRoundedRect(-w/2+4, -h+38, w-8, 4, 2); g.endFill();
      break;
    }
    case 'cabinet': {
      g.beginFill(c); g.drawRoundedRect(-w/2, -h, w, h, 3); g.endFill();
      g.beginFill(lc, 0.2); g.drawRoundedRect(-w/2, -h, w, 6, 2); g.endFill();
      // Ящики
      [0.25, 0.5, 0.75].forEach(t => {
        g.lineStyle(2, dc); g.drawRect(-w/2+4, -h+h*t-2, w-8, h*0.22); g.lineStyle(0);
        g.beginFill(lc,0.5); g.drawCircle(0, -h+h*t+h*0.1, 4); g.endFill();
      });
      break;
    }
    case 'mug': {
      g.beginFill(c); g.drawRoundedRect(-w/2, -h, w, h, 4); g.endFill();
      g.beginFill(0xffffff,0.15); g.drawRoundedRect(-w/2, -h, w, 5, 3); g.endFill();
      // Ручка
      g.lineStyle(4, darken(c,0.7));
      g.arc(w/2+2, -h/2, 8, -Math.PI*0.6, Math.PI*0.6); g.lineStyle(0);
      // Пар
      g.lineStyle(2, 0xffffff, 0.4);
      g.moveTo(-4,-h-4); g.bezierCurveTo(-8,-h-12,-2,-h-16,-4,-h-22);
      g.moveTo(4,-h-4);  g.bezierCurveTo(8,-h-12,2,-h-16,4,-h-22);
      g.lineStyle(0);
      break;
    }
    case 'stapler': {
      g.beginFill(c); g.drawRoundedRect(-w/2, -h, w, h*0.6, 3); g.endFill();
      g.beginFill(dc); g.drawRoundedRect(-w/2, -h*0.4, w, h*0.4, 3); g.endFill();
      g.beginFill(lc,0.3); g.drawRoundedRect(-w/2, -h, w, 4, 2); g.endFill();
      break;
    }
    case 'lamp': {
      // Основание
      g.beginFill(dc); g.drawEllipse(0, 0, w*0.8, 6); g.endFill();
      // Штанга
      g.beginFill(dc); g.drawRect(-3, -h*0.7, 6, h*0.7); g.endFill();
      // Плафон
      g.beginFill(c);
      g.moveTo(-w/2, -h); g.lineTo(w/2, -h); g.lineTo(w/3, -h*0.7);
      g.lineTo(-w/3, -h*0.7); g.closePath(); g.endFill();
      g.beginFill(0xffffcc,0.6);
      g.moveTo(-w/2+2, -h+2); g.lineTo(w/2-2, -h+2);
      g.lineTo(w/3-2, -h*0.7); g.lineTo(-w/3+2, -h*0.7); g.closePath(); g.endFill();
      break;
    }
  }
}

// Вспомогательные цветовые функции
function darken(hex, f) {
  const r = ((hex>>16)&0xff)*f, g = ((hex>>8)&0xff)*f, b=(hex&0xff)*f;
  return (Math.round(r)<<16)|(Math.round(g)<<8)|Math.round(b);
}
function lighten(hex, f) {
  const r = Math.min(255,((hex>>16)&0xff)*f);
  const g = Math.min(255,((hex>>8)&0xff)*f);
  const b = Math.min(255,(hex&0xff)*f);
  return (Math.round(r)<<16)|(Math.round(g)<<8)|Math.round(b);
}

function initObjects() {
  for (const def of OBJECT_DEFS) {
    const gfx = new PIXI.Graphics();
    drawObjectGfx(gfx, def);
    gfx.x = def.x; gfx.y = def.y;
    gfx.cursor = 'pointer';
    gfx.eventMode = 'static';
    layerObjects.addChild(gfx);
    objects.set(def.id, { def, gfx, x:def.x, y:def.y, homeX:def.x, homeY:def.y, resting:true });
  }
}
initObjects();
```

- [ ] Проверить в браузере — все 10 объектов видны на фоне офиса.
- [ ] Commit: `git commit -m "feat: 10 office objects drawn with Pixi.Graphics"`

---

### Task 6: Система взаимодействия — клик + заряд + выброс

**Files:**
- Modify: `index.html`

- [ ] Добавить GameController — главную логику взаимодействия:

```js
// ── GAME CONTROLLER ────────────────────────────────────────────────────────
let hitCount    = 0;
let activeObj   = null;  // { obj, chargeStart, chargeT, glowGfx }
let catBusy     = false; // кот занят (идёт/бьёт)

const counterEl = document.getElementById('hit-counter');
const resetBtn  = document.getElementById('btn-reset');

// Glow ring (рисуется поверх объекта при заряде)
const glowGfx = new PIXI.Graphics();
layerFlying.addChild(glowGfx); // поверх всего

function onObjectPress(objId) {
  const obj = objects.get(objId);
  if (!obj || !obj.resting) return;

  // Прерываем текущий удар (если был)
  if (activeObj) cancelCharge();

  // Если кот уже идёт к другому — просто меняем цель
  catBusy = true;
  cat.walkTo(obj.x, obj.y - 30, () => {
    startCharge(obj);
  });
}

function startCharge(obj) {
  activeObj = { obj, chargeStart: performance.now(), chargeT: 0 };
  cat.setState('charge');
}

function cancelCharge() {
  activeObj = null;
  glowGfx.clear();
}

function releaseCharge() {
  if (!activeObj) return;
  const { obj, chargeT } = activeObj;
  const ratio  = Math.min(chargeT / obj.def.maxCharge, 1.0);

  cat.setState('throw_anim');
  playSound('throw');

  // Вычисляем скорость броска
  const maxForce = 800 / obj.def.mass;
  const vx = (Math.random() > 0.5 ? 1 : -1) * maxForce * (0.5 + ratio * 0.5);
  const vy = -maxForce * (0.4 + ratio * 0.6);

  launchObject(obj, vx, vy);
  cancelCharge();

  hitCount++;
  counterEl.textContent = '💥 ' + hitCount;

  // Кот возвращается домой через ~0.8s
  setTimeout(() => {
    cat.goHome(() => { catBusy = false; });
  }, 800);
}

// Навешиваем события на объекты
for (const [id, obj] of objects) {
  const gfx = obj.gfx;
  gfx.on('pointerdown', (e) => {
    e.stopPropagation();
    onObjectPress(id);
    // touch: начинаем заряд сразу как кот дошёл (см startCharge)
  });
  gfx.on('pointerup',       () => { if(activeObj?.obj === obj) releaseCharge(); });
  gfx.on('pointerupoutside',() => { if(activeObj?.obj === obj) releaseCharge(); });
}

// Кнопка Reset
resetBtn.addEventListener('click', resetOffice);
resetBtn.addEventListener('touchend', (e) => { e.preventDefault(); resetOffice(); });
```

- [ ] Commit: `git commit -m "feat: interaction system - click, charge, release"`

---

### Task 7: Физика полёта

**Files:**
- Modify: `index.html`

- [ ] Добавить систему летящих объектов:

```js
// ── PHYSICS ────────────────────────────────────────────────────────────────
const flyingObjects = []; // { gfx, def, x,y,vx,vy,rot,vrot, landed }

const FLOOR = H - 30;     // нижняя граница
const WALL_L = 20;
const WALL_R = W - 20;

function launchObject(obj, vx, vy) {
  obj.resting = false;
  obj.gfx.visible = false; // скрываем оригинал

  // Клонируем Graphics для физики
  const flyGfx = new PIXI.Graphics();
  drawObjectGfx(flyGfx, obj.def);
  flyGfx.x = obj.x;
  flyGfx.y = obj.y;
  layerFlying.addChild(flyGfx);

  flyingObjects.push({
    gfx: flyGfx,
    obj,
    x: obj.x, y: obj.y,
    vx, vy,
    rot: 0,
    vrot: vx * 0.003,
    bounces: 0,
    landed: false,
  });
}

function updatePhysics(dt) {
  for (let i = flyingObjects.length - 1; i >= 0; i--) {
    const f = flyingObjects[i];
    if (f.landed) continue;

    f.vy  += GRAVITY * dt;
    f.x   += f.vx * dt;
    f.y   += f.vy * dt;
    f.rot += f.vrot;

    // Отскок от пола
    if (f.y >= FLOOR) {
      f.y  = FLOOR;
      f.vy *= -BOUNCE;
      f.vx *= FRICTION;
      f.vrot *= 0.6;
      f.bounces++;
      if (f.bounces === 1) playSound(f.obj.def.mass > 4 ? 'land_heavy' : 'land_light');

      if (Math.abs(f.vy) < 30) {
        f.vy = 0; f.vx = 0; f.vrot = 0; f.landed = true;
        // Объект лежит на новом месте
        f.obj.x = f.x;
        f.obj.y = f.y;
        f.gfx.x = f.x;
        f.gfx.y = f.y;
        f.gfx.rotation = f.rot;
        // Переносим gfx обратно в layerObjects и делаем кликабельным
        layerFlying.removeChild(f.gfx);
        f.gfx.visible = true;
        // Уже привязан к layerObjects через obj.gfx? нет, это клон
        // Заменяем gfx объекта клоном
        layerObjects.addChild(f.gfx);
        f.obj.gfx = f.gfx;
        f.obj.resting = true;
        flyingObjects.splice(i, 1);
      }
    }

    // Отскок от стен
    if (f.x < WALL_L) { f.x = WALL_L; f.vx *= -BOUNCE; }
    if (f.x > WALL_R) { f.x = WALL_R; f.vx *= -BOUNCE; }

    f.gfx.x = f.x;
    f.gfx.y = f.y;
    f.gfx.rotation = f.rot;
  }
}
```

- [ ] Commit: `git commit -m "feat: physics - gravity, bounce, wall collision"`

---

### Task 8: Индикатор заряда (glow ring)

**Files:**
- Modify: `index.html`

- [ ] Добавить в game loop обновление glow-кольца:

```js
function updateCharge(dt) {
  if (!activeObj) { glowGfx.clear(); return; }

  activeObj.chargeT = (performance.now() - activeObj.chargeStart) / 1000;
  const ratio = Math.min(activeObj.chargeT / activeObj.obj.def.maxCharge, 1.0);

  // Цвет: зелёный → жёлтый → красный
  const r = Math.round(ratio * 255);
  const g = Math.round((1 - ratio) * 255);
  const color = (r << 16) | (g << 8);

  const obj = activeObj.obj;
  const radius = Math.max(obj.def.w, obj.def.h) * 0.7;

  glowGfx.clear();

  // Внешнее свечение (несколько колец)
  for (let ring = 3; ring > 0; ring--) {
    const alpha = (ratio * 0.3) / ring;
    glowGfx.lineStyle(ring * 3, color, alpha);
    glowGfx.drawCircle(obj.x, obj.y - obj.def.h/2, radius + ring * 5);
  }

  // Основное кольцо прогресса (дуга)
  glowGfx.lineStyle(4, color, 0.9);
  glowGfx.arc(obj.x, obj.y - obj.def.h/2, radius, -Math.PI/2, -Math.PI/2 + ratio * Math.PI * 2);

  // Вибрация при максимуме
  if (ratio >= 1.0) {
    const shake = Math.sin(performance.now() * 0.05) * 3;
    obj.gfx.x = obj.x + shake;
  } else {
    obj.gfx.x = obj.x;
  }

  // Кот бьёт быстрее при большем заряде
  cat.anim.animationSpeed = 0.1 + ratio * 0.25;

  playSound('hit'); // тихий удар при каждом кадре заряда
}
```

- [ ] Commit: `git commit -m "feat: charge glow ring with color gradient"`

---

### Task 9: Звук — Web Audio API синтез

**Files:**
- Modify: `index.html`

- [ ] Добавить звуковую систему:

```js
// ── SOUND ──────────────────────────────────────────────────────────────────
let audioCtx = null;
let lastHitSound = 0;

function getAudioCtx() {
  if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)();
  return audioCtx;
}

function playSound(type) {
  const ctx = getAudioCtx();
  const now = ctx.currentTime;

  switch(type) {
    case 'hit': {
      // Тихий удар, не чаще раза в 0.12с
      if (now - lastHitSound < 0.12) return;
      lastHitSound = now;
      const b = ctx.createOscillator();
      const g = ctx.createGain();
      b.connect(g); g.connect(ctx.destination);
      b.type = 'sawtooth'; b.frequency.value = 80;
      b.frequency.exponentialRampToValueAtTime(40, now + 0.08);
      g.gain.setValueAtTime(0.15, now);
      g.gain.exponentialRampToValueAtTime(0.001, now + 0.09);
      b.start(now); b.stop(now + 0.1);
      break;
    }
    case 'throw': {
      const b = ctx.createOscillator();
      const g = ctx.createGain();
      b.connect(g); g.connect(ctx.destination);
      b.type = 'sine'; b.frequency.setValueAtTime(200, now);
      b.frequency.exponentialRampToValueAtTime(800, now + 0.15);
      b.frequency.exponentialRampToValueAtTime(200, now + 0.3);
      g.gain.setValueAtTime(0.25, now);
      g.gain.exponentialRampToValueAtTime(0.001, now + 0.35);
      b.start(now); b.stop(now + 0.4);
      break;
    }
    case 'land_heavy': {
      const b  = ctx.createOscillator();
      const g  = ctx.createGain();
      const bf = ctx.createBiquadFilter();
      b.connect(bf); bf.connect(g); g.connect(ctx.destination);
      b.type = 'sawtooth'; b.frequency.setValueAtTime(120, now);
      b.frequency.exponentialRampToValueAtTime(20, now + 0.3);
      bf.type = 'lowpass'; bf.frequency.value = 300;
      g.gain.setValueAtTime(0.5, now);
      g.gain.exponentialRampToValueAtTime(0.001, now + 0.35);
      b.start(now); b.stop(now + 0.4);
      // Noise burst
      const bufLen = ctx.sampleRate * 0.1;
      const buf = ctx.createBuffer(1, bufLen, ctx.sampleRate);
      const data = buf.getChannelData(0);
      for(let i=0;i<bufLen;i++) data[i]=(Math.random()*2-1)*0.3;
      const ns = ctx.createBufferSource();
      const ng = ctx.createGain();
      ns.buffer = buf; ns.connect(ng); ng.connect(ctx.destination);
      ng.gain.setValueAtTime(0.4, now);
      ng.gain.exponentialRampToValueAtTime(0.001, now+0.12);
      ns.start(now);
      break;
    }
    case 'land_light': {
      const b = ctx.createOscillator();
      const g = ctx.createGain();
      b.connect(g); g.connect(ctx.destination);
      b.type = 'triangle'; b.frequency.setValueAtTime(600, now);
      b.frequency.exponentialRampToValueAtTime(200, now + 0.1);
      g.gain.setValueAtTime(0.2, now);
      g.gain.exponentialRampToValueAtTime(0.001, now + 0.12);
      b.start(now); b.stop(now + 0.15);
      break;
    }
    case 'reset': {
      const b = ctx.createOscillator();
      const g = ctx.createGain();
      b.connect(g); g.connect(ctx.destination);
      b.type = 'sine';
      b.frequency.setValueAtTime(400, now);
      b.frequency.linearRampToValueAtTime(800, now + 0.2);
      b.frequency.linearRampToValueAtTime(600, now + 0.4);
      g.gain.setValueAtTime(0.2, now);
      g.gain.exponentialRampToValueAtTime(0.001, now + 0.45);
      b.start(now); b.stop(now + 0.5);
      break;
    }
  }
}
```

- [ ] Commit: `git commit -m "feat: Web Audio sound synthesis"`

---

### Task 10: Reset + Game Loop

**Files:**
- Modify: `index.html`

- [ ] Добавить Reset и главный Ticker:

```js
// ── RESET ──────────────────────────────────────────────────────────────────
function resetOffice() {
  if (catBusy) return;
  playSound('reset');
  cancelCharge();

  // Убираем все летящие объекты
  for (const f of flyingObjects) layerFlying.removeChild(f.gfx);
  flyingObjects.length = 0;

  // Анимируем возврат всех объектов
  for (const [id, obj] of objects) {
    obj.resting = false;
    const startX = obj.x, startY = obj.y;
    const endX = obj.homeX, endY = obj.homeY;

    // Убираем старый gfx из layerObjects и layerFlying
    if (obj.gfx.parent) obj.gfx.parent.removeChild(obj.gfx);

    // Пересоздаём gfx на исходной позиции
    const gfx = new PIXI.Graphics();
    drawObjectGfx(gfx, obj.def);
    gfx.x = startX; gfx.y = startY;
    gfx.cursor = 'pointer';
    gfx.eventMode = 'static';
    gfx.on('pointerdown', (e) => { e.stopPropagation(); onObjectPress(id); });
    gfx.on('pointerup',        () => { if(activeObj?.obj === obj) releaseCharge(); });
    gfx.on('pointerupoutside', () => { if(activeObj?.obj === obj) releaseCharge(); });
    layerObjects.addChild(gfx);
    obj.gfx = gfx;

    // Плавная анимация возврата
    const dur = 0.5 + Math.random() * 0.3;
    let t = 0;
    const anim = (dt) => {
      t += dt / 60 / dur;
      const ease = 1 - Math.pow(1 - Math.min(t,1), 3);
      gfx.x = startX + (endX - startX) * ease;
      gfx.y = startY + (endY - startY) * ease;
      gfx.rotation = gfx.rotation * (1 - ease * 0.1);
      if (t >= 1) {
        gfx.x = endX; gfx.y = endY; gfx.rotation = 0;
        obj.x = endX; obj.y = endY;
        obj.resting = true;
        app.ticker.remove(anim);
      }
    };
    app.ticker.add(anim);
  }
}

// ── MAIN GAME LOOP ──────────────────────────────────────────────────────────
app.ticker.add((delta) => {
  const dt = delta / 60; // секунды
  cat.update(dt);
  updatePhysics(dt);
  updateCharge(dt);
});
```

- [ ] Commit: `git commit -m "feat: reset animation + main game loop"`

---

### Task 11: Сборка финального HTML + деплой

**Files:**
- Modify: `index.html` (финальная сборка)

- [ ] Запустить сборочный скрипт:

```bash
python3 << 'EOF'
import base64, os, re

# Читаем template
with open('/tmp/cat-chair-game/index.html') as f:
    html = f.read()

# Подставляем assets
lines = []
with open('/tmp/sticker_frames/frame_016.png','rb') as f:
    bg = base64.b64encode(f.read()).decode()
lines.append(f'const BG_FRAME = "data:image/png;base64,{bg}";')

lines.append('const CAT_FRAMES = {')
for i in range(1, 17):
    path = f'/tmp/sticker_frames/frame_{i:03d}.png'
    with open(path,'rb') as f:
        b64 = base64.b64encode(f.read()).decode()
    lines.append(f'  f{i:02d}: "data:image/png;base64,{b64}",')
lines.append('};')

assets_block = '\n'.join(lines)
html = html.replace('// ── ASSETS (base64 вставляются в Task 2) ──────────────────────────────────\n// const BG_FRAME = "data:image/png;base64,...";\n// const CAT_FRAMES = { f01:"...", f02:"...", ... f16:"..." };', assets_block)

with open('/tmp/cat-chair-game/index.html','w') as f:
    f.write(html)
print(f"Built: {len(html)//1024}KB")
EOF
```

- [ ] `git add index.html && git commit -m "feat: final build with embedded assets"`
- [ ] `git push origin main`
- [ ] Проверить https://stremyashiysya.github.io/cat-chair-game/

---

## Self-Review

**Spec coverage:**
- ✅ Офис из стикера как фон
- ✅ 16 спрайтов кота, все состояния
- ✅ 10 интерактивных объектов с массами
- ✅ Drag = hold to charge, release = fly
- ✅ Физика: гравитация 980, bounce 0.55, friction 0.82
- ✅ Glow ring: зелёный→жёлтый→красный
- ✅ Звуки: 5 типов через Web Audio
- ✅ Reset с анимацией возврата
- ✅ Мобайл: pointer events (покрывает и touch и mouse)

**Gaps:** Нет — все требования спека покрыты задачами.
