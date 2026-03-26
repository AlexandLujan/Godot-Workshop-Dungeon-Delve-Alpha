# Godot Workshop Lab Document (Alpha)

Description: In this workshop, you will be tasked with building the baseline features required for a Dungeon Crawler like game in Godot 4.6, make sure you have that version of Godot installed before proceeding any further with the workshop.

Link to download Godot can be found below:
- **Windows:** https://godotengine.org/download/windows/
- **MacOS:** https://godotengine.org/download/macos/

Once you have completed the download, go ahead and open Godot.

# Preface
This project will be using a dungeon crawler asset sheet that has already been added to this repository, however feel free to use a different one if you'd like. Make sure to keep in mind that you'll have to account for the pixel dimensions of the asset sheet you are using since this one is going off of one that uses (16x16).

## 1. File Hierarchy
We will be using a very specific directory structure for this project, so it is imperative that it is followed through for Alpha and beyond.

In `res://`, create the following structure:
```notepad
Scenes/
   Main/
   Player/
   Enemies/
   World/
   UI/

Scripts/
   Main/
   Player/
   Enemies/
   World/
   UI/
```

This will organize each scene and script separately to make it easier to access them throughout the project.

## 2. Input Map Setup
We will need to set up player inputs so that Godot can recognize when the player activates an action in the game.

Go to: Project -> Project Settings -> Input Map
This can be found at the top of the Godot Engine Interface.

Add the following actions with the specified key bindings:
- move_up (W & Up Arrow)
- move_down (S & Down Arrow)
- move_left (A & Left Arrow)
- move_right (D & Right Arrow)
- attack (Space & Left Mouse Button)
- restart (R)

Once that is done, save project settings.

NOTE: You can also add bindings for Xbox/PlayStation Controller if you'd like.

## 3. Building the Player Scene
The first scene we will build is the Player Scene, which will be our main character that the player will control across the map.

Create a new scene.

Choose `CharacterBody2D` as it's node.

Rename it to `Player`.

From there, we also need to add the children nodes that represent different attributes of our character, everything including the sprite, hitbox, camera, and other esoteric/specific functionality nodes for our character.

Add more nodes to match the tree below:
```notepad
Player (CharacterBody2D)
   Sprite2D
   CollisionShape2D
   Camera2D
   Hurtbox (Area2D)
      CollisionShape2D
   AttackPivot (Node2D)
      AttackHitbox (Area2D)
         CollisionShape2D
   AttackCooldownTimer (Timer)
   InvincibilityTimer (Timer)
```

NOTE: A neat little trick to achieve a very quick "Camera Follow" feature is to make the Camera2D Node a child of the CharacterBody2D parent scene, which is what we are using for this scene.

Once you are done, save the asset as `res://Scenes/Player/Player.tscn`, Godot may ask you if you want to make this your root scene, if it does, go ahead and accept it for now, we can change it later.

Next thing we need to do is to assign a sprite to `Sprite2D`, this is where we will use the provided Sprite Sheet.

Go into the DungeonTileSet and set the Sprite to `knight_f_idle_anim_f0.png`.

Now from there, we need to set the Hurtbox's shape so it will actually detect collisions. Set the CollisionShape2D node to `RectangleShape2D` in the Inspector.

Go ahead and do so with the AttackHitbox's `CollisionShape2D` as well, but use a `RectangleShape2D` with the size at about 12x12.

Finally, we need the actual collision detection for the character itself. Under the `CharacterBody2D`, make sure it encaspulates most of the character.

Also, make sure the `Hurtbox` has a smaller `CollisionShape2D` than the player hitbox.

Next, let's worry about the timers.

For the `AttackCooldownTimer`, set One Shot to On and Wait Time to 0.2.

For the `InvincibilityTimer`, set One Shot to On and Wait Time to 0.6.

Now our Node Structure is complete, let's go ahead and move on to scripting the player.

## 3. Player Script
Create a new script and attach it to the Player. Name it `player.gd`.

Paste the following code in the script:
```gdscript
extends CharacterBody2D

signal health_changed(current_health: int, max_health: int)
signal player_died
signal attack_started
signal attack_ended

@export var move_speed: float = 120.0
@export var max_health: int = 3
@export var attack_duration: float = 0.18
@export var attack_damage: int = 1
@export var knockback_strength: float = 180.0

@onready var hurtbox: Area2D = $Hurtbox
@onready var attack_pivot: Node2D = $AttackPivot
@onready var attack_hitbox: Area2D = $AttackPivot/AttackHitbox
@onready var attack_collision: CollisionShape2D = $AttackPivot/AttackHitbox/CollisionShape2D
@onready var attack_cooldown_timer: Timer = $AttackCooldownTimer
@onready var invincibility_timer: Timer = $InvincibilityTimer
@onready var sprite: Sprite2D = $Sprite2D

var current_health: int
var move_input: Vector2 = Vector2.ZERO
var facing_direction: Vector2 = Vector2.DOWN
var is_attacking: bool = false
var is_invincible: bool = false
var is_dead: bool = false

func _ready() -> void:
	current_health = max_health
	attack_collision.disabled = true
	add_to_group("player")
	health_changed.emit(current_health, max_health)

func _physics_process(_delta: float) -> void:
	if is_dead:
		velocity = Vector2.ZERO
		move_and_slide()
		return

	handle_input()
	handle_movement()
	handle_attack()
	move_and_slide()

func handle_input() -> void:
	move_input = Input.get_vector("move_left", "move_right", "move_up", "move_down")

	if move_input != Vector2.ZERO:
		facing_direction = move_input.normalized()
		update_attack_direction()

func handle_movement() -> void:
	if is_attacking:
		velocity = Vector2.ZERO
	else:
		velocity = move_input * move_speed

func handle_attack() -> void:
	if Input.is_action_just_pressed("attack") and not is_attacking and attack_cooldown_timer.is_stopped():
		start_attack()

func start_attack() -> void:
	is_attacking = true
	attack_collision.disabled = false
	attack_started.emit()

	await get_tree().create_timer(attack_duration).timeout

	if is_instance_valid(self):
		attack_collision.disabled = true
		is_attacking = false
		attack_cooldown_timer.start()
		attack_ended.emit()

func update_attack_direction() -> void:
	var offset_distance := 16.0
	var direction := facing_direction.normalized()

	if abs(direction.x) > abs(direction.y):
		if direction.x > 0:
			attack_pivot.position = Vector2(offset_distance, 0)
		else:
			attack_pivot.position = Vector2(-offset_distance, 0)
	else:
		if direction.y > 0:
			attack_pivot.position = Vector2(0, offset_distance)
		else:
			attack_pivot.position = Vector2(0, -offset_distance)

func take_damage(amount: int, source_position: Vector2 = global_position) -> void:
	if is_dead or is_invincible:
		return

	current_health -= amount
	current_health = max(current_health, 0)
	health_changed.emit(current_health, max_health)

	if current_health <= 0:
		die()
		return

	is_invincible = true
	invincibility_timer.start()

	var knockback_dir := (global_position - source_position).normalized()
	if knockback_dir == Vector2.ZERO:
		knockback_dir = Vector2.DOWN
	velocity = knockback_dir * knockback_strength

	modulate = Color(1, 0.6, 0.6)

func die() -> void:
	is_dead = true
	attack_collision.disabled = true
	velocity = Vector2.ZERO
	player_died.emit()
	modulate = Color(0.5, 0.5, 0.5)

func _on_invincibility_timer_timeout() -> void:
	is_invincible = false
	modulate = Color(1, 1, 1)
```

Now, let's program the hitbox logic.
```gdscript
extends Area2D

@export var damage: int = 1

func _ready() -> void:
	area_entered.connect(_on_area_entered)
	body_entered.connect(_on_body_entered)

func _on_area_entered(area: Area2D) -> void:
	if area.get_parent() and area.get_parent().has_method("take_damage"):
		area.get_parent().take_damage(damage)

func _on_body_entered(body: Node) -> void:
	if body.has_method("take_damage"):
		body.take_damage(damage)
```

Finally, let's connect the `InvincibiliyTimer` Signal.
- Go to Node Tab
- Double Click Timeout
- Connect it to Player
- Choose Method `_on_invincibility_timer_timeout`

The signal should now be connected.

## 4. Slime Scene
Now, we will start making the Slime enemy, which follows a similar structure from our `Player`.

Go ahead and make a new Scene and choose `CharacterBody2D`. Rename the scene to slime.

Then, add more Nodes to the Slime as follows:
```gdscript
Slime (CharacterBody2D)
   Sprite2D
   CollisionShape2D
   Hurtbox (Area2D)
      CollisionShape2D
   DetectionArea (Area2D)
      CollisionShape2D
```

Save the scene as: `res://Scenes/Enemies/Slime.tscn`

Now we have the node tree structure set, let's go ahead and tweak our nodes.

Like the `Player`, the Enemy will need the same tweaks.

Make the following tweaks to the nodes:
- `CollisionShape2D`: Body sized rectangle or capsule, this will represent the hitbox of the actual character, we will still have the Hurtbox to deal with as well. Speaking of...
- `Hurtbox/CollisionShape2D`: Body-sized circle or rectangle, make sure it is smaller than the character's Collision Shape.
- `DetectionArea/CollisionShape2D`: Larger circle with a radius around 64 to 96, feel free to experiment with this since this is an area that can affect the difficulty of the game itself.

## 5. Slime Script
Now, we need to program the logic of the Slime character into our Slime Scene, we also need to start worrying about which groups it is in as well as all of the data that goes into the Slime itself.

Go ahead and create a new script and name it `slime.gd`.

Make sure it is in the following path: `res://Scripts/Enemies/slime.gd`
```gdscript
extends CharacterBody2D

signal slime_died(slime: Node)

@export var move_speed: float = 55.0
@export var max_health: int = 2
@export var contact_damage: int = 1

@onready var detection_area: Area2D = $DetectionArea
@onready var hurtbox: Area2D = $Hurtbox
@onready var sprite: Sprite2D = $Sprite2D

var current_health: int
var player: Node2D = null
var is_dead: bool = false

func _ready() -> void:
	current_health = max_health
	add_to_group("enemy")
	detection_area.body_entered.connect(_on_detection_area_body_entered)
	detection_area.body_exited.connect(_on_detection_area_body_exited)
	hurtbox.body_entered.connect(_on_hurtbox_body_entered)

func _physics_process(_delta: float) -> void:
	if is_dead:
		velocity = Vector2.ZERO
		move_and_slide()
		return

	if player and is_instance_valid(player):
		var direction := (player.global_position - global_position).normalized()
		velocity = direction * move_speed
	else:
		velocity = Vector2.ZERO

	move_and_slide()

func take_damage(amount: int) -> void:
	if is_dead:
		return

	current_health -= amount
	modulate = Color(1, 0.5, 0.5)

	if current_health <= 0:
		die()
	else:
		await get_tree().create_timer(0.08).timeout
		if is_instance_valid(self):
			modulate = Color(1, 1, 1)

func die() -> void:
	if is_dead:
		return

	is_dead = true
	slime_died.emit(self)
	queue_free()

func _on_detection_area_body_entered(body: Node) -> void:
	if body.is_in_group("player"):
		player = body

func _on_detection_area_body_exited(body: Node) -> void:
	if body == player:
		player = null

func _on_hurtbox_body_entered(body: Node) -> void:
	if body.is_in_group("player") and body.has_method("take_damage"):
		body.take_damage(contact_damage, global_position)
```

Save the scene and return to the game.

## 6. Room Scene
Next, we need to build our world logic, everything from the scrpiting to the node logic.

Go ahead and create a new scene out of a generic `Node2D` and name it to `Room`.

Save it as `res://Scenes/World/Room.tscn`

We can then add our Player and Slime scenes as children, and since we won't be controlling our Slime Character, we can actually duplicate the scene itself to add more Slime enemies.

Paste this script below into `room.tscn`:
```gdscript
extends Node2D

signal room_cleared

@export var auto_collect_slimes: bool = true

var enemies: Array[Node] = []
var cleared: bool = false

func _ready() -> void:
	if auto_collect_slimes:
		for child in get_children():
			if child.has_signal("slime_died"):
				register_enemy(child)

	check_if_cleared()

func register_enemy(enemy: Node) -> void:
	if enemy in enemies:
		return

	enemies.append(enemy)
	enemy.connect("slime_died", Callable(self, "_on_enemy_died"))

func _on_enemy_died(enemy: Node) -> void:
	enemies.erase(enemy)
	check_if_cleared()

func check_if_cleared() -> void:
	if cleared:
		return

	enemies = enemies.filter(func(enemy): return is_instance_valid(enemy))

	if enemies.is_empty():
		cleared = true
		room_cleared.emit()
```

Save the scene and return to the main window.

## 7. Exit Door Scene
The next important feature we need to implement is the way for a player to complete the game, which is where our Exit Door scene comes in.

Create a new scene out of an `Area2D` node and name it `ExitDoor`

Make sure the scene is saved as: `res://Scenes/World/ExitDoor.tscn`.

Make sure you also have a `Sprite2D` and `CollisionShape2D` node as children of the ExitDoor scene, these are to represent the appearance of the door as well as a hitbox to act as our logic container.

From there, create a new script and name it `exit_door.gd`.

Save it to `res://Scripts/World/exit_door.gd`

Then, inside the script, go ahead and paste this:
```gdscript
extends Area2D

signal player_won

@export var locked: bool = true

func _ready() -> void:
	body_entered.connect(_on_body_entered)
	update_visual()

func unlock() -> void:
	locked = false
	update_visual()

func lock() -> void:
	locked = true
	update_visual()

func update_visual() -> void:
	if locked:
		modulate = Color(1, 0.4, 0.4)
	else:
		modulate = Color(0.4, 1, 0.4)

func _on_body_entered(body: Node) -> void:
	if locked:
		return

	if body.is_in_group("player"):
		player_won.emit()
```

Save the script and then return to the 2D Viewport (look, I am running out of ways to say, "save and return", give me a break)

## 8. HUD
Now, let's go ahead and make a basic HUD display for our game, there is also a second .png file that contains all of the elements arranged in a sprite sheet, it is called `dungeonuiv1.png`.

Go ahead and create a new scene out of a `CanvasLayer` node, this node is under the umbrella of `Control` Nodes which are the nodes used to display in-game HUD.

Rename this scene to `HUD`.

From there, we will need a few more nodes to act as children nodes to our HUD scene, which will each serve a different purpose.

Follow the following hierarchy for the HUD scene:
```notepad
HUD (CanvasLayer)
   MarginContainer
      VBoxContainer
         HealthLabel
         StatusLabel
         RestartLabel
```

NOTE: Each of the Labels are using a `Label` node, which also belongs to the Control umbrella of Godot Nodes.

Save this scene as `res://Scenes/UI/HUD.tscn`

Then, let's go ahead and configure the HUD layout using the Inspector.

With the MarginContainer selected, set the Anchors to the top-left or full rect if preferred. This setting will be under the `Layout` Dropdown Menu in the Inspector.

From there, select the `HealthLabel` node and set the Default Label text to "HP: 3 / 3"

Leave the other two empty, we don't need to worry about those for right now.

Finally, create a new script and call it `hud.gd`, make sure it is saved as `res://Scripts/UI/hud.gd`

Paste the following into the script:
```gdscript
extends CanvasLayer

@onready var health_label: Label = $MarginContainer/VBoxContainer/HealthLabel
@onready var status_label: Label = $MarginContainer/VBoxContainer/StatusLabel
@onready var restart_label: Label = $MarginContainer/VBoxContainer/RestartLabel

func _ready() -> void:
	status_label.text = ""
	restart_label.text = ""

func update_health(current_health: int, max_health: int) -> void:
	health_label.text = "HP: %d / %d" % [current_health, max_health]

func show_game_over() -> void:
	status_label.text = "Game Over"
	restart_label.text = "Press R to Restart"

func show_victory() -> void:
	status_label.text = "You Win!"
	restart_label.text = "Press R to Restart"

func clear_status() -> void:
	status_label.text = ""
	restart_label.text = ""
```

Save the scene and return to the 2D Worldview.

## 9. Main Scene
This is the most important part of the workshop so far, we need to set our Main scene. This scene will be contain the ENTIRE game's logic, scenes, nodes, etc.

Create a new scene out of a generic `Node2D` and rename it to Main. From there, select this scene and press the Play Button at the top middle of the screen, it will then ask you if you want to make this your root scene, go ahead and set it as such.

Then, immediately stop the game.

Go ahead and add all of the existing Nodes as children if the engine didn't automatically do so.

Your project hierarchy should look like this:
```gdscript
Main (Node2D)
   Room
   ExitDoor
   Player
   HUD
```

From there, add a sprite for each node that doesn't have a sprite to represent it, this is likely the case for our Room scene.

NOTE: Another way to do this is to create a GridMap, we will worry about this in Beta so stay tuned.

Now, go ahead and press play.

You'll likely notice when you move the player, it passes right through each and every wall, and that is because we don't have any way of letting the game know that the player is hitting a wall to prevent the player from going any further beyond.

So, let's go ahead and fix that.

For a quick collision logic, add some `StaticBody2D` nodes and connect them directly to main. They should have a `CollisionShape2D` and `Sprite2D` node attached as children to represent specific functionality.

You can then duplicate this for each wall surrounding the room. (Top, Bottom, Left, and Right)

Create a script and call it `main.gd`, go ahead and attach that script to the Main scene and save it as `res://Scripts/Main/main.gd`

From there, go ahead and paste this script inside `main.gd`:
```gdscript
extends Node2D

@onready var player = $Player
@onready var exit_door = $ExitDoor
@onready var hud = $HUD
@onready var room = $Room

var game_finished: bool = false

func _ready() -> void:
	player.health_changed.connect(_on_player_health_changed)
	player.player_died.connect(_on_player_died)
	exit_door.player_won.connect(_on_player_won)

	if room and room.has_signal("room_cleared"):
		room.room_cleared.connect(_on_room_cleared)

	hud.update_health(player.current_health, player.max_health)

func _process(_delta: float) -> void:
	if game_finished and Input.is_action_just_pressed("restart"):
		get_tree().reload_current_scene()

func _on_player_health_changed(current_health: int, max_health: int) -> void:
	hud.update_health(current_health, max_health)

func _on_player_died() -> void:
	game_finished = true
	hud.show_game_over()

func _on_player_won() -> void:
	game_finished = true
	hud.show_victory()

func _on_room_cleared() -> void:
	exit_door.unlock()
```

Save the script and return to the 2D Viewport.

## 10. Rearrangement
Now, let's arrange all of our scenes to a starting game state, in other words, let's arrange them to what the player would see when they press Play.

In `Main.tscn`, set rough positions to the following:
- `Player`: Near center
- `Room`: At origin
- `Slime(s)`: Somewhere a short distance away from the player.
- `ExitDoor`: Near edge of room

## 11. Playtesting
Now, let's go ahead and playtest our game. Use the following checklist to markdown if everything works correctly:

- [ ] Player moves with WASD
- [ ] Diagonal movement works
- [ ] Camera follows the player

- [ ] Player cannot move through walls

- [ ] Slime starts still
- [ ] Slime chases when player enters range

- [ ] Touching slime reduces HP.
- [ ] Player flashes when hit by slime.
- [ ] Repeated damage is delayed by invincibility timer.

- [ ] Attack registers logically
- [ ] Hitbox damages slime
- [ ] Slime dies after enough hits

- [ ] Kill all slimes
- [ ] Exit changes from closed to open

- [ ] When player touches unlocked exit, the win text appears.
- [ ] Lose all HP leads to Game Over text.
- [ ] Pressing R restarts the game.

## 12. Common Bugs
A. Player does not move
- Check if input map names exactly match the script.
- Check if script is attached to `Player`
- Check if you are running `Main.tscn` and not any other script.

B. Timer method missing
- Check if the signal is connected.

C. Slime never chases
- Check if Detection Area has a collision shape
- Check if Player has a Physics Body (it should have a `CollisionShape2D` that is right below the `CharacterBody2D` as a child)
- Check if player is in the player group
- Check if slime script is attached to root `Slime`

D. Attack never hits
- Check if `AttackHitbox` script is attached.
- Check if `AttackHitbox` has a `CollisionShape2D`
- Check if Attack collision shape is positioned in front of a player
- Check if Enemy has `take_damage()` on root.
- Check if player attack hitbox is enabled briefly during attack.

E. Exit never unlocks
- Check if slimes are children of room
- Check if `Room` script is attached
- Check if `Room.room_cleared()` is connected in `Main`
- Check if `Slime` root has `slime_died` signal and emits it.

F. Restart does nothing
- Check if `restart` action exists in Input Map
- Check if R is bound correctly

Continued in Beta.
