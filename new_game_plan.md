# Bloons Tower Defense Clone - Godot Architecture Plan

## Project Overview
A learning project to build a Bloons TD-style tower defense game in Godot using GDScript. This document outlines the scene hierarchy, nodes, scripts, and assets needed for a basic but functional version.

---

## 1. SCENE HIERARCHY & STRUCTURE

### Root Scene Architecture
```
Main (Node)
├── MainMenu (CanvasLayer)
│   └── MenuPanel (Control)
│       ├── BackgroundOverlay (ColorRect)
│       └── ContentContainer (VBoxContainer)
│           ├── TitleLabel (Label)
│           ├── InstructionsLabel (Label)
│           ├── StartButton (Button)
│           └── QuitButton (Button)
└── GameWorld (Node2D)
    ├── Background (Sprite2D or TileMap)
    ├── EnemyPath (Path2D) ← Draw the curve here in the editor
    │   └── (Enemies will be added here at runtime)
    ├── Towers (Node - container)
    ├── Projectiles (Node - container)
    └── UI (CanvasLayer)
        └── GUI elements
```

**Why this structure:**
- `Main` controls overall game flow (menu ↔ gameplay)
- `GameWorld` is the active level/play area
- Separate containers for Enemies, Towers, Projectiles make management easier
- `UI` on CanvasLayer keeps it above game objects
- `MainMenu` can be toggled on/off without destroying world

---

## 2. SCENE BREAKDOWN WITH NODES

### A. Main Menu Scene
**File:** `scenes/ui/main_menu.tscn`

**Nodes:**
- `MainMenu` (CanvasLayer)
  - `MenuPanel` (Control)
  - `BackgroundOverlay` (ColorRect) - semi-transparent background
  - `ContentContainer` (VBoxContainer) - centered layout
      - `TitleLabel` (Label)
      - `InstructionsLabel` (Lable)
      - `StartButton` (Button)
      - `QuitButton` (Button)

**Script:** `scripts/ui/main_menu.gd`
- `_on_start_pressed()` → emit signal or call main scene
- `_on_quit_pressed()` → quit game

---

### B. Game World Scene
**File:** `scenes/levels/game_world.tscn`

**Root Node:** `GameWorld` (Node2D)

**Children:**

#### 1. Background & Map Layer
- `Background` (Sprite2D)
    - Texture: background image (grass field, etc.)
    - Position: centered at (0, 0)

- `PathTileMap` (TileMap)
    - Tileset: path tiles (road/grass)
    - Defines where enemies walk
    - Visual representation of the path

- `TowerExclusionMap` (TileMap) *(optional for future placement)*
    - Marks valid/invalid tower placement areas
    - Helps with collision detection

#### 2. Game Objects Containers
- `Enemies` (Node)
    - Container for all enemy instances
    - Children instantiated at runtime

- `Towers` (Node)
    - Container for all tower instances
    - Children instantiated at runtime (or placed in editor)

- `Projectiles` (Node)
    - Container for all projectile instances
    - Children instantiated when towers shoot

#### 3. UI Layer
- `CanvasLayer` (CanvasLayer)
    - Keeps UI above game objects
    - `GameUI` (Control) - contains:
        - Health/Lives display
        - Money/Resources display
        - Wave counter
        - Game over screen

**Script:** `scripts/game_world.gd`
- Manages game state (playing, paused, game over)
- Handles enemy spawning (waves)
- Manages tower placement (if interactive)
- Connects signals between systems
- Tracks game progress

---

### C. Enemy Scene
**File:** `scenes/enemies/enemy.tscn`

**Nodes:**
- `Enemy` (PathFollow2D) - Root
    - `Sprite2D` - visual representation (balloon sprite)
    - `HitBox` (Area2D) - tower/projectile collision detection
      - `HitBoxShape` (CollisionShape2D) - circle shape

**Script:** `scripts/enemies/enemy.gd`

**Variables:**
- `max_health: int`
- `current_health: int`
- `speed: float`
- `path: Path2D` (reference to the path)
- `progress: float` (how far along the path)

**Key Functions:**
- `_physics_process(delta)` - move along path
- `take_damage(amount)` - reduce health
- `_on_reached_end()` - signal when leaving map
- `die()` - remove from scene, emit signal

**Signals:**
- `died` - emitted when health ≤ 0
- `reached_end` - emitted when exits map (deducts lives)
- `health_changed` - for UI updates

---

### D. Tower Scene
**File:** `scenes/towers/tower.tscn`

**Nodes:**
- `Tower` (Node2D) - Root
    - `BaseSprite` (Sprite2D) - stationary base
    - `TurretSprite` (Sprite2D) - rotates to aim
    - `DetectionArea` (Area2D) - detection radius
        - `DetectionShape` (CollisionShape2D) - circle shape
    - `FireTimer` (Timer) - controls shoot rate

**Script:** `scripts/towers/tower.gd`

**Variables:**
- `target_enemy: Enemy` (current target)
- `enemies_in_range: Array` (detected enemies)
- `damage: int`
- `fire_rate: float` (seconds between shots)
- `range: float` (detection radius)
- `projectile_scene: PackedScene` (preloaded projectile)

**Key Functions:**
- `_ready()` - set up signals/timers
- `_process(delta)` - rotate turret toward target
- `_on_area_entered(area)` - add enemy to detection
- `_on_area_exited(area)` - remove enemy from detection
- `_on_fire_timer_timeout()` - shoot projectile
- `get_closest_enemy()` - find best target
- `shoot()` - instantiate projectile

**Signals:**
- `enemy_entered_range`
- `enemy_left_range`
- `fired`

**Key Logic:**
- Uses `Area2D.look_at()` to rotate turret sprite toward nearest enemy
- Timer triggers projectile instantiation
- Only shoots if enemies are in range

---

### E. Projectile Scene
**File:** `scenes/projectiles/projectile.tscn`

**Nodes:**
- `Projectile` (Area2D) - Root
    - `PropjectileSprite` (Sprite2D) - visual (small circle or bullet)
    - `ProjectileShape` (CollisionShape2D) - circle

**Script:** `scripts/projectiles/projectile.gd`

**Variables:**
- `direction: Vector2` (movement direction)
- `speed: float`
- `damage: int`
- `target: Enemy` *(optional - for homing projectiles)*

**Key Functions:**
- `_ready()` - set initial velocity
- `_physics_process(delta)` - move projectile
- `_on_area_entered(area)` - handle collision with enemy
- `hit_enemy(enemy)` - call take_damage on enemy, destroy projectile

**Signals:**
- `hit` - emitted on collision

**Key Logic:**
- Moves in straight line in given direction
- Area2D signals detect collision with enemies
- Calls `enemy.take_damage()` directly
- Removes itself after hitting or going off-screen

---

## 3. DETAILED SCRIPT BREAKDOWN

### Main Game Controller
**File:** `scripts/main.gd`

**Responsibilities:**
- Handle menu ↔ game transitions
- Create/destroy GameWorld
- Manage global game state

```gdscript
extends Node

@onready var main_menu: CanvasLayer = $MainMenu

var game_world: Node2D = null
var game_world_scene: PackedScene = preload("res://scenes/levels/game_world.tscn")

func _ready():
	main_menu.start_game.connect(_on_start_game)
	main_menu.quit_game.connect(_on_quit_game)

func _on_start_game():
	main_menu.hide()
	
	# Create game world if it doesn't exist
	if game_world == null:
		game_world = game_world_scene.instantiate()
		game_world.game_over.connect(_on_game_over)
		add_child(game_world)

func _on_quit_game():
	get_tree().quit()

func _on_game_over():
	# Clean up game world
	if game_world:
		game_world.queue_free()
		game_world = null
	
	# Show menu again
	main_menu.show()

func return_to_menu():
	# Call this from pause menu or other places
	if game_world:
		game_world.queue_free()
		game_world = null
	main_menu.show()
```

### Main Menu
**File:** `scripts/ui/main_menu.gd`

**Responsibilities:**
- TODO - decribe

```gdscript
extends CanvasLayer

signal start_game
signal quit_game

@onready var start_button: Button = $MenuPanel/ContentContainer/StartButton
@onready var quit_button: Button = $MenuPanel/ContentContainer/QuitButton

func _ready():
	start_button.pressed.connect(_on_start_button_pressed)
	quit_button.pressed.connect(_on_quit_button_pressed)

func _on_start_button_pressed():
	start_game.emit()

func _on_quit_button_pressed():
	quit_game.emit()
```

---

### Game World Manager
**File:** `scripts/game_world.gd`

**Responsibilities:**
- Spawn enemies in waves
- Track game state
- Manage round progression
- Handle game over

```gdscript
extends Node2D

@export var enemy_scene: PackedScene
@export var wave_data: Array = [] # [{count: 5, delay: 0.5}, ...]

@onready var enemy_path: Path2D = $EnemyPath
@onready var towers_container: Node = $Towers
@onready var projectiles_container: Node = $Projectiles
@onready var ui: Control = $UI/GameUI

var current_wave: int = 0
var player_health: int = 20
var current_money: int = 100
var enemies_alive: int = 0
var is_spawning: bool = false # Prevent overlapping spawn calls

signal wave_started(wave_index: int)
signal wave_completed(wave_index: int)
signal game_over
signal money_changed(new_amount: int)
signal health_changed(new_health: int)

func _ready():
	# Add projectiles container to group for easy access from towers
	projectiles_container.add_to_group("projectile_container")
	
	# Start first wave (or wait for player input)
	spawn_wave(0)

func spawn_wave(wave_index: int):
	if is_spawning:
		return
	if wave_index >= wave_data.size():
		print("All waves complete!")
		return

	is_spawning = true
	current_wave = wave_index
	wave_started.emit(wave_index)

	var wave = wave_data[wave_index]
	var count = wave.get("count", 1)
	var delay = wave.get("delay", 0.5)

	for i in range(count):
		var enemy = enemy_scene.instantiate()

		# Connect signals BEFORE adding to tree
		enemy.died.connect(_on_enemy_died.bind(enemy))
		enemy.reached_end.connect(_on_enemy_reached_end.bind(enemy))

		enemy_path.add_child(enemy)
		enemies_alive += 1

		# Wait between spawns (except after last enemy)
		if i < count - 1:
			await get_tree().create_timer(delay).timeout

	is_spawning = false

func _on_enemy_died(enemy: Node):
	var reward = enemy.reward if "reward" in enemy else 0
	current_money += reward
	money_changed.emit(current_money)
	enemies_alive -= 1
	_check_wave_complete()

func _on_enemy_reached_end(enemy: Node):
	var damage = 1 # Or enemy.damage if enemies have variable damage
	player_health -= damage
	health_changed.emit(player_health)
	enemies_alive -= 1

	if player_health <= 0:
		game_over.emit()
	else:
		_check_wave_complete()

func _check_wave_complete():
	if enemies_alive <= 0 and not is_spawning:
		wave_completed.emit(current_wave)
		
		# Auto-start next wave (or wait for player input)
		if current_wave + 1 < wave_data.size():
			# Optional: add delay between waves
			await get_tree().create_timer(2.0).timeout
			spawn_wave(current_wave + 1)

func add_money(amount: int):
	current_money += amount
	money_changed.emit(current_money)

func spend_money(amount: int) -> bool:
	if current_money >= amount:
		current_money -= amount
		money_changed.emit(current_money)
		return true
	return false
```

---

### Enemy Script
**File:** `scripts/enemies/enemy.gd`

```gdscript
extends PathFollow2D

@export var max_health: int = 3
@export var speed: float = 100.0
@export var reward: int = 50

var current_health: int

signal died
signal reached_end
signal health_changed(new_health: int)

func _ready():
    current_health = max_health
    # Ensure loop is off so progress_ratio stops at 1.0
    loop = false
    # Add to group so projectiles can identify us
    add_to_group("enemies")

func _physics_process(delta: float):
    # Move along the path
    progress += speed * delta

    # Check if reached end of path
    if progress_ratio >= 1.0:
        reached_end.emit()
        queue_free()

func take_damage(amount: int):
    current_health -= amount
    health_changed.emit(current_health)
    if current_health <= 0:
        die()

func die():
    died.emit()
    queue_free()
```

---

### Tower Script
**File:** `scripts/towers/tower.gd`

```gdscript
extends Node2D

@export var damage: int = 1
@export var fire_rate: float = 1.0  # shots per second
@export var detection_range: float = 200.0

@onready var turret: Sprite2D = $TurretSprite
@onready var detection_area: Area2D = $DetectionArea # or $Area2D if you kept the old name
@onready var fire_timer: Timer = $FireTimer
var projectile_scene: PackedScene = preload("res://scenes/projectiles/projectile.tscn")

var enemies_in_range: Array = []
var target_enemy: Node2D = null # Changed type: enemy is PathFollow2D, which extends Node2D

signal fired

func _ready():
	# Set detection radius dynamically
	var collision_shape = detection_area.get_node("DetectionShape")
	var circle = CircleShape2D.new()
	circle.radius = detection_range
	collision_shape.shape = circle

	# Connect signals
	detection_area.area_entered.connect(_on_area_entered)
	detection_area.area_exited.connect(_on_area_exited)

	# Set fire rate
	fire_timer.wait_time = 1.0 / fire_rate
	fire_timer.timeout.connect(_on_fire_timer_timeout)
	fire_timer.start()

func _process(_delta: float):
	target_enemy = get_closest_enemy()

	if target_enemy:
		# Rotate turret to face target
		turret.look_at(target_enemy.global_position)

func _on_area_entered(area: Area2D):
	# CHANGED: The 'area' is the enemy's child Area2D (HitBox).
	# We need to get the enemy node (the parent PathFollow2D).
	var enemy = area.get_parent()
	
	# Verify it's actually an enemy using the group we added in enemy.gd
	if enemy.is_in_group("enemies"):
		enemies_in_range.append(enemy)

func _on_area_exited(area: Area2D):
	var enemy = area.get_parent()
	if enemy in enemies_in_range:
		enemies_in_range.erase(enemy)

func get_closest_enemy() -> Node2D:
	var closest: Node2D = null
	var min_distance: float = INF

	# Iterate over a copy to safely modify during iteration
	for enemy in enemies_in_range.duplicate():
		if not is_instance_valid(enemy):
			enemies_in_range.erase(enemy)
			continue

		var distance = global_position.distance_to(enemy.global_position)
		if distance < min_distance:
			min_distance = distance
			closest = enemy

	return closest

func _on_fire_timer_timeout():
	if target_enemy and is_instance_valid(target_enemy):
		shoot()

func shoot():
	var projectile = projectile_scene.instantiate()

	var direction = (target_enemy.global_position - global_position).normalized()

	projectile.global_position = global_position
	projectile.direction = direction
	projectile.damage = damage

	# Use group to find projectiles container (more robust)
	var container = get_tree().get_first_node_in_group("projectile_container")
	if container:
		container.add_child(projectile)

	fired.emit()
```

---

### Projectile Script
**File:** `scripts/projectiles/projectile.gd`

```gdscript
extends Area2D

@export var speed: float = 400.0
var direction: Vector2 = Vector2.ZERO
var damage: int = 1

signal hit

func _ready():
	area_entered.connect(_on_area_entered)

func _physics_process(delta: float):
	global_position += direction * speed * delta

	# Remove if off-screen
	if not get_viewport_rect().has_point(global_position):
		queue_free()

func _on_area_entered(area: Area2D):
	# CHANGED: 'area' is the enemy's child HitBox (Area2D).
	# We need the enemy itself (the parent PathFollow2D).
	var enemy = area.get_parent()

	if enemy.is_in_group("enemies"):
		if enemy.has_method("take_damage"):
			enemy.take_damage(damage)
		hit.emit()
		queue_free()
```

---

### UI Manager Script
**File:** `scripts/ui/game_ui.gd`

```gdscript
extends Control

@onready var health_label: Label = $VBoxContainer/HealthLabel
@onready var money_label: Label = $VBoxContainer/MoneyLabel
@onready var wave_label: Label = $VBoxContainer/WaveLabel

var game_world: Node2D

func _ready():
	game_world = get_parent().get_parent()  # Navigate to GameWorld

	# Connect signals instead of polling
	game_world.wave_started.connect(_on_wave_started)
	game_world.wave_completed.connect(_on_wave_completed)
	game_world.game_over.connect(_on_game_over)
	game_world.money_changed.connect(_on_money_changed)
	game_world.health_changed.connect(_on_health_changed)

	# Set initial values
	_update_health(game_world.player_health)
	_update_money(game_world.current_money)
	_update_wave(game_world.current_wave)

func _update_health(value: int):
	health_label.text = "Health: %d" % value

func _update_money(value: int):
	money_label.text = "Money: $%d" % value

func _update_wave(value: int):
	wave_label.text = "Wave: %d" % (value + 1)  # Display 1-indexed

func _on_wave_started(wave_index: int):
	_update_wave(wave_index)

func _on_wave_completed(wave_index: int):
	pass  # Could show "Wave Complete!" message

func _on_game_over():
	# Show game over screen
	pass

func _on_money_changed(new_amount: int):
	_update_money(new_amount)

func _on_health_changed(new_health: int):
	_update_health(new_health)
```

---

## 4. ASSETS NEEDED

### Graphics
- [ ] Background image (grass field, landscape)
- [ ] Path tileset (road/grass tiles)
- [ ] Tower base sprite
- [ ] Tower turret sprite
- [ ] Enemy sprite (balloon or basic circle)
- [ ] Projectile sprite (bullet or dart)
- [ ] UI buttons and backgrounds

### Tilesets
- [ ] Main path tileset (TileSet resource)
- [ ] Optional: tower placement overlay tileset

### Sounds (Optional for MVP)
- [ ] Tower fire sound
- [ ] Enemy die sound
- [ ] UI click sounds

---

## 5. GAME FLOW DIAGRAM

```
Start Game
    ↓
Main Menu Scene
    ↓ (Start Button)
GameWorld Scene Loaded
    ↓
Wave 0 Spawns
    ↓ (Loop)
Enemies spawn → move along path → reach end or die
    ↓ (When all enemies dead)
Wave Complete → Prepare next wave
    ↓ (Player loses all lives)
Game Over → Return to main menu
```

---

## 6. IMPLEMENTATION ROADMAP (Recommended Order)

### Phase 1: Core Setup
1. [ ] Create Main scene with menu UI
2. [ ] Create GameWorld scene structure
3. [ ] Set up TileMaps for background and path

### Phase 2: Enemies
4. [ ] Create Enemy scene and basic movement script
5. [ ] Implement path following (PathFollow2D)
6. [ ] Test enemy spawning in waves

### Phase 3: Towers
7. [ ] Create Tower scene with base and turret
8. [ ] Implement detection (Area2D)
9. [ ] Implement turret aiming toward enemies

### Phase 4: Projectiles
10. [ ] Create Projectile scene
11. [ ] Implement projectile shooting from towers
12. [ ] Implement collision and damage

### Phase 5: Game Management
13. [ ] Implement wave spawning system
14. [ ] Add health/lives tracking
15. [ ] Add game over conditions

### Phase 6: Polish
16. [ ] Add UI (health, money, wave counter)
17. [ ] Add visual feedback (animations, effects)
18. [ ] Add sounds (optional)

---

## 7. KEY GODOT ELEMENTS & WHEN TO USE THEM

| Element | Purpose | When to Use |
|---------|---------|-------------|
| **Node2D** | 2D scene root | Game world, towers, enemies |
| **Sprite2D** | Display images | All visual elements |
| **CharacterBody2D** | Physics movement | Enemies, player (if added later) |
| **Area2D** | Collision detection | Tower detection range, projectiles |
| **CollisionShape2D** | Shape for collisions | Paired with physics/detection nodes |
| **Timer** | Time-based events | Tower fire rate, wave delays |
| **PathFollow2D** | Move along a path | Enemy movement along predetermined route |
| **Path2D** | Define a path | Visual representation of enemy route |
| **TileMap** | Grid-based visuals | Background, path, terrain |
| **CanvasLayer** | Layering control | Keep UI above game objects |
| **Control** | UI root node | Buttons, labels, containers |
| **Signals** | Event communication | Tower firing, enemy death, wave completion |
| **Groups** | Node categorization | Tag enemies so projectiles identify them |

---

## 8. IMPORTANT GDSCRIPT PATTERNS FOR THIS PROJECT

### Pattern 1: Preloading Scenes
```gdscript
var enemy_scene: PackedScene = preload("res://scenes/enemies/enemy.tscn")
var enemy = enemy_scene.instantiate()
add_child(enemy)
```

### Pattern 2: Using Signals for Decoupling
```gdscript
# In enemy.gd
signal died

func die():
    died.emit()

# In game_world.gd
enemy.died.connect(_on_enemy_died)
```

### Pattern 3: Groups for Identification
```gdscript
# In enemy.gd _ready()
add_to_group("enemies")

# In projectile.gd
if area.is_in_group("enemies"):
    area.take_damage(damage)
```

### Pattern 4: Using @onready for Node References
```gdscript
@onready var sprite = $Sprite2D
@onready var collision = $CollisionShape2D
```

### Pattern 5: Export Variables for Tweaking
```gdscript
@export var speed = 100.0
@export var damage = 1
```

---

## 9. COMMON ISSUES & SOLUTIONS

| Issue | Cause | Solution |
|-------|-------|----------|
| Enemies not moving | PathFollow2D not set up | Verify Path2D exists, PathFollow2D references it |
| Tower not shooting | Area2D detection not working | Check collision layers/masks are compatible |
| Projectiles passing through enemies | No collision signal connection | Connect `area_entered` signal on projectile |
| Turret not rotating | Rotation not applied to sprite | Verify turret sprite is separate child node |
| Game crashes on enemy death | Accessing invalid enemy reference | Use `is_instance_valid()` before accessing |
| UI not visible | CanvasLayer z-index too low | Increase z-index or move to proper layer |

---

## 10. NEXT STEPS AFTER BASIC VERSION

Once MVP is working:
- [ ] Multiple tower types with different damage/speed/range
- [ ] Tower placement system (click to place)
- [ ] Upgrade system (increase damage, range, speed)
- [ ] Multiple enemy types
- [ ] Better graphics/animations
- [ ] Sound effects
- [ ] Difficulty scaling
- [ ] High score tracking
