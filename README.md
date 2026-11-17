const STORAGE_KEY = 'angry-neighbor-save-v1';

const state = {
  started: false,
  menuOpen: true,
  difficulty: 'normal',
  music: true,
  sfx: true,
  level: 1,
  maxLevel: 10,
  totalTasks: 30,
  completedTasks: 0,
  mood: 0,
  timeOfDay: 0.18,
  inventory: [],
  hint: 'Tip: stay out of windows and keep moving when the neighbor looks around.',
  objective: 'Explore the street',
  objectiveBanner: 'Find the first key and sneak into the side gate.',
  gameOver: false,
  victory: false,
  player: {
    x: -2,
    y: 1.6,
    z: -18,
    yaw: 0,
    pitch: 0,
    speed: 0,
    sprinting: false,
    health: 100,
  },
  camera: {
    x: 0,
    y: 2.2,
    z: 10,
    yaw: 0,
    pitch: -0.4,
  },
  neighbor: {
    x: 12,
    y: 1.5,
    z: -8,
    yaw: 0,
    patrolIndex: 0,
    route: [
      { x: 14, z: -12 },
      { x: 26, z: -12 },
      { x: 26, z: 8 },
      { x: 10, z: 18 },
      { x: 6, z: -4 },
    ],
    alert: 0,
    seenPlayer: false,
    lastSeen: null,
    suspicion: 0,
    rageLevel: 0,
    state: 'patrol',
    speed: 1.1,
  },
  tasks: [],
  items: [],
  world: {
    width: 100,
    height: 100,
  },
  flags: {
    doorOpen: false,
    gateOpen: false,
    noteRead: false,
    garageFound: false,
    basementFound: false,
    secretRoom: false,
    redHerring: false,
    puzzleSolved: false,
    battleWon: false,
  },
};

const ui = {
  menu: document.getElementById('menuScreen'),
  hud: document.getElementById('hud'),
  toast: document.getElementById('toast'),
  taskCounter: document.getElementById('taskCounter'),
  objectiveText: document.getElementById('objectiveText'),
  objectiveBanner: document.getElementById('objectiveBanner'),
  hintText: document.getElementById('hintText'),
  moodMeter: document.getElementById('moodMeter'),
  moodText: document.getElementById('moodText'),
  timeText: document.getElementById('timeText'),
  inventoryGrid: document.getElementById('inventoryGrid'),
  inventoryCount: document.getElementById('inventoryCount'),
  difficultyText: document.getElementById('difficultyText'),
  musicToggle: document.getElementById('musicToggle'),
  sfxToggle: document.getElementById('sfxToggle'),
  gameOverScreen: document.getElementById('gameOverScreen'),
  victoryScreen: document.getElementById('victoryScreen'),
  playBtn: document.getElementById('playBtn'),
  continueBtn: document.getElementById('continueBtn'),
  resetBtn: document.getElementById('resetBtn'),
  restartBtn: document.getElementById('restartBtn'),
  nextLevelBtn: document.getElementById('nextLevelBtn'),
  menuBtn: document.getElementById('menuBtn'),
  menuBtn2: document.getElementById('menuBtn2'),
};

const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');

const keys = {};
let lastTime = 0;
let pointerLocked = false;
let pointerX = 0;
let pointerY = 0;

const difficultyMap = {
  easy: { neighborSpeed: 0.82, moodGain: 0.8 },
  normal: { neighborSpeed: 1.0, moodGain: 1.0 },
  hard: { neighborSpeed: 1.24, moodGain: 1.25 },
};

const iconMap = {
  key: '🗝️',
  note: '📝',
  tool: '🔧',
  item: '📦',
  toy: '🧸',
  puzzle: '🧩',
  trait: '👣',
  door: '🚪',
  torch: '🔥',
  candy: '🍬',
  camera: '📷',
  clock: '🕒',
  battery: '🔋',
  ladder: '🪜',
  mask: '🎭',
  cookie: '🍪',
};

function cloneTemplateItems() {
  return [
    { id: 'front_key', name: 'Front Door Key', icon: iconMap.key, x: 8, z: -20, found: false },
    { id: 'garden_note', name: 'Garden Note', icon: iconMap.note, x: -16, z: 6, found: false },
    { id: 'crowbar', name: 'Crowbar', icon: iconMap.tool, x: 30, z: 10, found: false },
    { id: 'ladder', name: 'Ladder', icon: iconMap.ladder, x: -34, z: -15, found: false },
    { id: 'battery', name: 'Battery', icon: iconMap.battery, x: 20, z: -18, found: false },
    { id: 'lantern', name: 'Lantern', icon: iconMap.torch, x: -28, z: 18, found: false },
    { id: 'cookie', name: 'Cookie', icon: iconMap.cookie, x: 26, z: 24, found: false },
    { id: 'mask', name: 'Masked Toy', icon: iconMap.mask, x: -10, z: 26, found: false },
    { id: 'candy', name: 'Candy', icon: iconMap.candy, x: 34, z: -24, found: false },
    { id: 'puzzle', name: 'Puzzle Piece', icon: iconMap.puzzle, x: -26, z: -22, found: false },
    { id: 'camera', name: 'Camera', icon: iconMap.camera, x: 14, z: 28, found: false },
    { id: 'clock', name: 'Broken Clock', icon: iconMap.clock, x: 0, z: 20, found: false },
  ];
}

function createTaskList() {
  return [
    { id: 'front_key', title: 'Find the front door key', check: () => state.inventory.some(i => i.id === 'front_key') },
    { id: 'garden_note', title: 'Read the suspicious note', check: () => state.flags.noteRead },
    { id: 'open_gate', title: 'Open the garden gate', check: () => state.flags.gateOpen },
    { id: 'garage', title: 'Inspect the garage', check: () => state.flags.garageFound },
    { id: 'basement', title: 'Find the basement stairs', check: () => state.flags.basementFound },
    { id: 'secret_room', title: 'Unlock the hidden room', check: () => state.flags.secretRoom },
    { id: 'puzzle', title: 'Solve the puzzle', check: () => state.flags.puzzleSolved },
    { id: 'crowbar', title: 'Pick up the crowbar', check: () => state.inventory.some(i => i.id === 'crowbar') },
    { id: 'ladder', title: 'Grab the ladder', check: () => state.inventory.some(i => i.id === 'ladder') },
    { id: 'battery', title: 'Collect the battery', check: () => state.inventory.some(i => i.id === 'battery') },
    { id: 'lantern', title: 'Take the lantern', check: () => state.inventory.some(i => i.id === 'lantern') },
    { id: 'mask', title: 'Find the prank mask', check: () => state.inventory.some(i => i.id === 'mask') },
    { id: 'cookie', title: 'Steal a cookie', check: () => state.inventory.some(i => i.id === 'cookie') },
    { id: 'candy', title: 'Grab the candy', check: () => state.inventory.some(i => i.id === 'candy') },
    { id: 'camera', title: 'Take the hidden camera', check: () => state.inventory.some(i => i.id === 'camera') },
    { id: 'clock', title: 'Inspect the broken clock', check: () => state.flags.redHerring },
    { id: 'quiet_run', title: 'Sneak past the neighbor', check: () => state.neighbor.suspicion < 55 },
    { id: 'hide_bush', title: 'Hide in the bush', check: () => state.flags.redHerring },
    { id: 'door_noise', title: 'Make a distraction', check: () => state.flags.doorOpen },
    { id: 'garage_lure', title: 'Set a garage trap', check: () => state.flags.garageFound },
    { id: 'music', title: 'Turn up the music prank', check: () => state.flags.redHerring },
    { id: 'broom', title: 'Use the broom as decoy', check: () => state.flags.redHerring },
    { id: 'sneak_behind', title: 'Slip behind the fence', check: () => state.player.x < -14 && state.player.z < 18 },
    { id: 'search_house', title: 'Search the house hall', check: () => state.player.x > 8 && state.player.z < 5 },
    { id: 'parking', title: 'Check the parking lot', check: () => state.player.x > 20 && state.player.z > -20 },
    { id: 'park', title: 'Visit the park', check: () => state.player.x < -25 && state.player.z > 15 },
    { id: 'lantern_path', title: 'Use the lantern for light', check: () => state.inventory.some(i => i.id === 'lantern') },
    { id: 'bypass', title: 'Find a bypass route', check: () => state.player.x > -12 && state.player.z > 22 },
    { id: 'final_stash', title: 'Recover the final stash', check: () => state.flags.battleWon },
    { id: 'level_clear', title: 'Complete the level', check: () => state.victory || state.completedTasks >= state.totalTasks },
    { id: 'room_clear', title: 'Clear the hidden room', check: () => state.flags.secretRoom },
  ];
}

function clamp(value, min, max) {
  return Math.min(Math.max(value, min), max);
}

function clampAngle(angle) {
  while (angle < -Math.PI) angle += Math.PI * 2;
  while (angle > Math.PI) angle -= Math.PI * 2;
  return angle;
}

function setToast(msg) {
  ui.toast.textContent = msg;
  ui.toast.classList.add('visible');
  clearTimeout(setToast.timeoutId);
  setToast.timeoutId = setTimeout(() => ui.toast.classList.remove('visible'), 1400);
}

function updateDifficultyUI() {
  const buttons = document.querySelectorAll('.difficulty-btn');
  buttons.forEach((button) => {
    const isActive = button.dataset.difficulty === state.difficulty;
    button.classList.toggle('active', isActive);
  });
  ui.difficultyText.textContent = state.difficulty[0].toUpperCase() + state.difficulty.slice(1);
}

function updateToggleButtons() {
  ui.musicToggle.classList.toggle('active', state.music);
  ui.musicToggle.textContent = state.music ? 'On' : 'Off';
  ui.sfxToggle.classList.toggle('active', state.sfx);
  ui.sfxToggle.textContent = state.sfx ? 'On' : 'Off';
}

function showMenu() {
  state.menuOpen = true;
  ui.menu.classList.remove('hidden');
  ui.hud.classList.add('hidden');
  ui.gameOverScreen.classList.add('hidden');
  ui.victoryScreen.classList.add('hidden');
}

function hideMenu() {
  state.menuOpen = false;
  ui.menu.classList.add('hidden');
  ui.hud.classList.remove('hidden');
  ui.gameOverScreen.classList.add('hidden');
  ui.victoryScreen.classList.add('hidden');
}

function loadSave() {
  const raw = localStorage.getItem(STORAGE_KEY);
  if (!raw) return null;
  try {
    return JSON.parse(raw);
  } catch {
    return null;
  }
}

function saveGame() {
  const payload = {
    level: state.level,
    difficulty: state.difficulty,
    music: state.music,
    sfx: state.sfx,
    mood: state.mood,
    completedTasks: state.completedTasks,
    totalTasks: state.totalTasks,
    objective: state.objective,
    objectiveBanner: state.objectiveBanner,
    hint: state.hint,
    inventory: state.inventory,
    items: state.items,
    player: state.player,
    neighbor: state.neighbor,
    flags: state.flags,
    started: state.started,
  };
  localStorage.setItem(STORAGE_KEY, JSON.stringify(payload));
}

function resetSave() {
  localStorage.removeItem(STORAGE_KEY);
  setToast('Save reset');
  initGame();
  showMenu();
}

function refreshInventory() {
  const slots = Array.from({ length: 9 }, (_, i) => {
    const item = state.inventory[i] || null;
    const node = document.createElement('div');
    node.className = 'slot ' + (item ? '' : 'empty');
    if (item) {
      node.innerHTML = `${item.icon}<span class="count">${item.amount || 1}</span>`;
    }
    return node;
  });

  ui.inventoryGrid.innerHTML = '';
  slots.forEach(slot => ui.inventoryGrid.appendChild(slot));
  ui.inventoryCount.textContent = String(state.inventory.length);
}

function addInventoryItem(item) {
  const existing = state.inventory.find(entry => entry.id === item.id);
  if (existing) {
    existing.amount = (existing.amount || 1) + 1;
  } else {
    state.inventory.push({ ...item, amount: 1 });
  }
  refreshInventory();
}

function setObjective(title, hint, banner) {
  state.objective = title;
  state.hint = hint;
  state.objectiveBanner = banner;
  ui.objectiveText.textContent = title;
  ui.hintText.textContent = hint;
  ui.objectiveBanner.textContent = banner;
}

function getCurrentTask() {
  const active = state.tasks.find(task => !task.check());
  return active || null;
}

function updateTaskProgress() {
  const total = state.tasks.length;
  state.completedTasks = state.tasks.filter(task => task.check()).length;
  state.totalTasks = total;
  ui.taskCounter.textContent = `${state.completedTasks} / ${state.totalTasks}`;

  const active = getCurrentTask();
  if (active) {
    setObjective(active.title, 'Keep moving and stay hidden.', 'Complete the current mission without getting seen.');
  }
}

function addTaskProgressIfNeeded(key) {
  const task = state.tasks.find(entry => entry.id === key);
  if (task) {
    const current = task.check();
    if (!current) {
      state.completedTasks = state.tasks.filter(item => item.check()).length;
      ui.taskCounter.textContent = `${state.completedTasks} / ${state.totalTasks}`;
    }
  }
}

function handlePickup(itemId) {
  const item = state.items.find(entry => entry.id === itemId && !entry.found);
  if (!item) return false;

  item.found = true;
  addInventoryItem({ id: item.id, name: item.name, icon: item.icon, amount: 1 });
  setToast(`${item.name} collected`);
  if (item.id === 'front_key') {
    state.flags.gateOpen = true;
    setObjective('Find the gate key', 'The first door is now open. Head to the back garden.', 'A useful key has been found.');
  }
  if (item.id === 'garden_note') {
    state.flags.noteRead = true;
    setObjective('Analyze the note', 'The note points to the hidden route behind the shed.', 'The note suggests a secret lane near the back garden.');
  }
  updateTaskProgress();
  saveGame();
  return true;
}

function interact() {
  const nearbyItems = state.items.filter(item => {
    const dx = state.player.x - item.x;
    const dz = state.player.z - item.z;
    return Math.hypot(dx, dz) < 2.4;
  });

  if (nearbyItems.length > 0) {
    const item = nearbyItems[0];
    const picked = handlePickup(item.id);
    if (picked) {
      return;
    }
  }

  const nearbyGate = Math.hypot(state.player.x - 12, state.player.z + 2) < 2.6;
  if (nearbyGate) {
    state.flags.gateOpen = true;
    setToast('Garden gate opened');
    state.objective = 'Search the house and make a distraction';
    setObjective('Search the house and make a distraction', 'The gate is open; the neighbor has room to move around the garden.', 'Sneak through the gate and keep to cover.');
    state.completedTasks = Math.min(state.completedTasks + 1, state.totalTasks);
    saveGame();
  }

  const nearbyGarage = Math.hypot(state.player.x - 30, state.player.z + 20) < 3.2;
  if (nearbyGarage) {
    state.flags.garageFound = true;
    setToast('Garage found');
    state.mood = clamp(state.mood + 8, 0, 100);
    setObjective('Inspect the garage', 'A sound trap is hidden inside the garage. Keep it quiet.', 'The garage smells of old tools and suspicious crates.');
    saveGame();
  }

  const nearbyBasement = Math.hypot(state.player.x - 6, state.player.z - 20) < 2.75;
  if (nearbyBasement) {
    state.flags.basementFound = true;
    setToast('Basement stairs found');
    setObjective('Open the secret room', 'A hidden passage is behind the basement stairs.', 'Follow the stairs before the neighbor reaches the hallway.');
    saveGame();
  }

  const nearbySecret = Math.hypot(state.player.x - 18, state.player.z - 4) < 2.9;
  if (nearbySecret) {
    state.flags.secretRoom = true;
    setToast('Secret room discovered');
    setObjective('Solve the final puzzle', 'The door is open, but a final riddle remains.', 'The secret room contains the last clue.');
    saveGame();
  }

  if (state.flags.secretRoom && state.inventory.some(i => i.id === 'puzzle')) {
    state.flags.puzzleSolved = true;
    setToast('Puzzle solved');
    state.flags.battleWon = true;
    setObjective('Final stash recovered', 'The final stash is in the hidden room.', 'You found the evidence and the case is almost over.');
    saveGame();
  }

  updateTaskProgress();
}

function updateNeighbor(dt) {
  const diffX = state.player.x - state.neighbor.x;
  const diffZ = state.player.z - state.neighbor.z;
  const dist = Math.hypot(diffX, diffZ);
  const speed = difficultyMap[state.difficulty].neighborSpeed;

  if (dist < 12) {
    state.neighbor.state = 'chase';
    state.neighbor.alert += dt * 1.5 * difficultyMap[state.difficulty].moodGain;
    state.neighbor.suspicion = clamp(state.neighbor.suspicion + dt * 14, 0, 100);
    state.neighbor.lastSeen = { x: state.player.x, z: state.player.z };
    state.neighbor.seenPlayer = true;
  } else {
    state.neighbor.state = 'patrol';
    state.neighbor.alert = clamp(state.neighbor.alert - dt * 0.7, 0, 100);
  }

  if (state.neighbor.state === 'patrol') {
    const target = state.neighbor.route[state.neighbor.patrolIndex];
    const dx = target.x - state.neighbor.x;
    const dz = target.z - state.neighbor.z;
    const len = Math.hypot(dx, dz) || 1;
    if (len < 1.4) {
      state.neighbor.patrolIndex = (state.neighbor.patrolIndex + 1) % state.neighbor.route.length;
    } else {
      state.neighbor.x += (dx / len) * dt * speed * 2.1;
      state.neighbor.z += (dz / len) * dt * speed * 2.1;
    }
  } else {
    const dx = state.player.x - state.neighbor.x;
    const dz = state.player.z - state.neighbor.z;
    const len = Math.hypot(dx, dz) || 1;
    state.neighbor.x += (dx / len) * dt * speed * 2.5;
    state.neighbor.z += (dz / len) * dt * speed * 2.5;
  }

  const yaw = Math.atan2(state.player.x - state.neighbor.x, state.player.z - state.neighbor.z);
  state.neighbor.yaw = yaw * 180 / Math.PI;

  const hearingBoost = (keys.Shift ? 18 : 0) + (state.objective.startsWith('Find the gate') ? 5 : 0);
  if (hearingBoost > 0 && dist < 15) {
    state.neighbor.suspicion = clamp(state.neighbor.suspicion + dt * 12, 0, 100);
  }

  state.mood = clamp(state.mood + (dist < 9 ? 1.4 : 0.2) * dt * difficultyMap[state.difficulty].moodGain * 18, 0, 100);

  if (dist < 2.4) {
    state.gameOver = true;
    showLoseScreen();
  }

  if (state.completedTasks >= state.totalTasks || state.flags.battleWon) {
    state.victory = true;
    showWinScreen();
  }

  updateMoodUI();
}

function updateMoodUI() {
  const mood = clamp(state.mood, 0, 100);
  ui.moodMeter.style.width = `${mood}%`;
  if (mood < 25) ui.moodText.textContent = 'Calm';
  else if (mood < 55) ui.moodText.textContent = 'Suspicious';
  else if (mood < 75) ui.moodText.textContent = 'Angry';
  else ui.moodText.textContent = 'Hostile';
}

function showLoseScreen() {
  ui.gameOverScreen.classList.remove('hidden');
  setToast('Caught!');
}

function showWinScreen() {
  ui.victoryScreen.classList.remove('hidden');
  setToast('Case solved!');
}

function initGame() {
  state.started = false;
  state.gameOver = false;
  state.victory = false;
  state.level = 1;
  state.timeOfDay = 0.18;
  state.player.x = -2;
  state.player.z = -18;
  state.player.yaw = 0;
  state.mood = 0;
  state.flags = {
    doorOpen: false,
    gateOpen: false,
    noteRead: false,
    garageFound: false,
    basementFound: false,
    secretRoom: false,
    redHerring: false,
    puzzleSolved: false,
    battleWon: false,
  };
  state.inventory = [];
  state.items = cloneTemplateItems();
  state.neighbor.x = 12;
  state.neighbor.z = -8;
  state.neighbor.patrolIndex = 0;
  state.neighbor.state = 'patrol';
  state.neighbor.suspicion = 0;
  state.neighbor.alert = 0;
  state.tasks = createTaskList();
  state.objective = 'Explore the street';
  state.objectiveBanner = 'Find the first key and sneak into the side gate.';
  state.hint = 'Tip: stay out of windows and keep moving when the neighbor looks around.';
  updateMoodUI();
  refreshInventory();
  updateTaskProgress();
  saveGame();
}

function startGame() {
  hideMenu();
  state.started = true;
  state.gameOver = false;
  state.victory = false;
  ui.gameOverScreen.classList.add('hidden');
  ui.victoryScreen.classList.add('hidden');
  if (!pointerLocked) {
    canvas.requestPointerLock?.();
  }
  setToast('Mission started');
  saveGame();
}

function continueGame() {
  const save = loadSave();
  if (!save) {
    initGame();
    startGame();
    return;
  }

  state.level = save.level || 1;
  state.difficulty = save.difficulty || 'normal';
  state.music = save.music ?? true;
  state.sfx = save.sfx ?? true;
  state.mood = save.mood || 0;
  state.completedTasks = save.completedTasks || 0;
  state.totalTasks = save.totalTasks || 30;
  state.objective = save.objective || 'Explore the street';
  state.objectiveBanner = save.objectiveBanner || 'Find the first key and sneak into the side gate.';
  state.hint = save.hint || 'Tip: stay out of windows and keep moving when the neighbor looks around.';
  state.inventory = save.inventory || [];
  state.items = save.items || cloneTemplateItems();
  state.player = save.player || state.player;
  state.neighbor = save.neighbor || state.neighbor;
  state.flags = save.flags || state.flags;
  state.started = true;
  state.menuOpen = false;
  hideMenu();
  updateMoodUI();
  refreshInventory();
  setToast('Continue loaded');
}

function handleKeyDown(event) {
  keys[event.code] = true;
  if (event.code === 'KeyE') interact();
  if (event.code === 'Escape') {
    showMenu();
  }
}

function handleKeyUp(event) {
  keys[event.code] = false;
}

window.addEventListener('keydown', handleKeyDown);
window.addEventListener('keyup', handleKeyUp);

canvas.addEventListener('click', () => {
  if (!pointerLocked) {
    canvas.requestPointerLock?.();
  }
});

document.addEventListener('pointerlockchange', () => {
  pointerLocked = document.pointerLockElement === canvas;
});

document.addEventListener('mousemove', (event) => {
  if (!pointerLocked || !state.started || state.gameOver || state.victory) return;
  state.player.yaw -= event.movementX * 0.0025;
  state.camera.yaw = state.player.yaw;
});

function updatePlayer(dt) {
  if (!state.started || state.gameOver || state.victory) return;

  let moveX = 0;
  let moveZ = 0;
  if (keys.KeyW || keys.ArrowUp) moveZ -= 1;
  if (keys.KeyS || keys.ArrowDown) moveZ += 1;
  if (keys.KeyA || keys.ArrowLeft) moveX -= 1;
  if (keys.KeyD || keys.ArrowRight) moveX += 1;

  const sprint = keys.Shift ? 1.55 : 1;
  if (moveX !== 0 || moveZ !== 0) {
    const len = Math.hypot(moveX, moveZ) || 1;
    const moveYaw = Math.atan2(moveX, moveZ);
    const worldYaw = state.player.yaw + moveYaw;
    const nx = Math.sin(worldYaw) * (moveX / len);
    const nz = Math.cos(worldYaw) * (moveZ / len);
    state.player.x += nx * dt * 8 * sprint;
    state.player.z += nz * dt * 8 * sprint;
  }

  state.player.x = clamp(state.player.x, -48, 48);
  state.player.z = clamp(state.player.z, -48, 48);

  state.camera.x = state.player.x + Math.sin(state.player.yaw) * 2.5;
  state.camera.z = state.player.z + Math.cos(state.player.yaw) * 2.5;
  state.camera.yaw = state.player.yaw;
}

function projectPoint(x, z, cameraX, cameraZ, cameraYaw) {
  const dx = x - cameraX;
  const dz = z - cameraZ;
  const cosA = Math.cos(-cameraYaw);
  const sinA = Math.sin(-cameraYaw);
  const rx = dx * cosA - dz * sinA;
  const rz = dx * sinA + dz * cosA;
  return { x: rx, z: rz };
}

function drawGround() {
  const centerX = canvas.width / 2;
  const centerY = canvas.height * 0.68;
  const gradient = ctx.createLinearGradient(0, 0, 0, canvas.height);
  gradient.addColorStop(0, '#8fd0ff');
  gradient.addColorStop(0.46, '#dfefff');
  gradient.addColorStop(0.7, '#afcf86');
  gradient.addColorStop(1, '#6d8e5d');
  ctx.fillStyle = gradient;
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  const skyGlow = state.timeOfDay > 0.45 ? '#112131' : '#ffe6a3';
  ctx.fillStyle = skyGlow;
  ctx.beginPath();
  ctx.arc(canvas.width - 120, 90, 48, 0, Math.PI * 2);
  ctx.fill();

  // road
  ctx.fillStyle = '#464a59';
  ctx.fillRect(centerX - 140, centerY - 40, 280, 220);

  // path segments
  ctx.fillStyle = '#8d735d';
  ctx.fillRect(centerX - 180, centerY + 120, 360, 60);

  // yard and house shapes, simplified pseudo-3D buildings
  const houseX = centerX + 140;
  const houseY = centerY - 80;
  const houseW = 200;
  const houseH = 180;

  ctx.fillStyle = '#b99a7d';
  ctx.fillRect(houseX, houseY, houseW, houseH);
  ctx.fillStyle = '#8d6d53';
  ctx.fillRect(houseX + 20, houseY + 20, houseW - 40, houseH - 40);

  ctx.fillStyle = '#d7e7f5';
  for (let i = 0; i < 5; i++) {
    ctx.fillRect(houseX + 28 + i * 30, houseY + 35, 18, 42);
  }

  // garage
  ctx.fillStyle = '#8b8f95';
  ctx.fillRect(centerX - 290, centerY + 10, 120, 100);

  // park
  ctx.fillStyle = '#7dbd67';
  ctx.fillRect(centerX + 200, centerY + 100, 110, 90);

  // bushes
  ctx.fillStyle = '#3a7b44';
  for (let i = 0; i < 7; i++) {
    ctx.beginPath();
    ctx.arc(centerX - 180 + i * 44, centerY + 160, 18, 0, Math.PI * 2);
    ctx.fill();
  }
}

function drawSprite(worldX, worldZ, label, color, size = 24, isPlayer = false) {
  const cameraX = state.camera.x;
  const cameraZ = state.camera.z;
  const yaw = state.camera.yaw;
  const p = projectPoint(worldX, worldZ, cameraX, cameraZ, yaw);
  const cx = canvas.width / 2 + p.x * 11.2;
  const cy = canvas.height * 0.7 - p.z * 8.5;
  const s = size * (1 / (Math.max(1, p.z) * 0.4 + 1));

  if (p.z < 0.2) return;

  ctx.fillStyle = color;
  ctx.fillRect(cx - s / 2, cy - s, s, s * 1.4);
  ctx.fillStyle = '#0b1015';
  ctx.fillRect(cx - s * 0.8, cy - s * 1.2, s * 1.6, 8);
  ctx.fillStyle = '#fff';
  ctx.font = '12px sans-serif';
  ctx.textAlign = 'center';
  ctx.fillText(label, cx, cy - s * 1.55);
}

function drawItem(item) {
  if (item.found) return;
  const p = projectPoint(item.x, item.z, state.camera.x, state.camera.z, state.camera.yaw);
  const cx = canvas.width / 2 + p.x * 11.2;
  const cy = canvas.height * 0.7 - p.z * 8.5;
  if (p.z < 0.2) return;

  ctx.font = '28px sans-serif';
  ctx.textAlign = 'center';
  ctx.fillText(item.icon, cx, cy);

  ctx.font = '12px sans-serif';
  ctx.fillStyle = 'rgba(0,0,0,0.6)';
  ctx.fillRect(cx - 42, cy + 18, 84, 18);
  ctx.fillStyle = '#ffffff';
  ctx.fillText(item.name, cx, cy + 31);
}

function renderScene() {
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  drawGround();

  state.items.forEach(drawItem);
  drawSprite(state.neighbor.x, state.neighbor.z, 'NEIGHBOR', '#ff8d63', 24);
  drawSprite(state.player.x, state.player.z, 'YOU', '#70d8ff', 20, true);

  // decorative trees and props
  const treePositions = [
    { x: 18, z: -20 }, { x: -22, z: 12 }, { x: 32, z: 18 }, { x: -30, z: -10 }, { x: -8, z: 24 },
  ];
  treePositions.forEach(tree => {
    const p = projectPoint(tree.x, tree.z, state.camera.x, state.camera.z, state.camera.yaw);
    const cx = canvas.width / 2 + p.x * 11.2;
    const cy = canvas.height * 0.7 - p.z * 8.5;
    if (p.z < 0.15) return;
    ctx.fillStyle = '#5f8a49';
    ctx.beginPath();
    ctx.arc(cx, cy - 10, 14, 0, Math.PI * 2);
    ctx.fill();
    ctx.fillStyle = '#7d5a3b';
    ctx.fillRect(cx - 4, cy + 6, 8, 18);
  });
}

function updateDayNight(dt) {
  state.timeOfDay += dt * 0.025;
  if (state.timeOfDay > 1) state.timeOfDay = 0;
  const isNight = state.timeOfDay > 0.55;
  ui.timeText.textContent = isNight ? 'Night' : 'Day';
}

function updateTaskUI() {
  ui.taskCounter.textContent = `${state.completedTasks} / ${state.totalTasks}`;
  ui.objectiveText.textContent = state.objective;
  ui.hintText.textContent = state.hint;
  ui.objectiveBanner.textContent = state.objectiveBanner;
}

function nextLevel() {
  state.level = Math.min(state.maxLevel, state.level + 1);
  initGame();
  startGame();
}

function tick(timestamp) {
  const dt = Math.min(0.033, (timestamp - lastTime) / 1000 || 0.016);
  lastTime = timestamp;

  if (state.started && !state.gameOver && !state.victory) {
    updateDayNight(dt);
    updatePlayer(dt);
    updateNeighbor(dt);
    updateTaskUI();
    updateMoodUI();
    saveGame();
  }

  renderScene();
  requestAnimationFrame(tick);
}

function bindUI() {
  document.querySelectorAll('.difficulty-btn').forEach((btn) => {
    btn.addEventListener('click', () => {
      state.difficulty = btn.dataset.difficulty;
      updateDifficultyUI();
      saveGame();
    });
  });

  ui.musicToggle.addEventListener('click', () => {
    state.music = !state.music;
    updateToggleButtons();
    saveGame();
  });

  ui.sfxToggle.addEventListener('click', () => {
    state.sfx = !state.sfx;
    updateToggleButtons();
    saveGame();
  });

  ui.playBtn.addEventListener('click', () => {
    initGame();
    startGame();
  });

  ui.continueBtn.addEventListener('click', () => {
    continueGame();
  });

  ui.resetBtn.addEventListener('click', () => {
    resetSave();
  });

  ui.restartBtn.addEventListener('click', () => {
    initGame();
    startGame();
  });

  ui.nextLevelBtn.addEventListener('click', () => {
    nextLevel();
  });

  ui.menuBtn.addEventListener('click', showMenu);
  ui.menuBtn2.addEventListener('click', showMenu);
}

function resizeCanvas() {
  const ratio = window.devicePixelRatio || 1;
  canvas.width = window.innerWidth * ratio;
  canvas.height = window.innerHeight * ratio;
  canvas.style.width = `${window.innerWidth}px`;
  canvas.style.height = `${window.innerHeight}px`;
  ctx.setTransform(ratio, 0, 0, ratio, 0, 0);
}

window.addEventListener('resize', resizeCanvas);
resizeCanvas();

bindUI();
updateDifficultyUI();
updateToggleButtons();
initGame();
showMenu();
updateTaskUI();
requestAnimationFrame(tick);

// Simple sound generation for footsteps and tension
function playTone(freq, duration, type = 'sine', volume = 0.03) {
  if (!state.sfx) return;
  const AudioCtx = window.AudioContext || window.webkitAudioContext;
  if (!AudioCtx) return;
  const ctxAudio = new AudioCtx();
  const osc = ctxAudio.createOscillator();
  const gain = ctxAudio.createGain();
  osc.type = type;
  osc.frequency.value = freq;
  gain.gain.value = volume;
  osc.connect(gain).connect(ctxAudio.destination);
  osc.start();
  osc.stop(ctxAudio.currentTime + duration);
  gain.gain.exponentialRampToValueAtTime(0.0001, ctxAudio.currentTime + duration);
}

window.addEventListener('keydown', (event) => {
  if (['KeyW','KeyA','KeyS','KeyD','ArrowUp','ArrowDown','ArrowLeft','ArrowRight','ShiftLeft','ShiftRight'].includes(event.code)) {
    if (state.sfx) playTone(80, 0.08, 'triangle', 0.015);
  }
});
