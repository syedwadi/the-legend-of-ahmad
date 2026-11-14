extends Node2D
# Game manager: builds 10 procedural levels + the boss arena, handles HUD/flow.
# Pixel look: the whole game world is rendered into a low-res SubViewport and
# scaled up with nearest-neighbor filtering.
# Meteors fall from the sky and get harder with each level.
# Ground is drawn as cratered moon rock. Some platforms are icy or bouncy.

const TS := 32
const MAX_LEVEL := 10
const PIXEL := 3   # pixel size: 2 = subtle, 3 = chunky, 4/6 = very blocky
const METEORS := true            # master switch for meteors
const METEORS_IN_BOSS := true    # meteors also fall in the boss arena (slower)
const ICE_GLIDE := 2.2           # lower = more slippery (try 1.0 - 4.0)
const BOUNCE_SPEED := 700.0      # launch speed on bouncy platforms

var level := 1
var deaths := 0
var won := false
var busy := false
var boss_mode := false
var spawn := Vector2.ZERO
var world: Node2D
var pix: SubViewport   # low-res viewport that holds the world
var player = null   # untyped on purpose (custom script instance)
var boss = null     # untyped on purpose (custom script instance)
var hud: Label
var msg: Label

# meteor state
var meteors: Array = []      # each entry: {node, rock, vel, spin}
var meteor_timer := 0.0
var level_time := 0.0

# surface state
var slide_vx := 0.0
var bounce_cd := 0.0

func _ready() -> void:
	# run after the player's physics, so we can react to what it just touched
	process_physics_priority = 100
	_setup_input()
	_setup_pixel_view()
	var layer := CanvasLayer.new()
	add_child(layer)
	hud = Label.new()
	hud.position = Vector2(14, 10)
	hud.add_theme_font_size_override("font_size", 20)
	layer.add_child(hud)
	msg = Label.new()
	msg.position = Vector2(0, 220)
	msg.size = Vector2(1152, 140)
	msg.horizontal_alignment = HORIZONTAL_ALIGNMENT_CENTER
	msg.add_theme_font_size_override("font_size", 36)
	layer.add_child(msg)
	start_level(1)

# Low-res render target, drawn behind the HUD and scaled up crisply.
func _setup_pixel_view() -> void:
	var pl := CanvasLayer.new()
	pl.layer = -1
	add_child(pl)
	var container := SubViewportContainer.new()
	container.stretch = true
	container.stretch_shrink = PIXEL
	container.texture_filter = CanvasItem.TEXTURE_FILTER_NEAREST
	container.mouse_filter = Control.MOUSE_FILTER_IGNORE
	container.set_anchors_preset(Control.PRESET_FULL_RECT)
	pl.add_child(container)
	pix = SubViewport.new()
	pix.handle_input_locally = false
	pix.snap_2d_transforms_to_pixel = true
	pix.snap_2d_vertices_to_pixel = true
	pix.render_target_update_mode = SubViewport.UPDATE_ALWAYS
	container.add_child(pix)

func _setup_input() -> void:
	var map := {"move_left": KEY_LEFT, "move_right": KEY_RIGHT, "jump": KEY_UP}
	for a in map:
		if not InputMap.has_action(a):
			InputMap.add_action(a)
		var ev := InputEventKey.new()
		ev.physical_keycode = map[a]
		InputMap.action_add_event(a, ev)

func _input(e: InputEvent) -> void:
	if e is InputEventKey and e.pressed and not e.echo and e.keycode == KEY_R:
		if busy and not won:
			return
		if won:
			deaths = 0
			start_level(1)
		else:
			start_level(level)

func _process(_d: float) -> void:
	if won:
		hud.text = "VICTORY!   Deaths: %d" % deaths
	elif boss_mode and is_instance_valid(boss) and is_instance_valid(player):
		hud.text = "BOSS HP: %d    YOU: %s" % [boss.hp, "♥".repeat(maxi(player.health, 0))]
	else:
		hud.text = "LEVEL %d / %d    Deaths: %d" % [mini(level, MAX_LEVEL), MAX_LEVEL, deaths]

func _physics_process(delta: float) -> void:
	_update_surfaces(delta)
	_update_meteors(delta)

# ---------------------------------------------------------------- flow
func start_level(n: int) -> void:
	level = n
	busy = false
	won = false
	msg.text = ""
	boss_mode = n > MAX_LEVEL
	meteors.clear()
	meteor_timer = 1.5     # short grace period before the first meteor
	level_time = 0.0
	slide_vx = 0.0
	bounce_cd = 0.0
	if is_instance_valid(world):
		world.queue_free()
	world = Node2D.new()
	pix.add_child(world)   # world lives inside the low-res viewport
	var right_edge: float = _build_boss_arena() if boss_mode else _build_level(n)

	var player_script = load("res://player.gd")
	player = player_script.new()
	player.position = spawn
	world.add_child(player)
	player.died.connect(_on_died)
	player.cam.zoom = Vector2.ONE / float(PIXEL)   # same view area, fewer pixels
	player.cam.limit_left = 0
	player.cam.limit_top = 0
	player.cam.limit_right = int(right_edge)
	player.cam.limit_bottom = 648 if boss_mode else 24 * TS
	player.cam.reset_smoothing()

	if boss_mode:
		player.health = 3
		var boss_script = load("res://boss.gd")
		boss = boss_script.new()
		boss.player = player
		boss.position = Vector2(900, 508)
		boss.defeated.connect(_on_boss_defeated)
		world.add_child(boss)
	else:
		boss = null
		player.fall_y = 26.0 * TS
		if n == 1:
			var hint := "←  →  move      ↑  jump      Right Shift  dash"
			msg.text = hint
			await get_tree().create_timer(4.0).timeout
			if msg.text == hint:
				msg.text = ""

func _on_died() -> void:
	if busy:
		return
	busy = true
	deaths += 1
	msg.text = "You died!"
	await get_tree().create_timer(0.9).timeout
	start_level(level)

func _on_goal(b: Node) -> void:
	if b != player or busy:
		return
	busy = true
	msg.text = "Get ready for the BOSS!" if level == MAX_LEVEL else "Level %d complete!" % level
	await get_tree().create_timer(1.2).timeout
	start_level(level + 1)

func _on_boss_defeated() -> void:
	won = true
	busy = true
	msg.text = "YOU WIN!\nPress R to play again"

# ---------------------------------------------------------------- slippery / bouncy surfaces
# Looks at what the player is standing on (after the player's own physics
# ran this frame) and applies the ice glide or the bounce.
func _update_surfaces(delta: float) -> void:
	if not is_instance_valid(player) or not (player is CharacterBody2D):
		return
	bounce_cd = maxf(bounce_cd - delta, 0.0)

	var surf := ""
	if player.is_on_floor():
		for i in player.get_slide_collision_count():
			var c = player.get_slide_collision(i)
			if c.get_normal().y < -0.7:
				var obj = c.get_collider()
				if obj != null and obj.has_meta("surface"):
					surf = str(obj.get_meta("surface"))
					break

	var vel: Vector2 = player.velocity

	if surf == "ice":
		# ease the effective speed toward the player's input speed
		var k := 1.0 - exp(-ICE_GLIDE * delta)
		slide_vx = lerpf(slide_vx, vel.x, k)
		var extra := slide_vx - vel.x
		if absf(extra) > 0.5:
			var hit = player.move_and_collide(Vector2(extra * delta, 0.0))
			if hit != null:
				slide_vx = 0.0
	else:
		slide_vx = vel.x

	if surf == "bounce" and bounce_cd <= 0.0:
		vel.y = -BOUNCE_SPEED
		player.velocity = vel
		bounce_cd = 0.25

# Deterministic (does not touch the level's random generator, so the level
# layouts stay the same). Returns "", "ice" or "bounce".
func _surface_for(n: int, x: int, w: int) -> String:
	if n < 2 or w < 4:
		return ""
	var hsh := absi((x * 2654435761 + n * 40503) % 100)
	var ice_limit := 10 + n * 2
	if hsh < ice_limit:
		return "ice"
	if n >= 3 and hsh < ice_limit + 12:
		return "bounce"
	return ""

# ---------------------------------------------------------------- meteors
# Seconds between meteors: shrinks with the level and with time spent in it.
func _meteor_interval() -> float:
	var t := clampf(1.7 - level * 0.12 - level_time * 0.006, 0.35, 1.7)
	if boss_mode:
		t *= 1.6
	return t

func _update_meteors(delta: float) -> void:
	if not METEORS or not is_instance_valid(player) or not is_instance_valid(world):
		return
	var cam_c: Vector2 = player.cam.get_screen_center_position()

	# move existing meteors, clean up the ones that left the screen
	for i in range(meteors.size() - 1, -1, -1):
		var m: Dictionary = meteors[i]
		var node = m["node"]
		if not is_instance_valid(node):
			meteors.remove_at(i)
			continue
		node.position += m["vel"] * delta
		var rock = m["rock"]
		if is_instance_valid(rock):
			rock.rotation += m["spin"] * delta
		if node.position.y > cam_c.y + 324.0 + 150.0:
			node.queue_free()
			meteors.remove_at(i)

	if busy or won:
		return
	if boss_mode and not METEORS_IN_BOSS:
		return

	level_time += delta
	meteor_timer -= delta
	if meteor_timer <= 0.0:
		meteor_timer = _meteor_interval() * randf_range(0.7, 1.3)
		_spawn_meteor(cam_c)
		# from level 5 on, sometimes two at once
		if level >= 5 and randf() < 0.15 + 0.04 * level:
			_spawn_meteor(cam_c)

func _spawn_meteor(cam_c: Vector2) -> void:
	# half of them are aimed near the player, half fall anywhere on screen
	var x: float
	if randf() < 0.5:
		x = player.position.x + randf_range(-300.0, 300.0)
	else:
		x = cam_c.x + randf_range(-650.0, 650.0)
	if boss_mode:
		x = clampf(x, 16.0, 36.0 * TS - 16.0)
	var y := cam_c.y - 324.0 - 60.0

	var speed := 170.0 + level * 28.0 + randf_range(0.0, 80.0)
	var vel := Vector2(randf_range(-70.0, 70.0), speed)
	var r := randf_range(10.0, 14.0 + level)

	var meteor := Area2D.new()
	meteor.position = Vector2(x, y)
	world.add_child(meteor)

	# fiery tail, pointing against the direction of travel
	var tail := Polygon2D.new()
	tail.polygon = PackedVector2Array([Vector2(0, -r * 0.8), Vector2(r * 4.0, 0), Vector2(0, r * 0.8)])
	tail.color = Color(1.0, 0.55, 0.15, 0.8)
	tail.rotation = (-vel).angle()
	meteor.add_child(tail)

	# rocky body: irregular polygon that spins
	var rock := Node2D.new()
	meteor.add_child(rock)
	var pts := PackedVector2Array()
	var sides := 9
	for i in sides:
		var a := i * TAU / sides
		pts.append(Vector2(cos(a), sin(a)) * r * randf_range(0.75, 1.1))
	var body := Polygon2D.new()
	body.polygon = pts
	body.color = Color(0.45, 0.32, 0.25)
	rock.add_child(body)
	var core := Polygon2D.new()
	var core_pts := PackedVector2Array()
	for p in pts:
		core_pts.append(p * 0.55)
	core.polygon = core_pts
	core.color = Color(0.7, 0.5, 0.35)
	rock.add_child(core)

	var cs := CollisionShape2D.new()
	var circle := CircleShape2D.new()
	circle.radius = r * 0.85
	cs.shape = circle
	meteor.add_child(cs)
	meteor.body_entered.connect(_on_meteor_hit.bind(meteor))

	meteors.append({"node": meteor, "rock": rock, "vel": vel, "spin": randf_range(-4.0, 4.0)})

func _on_meteor_hit(b: Node, meteor: Area2D) -> void:
	if not is_instance_valid(meteor):
		return
	if b == player:
		if player.has_method("hurt"):
			player.hurt()
	elif not (b is StaticBody2D):
		return    # ignore enemies, the boss, etc.
	call_deferred("_impact", meteor.position)
	meteor.queue_free()

# small spark burst where a meteor lands
func _impact(pos: Vector2) -> void:
	if not is_instance_valid(world):
		return
	for i in 7:
		var s := ColorRect.new()
		s.size = Vector2(PIXEL * 2, PIXEL * 2)
		s.color = Color(1.0, randf_range(0.5, 0.85), 0.2)
		s.position = pos
		world.add_child(s)
		var tw := s.create_tween().set_parallel(true)
		var dest := pos + Vector2(randf_range(-40.0, 40.0), randf_range(-50.0, -5.0))
		tw.tween_property(s, "position", dest, 0.4)
		tw.tween_property(s, "modulate:a", 0.0, 0.4)
		tw.chain().tween_callback(s.queue_free)

# ---------------------------------------------------------------- level generation
func _build_level(n: int) -> float:
	RenderingServer.set_default_clear_color(Color.from_hsv(fmod(0.55 + n * 0.09, 1.0), 0.35, 0.28))
	_level_stars(n)
	var rng := RandomNumberGenerator.new()
	rng.seed = 7919 * n
	var col := Color.from_hsv(fmod(0.08 + n * 0.09, 1.0), 0.45, 0.45)
	var length := 40 + n * 8
	var max_gap := 2 + int(n / 3.0)      # 2 tiles early, up to 5 by level 10 (dash helps!)
	var max_step := 1 if n < 3 else 2
	var h := 14
	var x := 0
	_ground(x, 8, h, col)
	spawn = Vector2(2 * TS, (h - 2) * TS)
	x += 8
	while x < length:
		x += rng.randi_range(2, max_gap)
		h = clampi(h + rng.randi_range(-max_step, max_step), 9, 17)
		var w := rng.randi_range(4, 8)
		_ground(x, w, h, col, _surface_for(n, x, w))
		if n >= 3 and w >= 6 and rng.randf() < 0.5:
			_spike(x + int(w / 2.0), h)
		elif n >= 2 and w >= 5 and rng.randf() < 0.25 + n * 0.04:
			_enemy(x, w, h, rng)
		x += w
	x += rng.randi_range(2, max_gap)
	_ground(x, 10, h, col)
	_goal(x + 6, h)
	return (x + 10) * TS

func _build_boss_arena() -> float:
	_space_background()
	var col := Color(0.22, 0.24, 0.38)
	_block(0, 17, 36, 3, col)
	_block(-1, -10, 1, 40, col)
	_block(36, -10, 1, 40, col)
	_block(5, 15, 4, 1, col, "bounce")
	_block(27, 15, 4, 1, col, "ice")
	spawn = Vector2(3 * TS, 15 * TS)
	return 36.0 * TS

# ---------------------------------------------------------------- stars / backgrounds
# Stars for normal levels: screen-fixed layer behind everything, so they
# stay visible while the camera scrolls along the level.
func _level_stars(n: int) -> void:
	var layer := CanvasLayer.new()
	layer.layer = -10
	# the viewport is PIXEL times smaller, so scale the layer to keep
	# the star positions laid out in 1152x648 screen space
	layer.scale = Vector2.ONE / float(PIXEL)
	world.add_child(layer)
	var rng := RandomNumberGenerator.new()
	rng.seed = 1000 + n
	_stars(layer, rng, 130, 648.0)

func _space_background() -> void:
	RenderingServer.set_default_clear_color(Color(0.02, 0.02, 0.08))
	var rng := RandomNumberGenerator.new()
	rng.seed = 42
	var bg := Node2D.new()
	bg.z_index = -10
	world.add_child(bg)
	# big planet + small moon
	_disc(bg, Vector2(820, 170), 110.0, Color(0.35, 0.2, 0.55))
	_disc(bg, Vector2(790, 150), 70.0, Color(0.45, 0.3, 0.68))
	_disc(bg, Vector2(260, 110), 34.0, Color(0.7, 0.7, 0.75))
	_stars(bg, rng, 150, 540.0)

# Shared star field (every third star twinkles).
# Star size is a whole number of low-res pixels so stars stay crisp.
func _stars(parent: Node, rng: RandomNumberGenerator, count: int, height: float) -> void:
	for i in count:
		var s := ColorRect.new()
		var sz := rng.randf_range(1.0, 2.0) * PIXEL
		s.size = Vector2(sz, sz)
		s.position = Vector2(rng.randf_range(0.0, 1152.0), rng.randf_range(0.0, height))
		s.color = Color(1, 1, 1, rng.randf_range(0.4, 1.0))
		parent.add_child(s)
		if i % 3 == 0:
			var tw := s.create_tween().set_loops()
			tw.tween_property(s, "modulate:a", 0.2, rng.randf_range(0.6, 1.6))
			tw.tween_property(s, "modulate:a", 1.0, rng.randf_range(0.6, 1.6))

func _disc(parent: Node, pos: Vector2, radius: float, col: Color) -> void:
	var pts := PackedVector2Array()
	for i in 48:
		pts.append(Vector2(cos(i * TAU / 48.0), sin(i * TAU / 48.0)) * radius)
	var p := Polygon2D.new()
	p.polygon = pts
	p.color = col
	p.position = pos
	parent.add_child(p)

# ---------------------------------------------------------------- builders
func _ground(tx: int, w: int, h: int, col: Color, surface: String = "") -> void:
	_block(tx, h, w, 24 - h, col, surface)

func _block(tx: int, ty: int, tw: int, th: int, col: Color, surface: String = "") -> void:
	var b := StaticBody2D.new()
	b.position = Vector2(tx * TS, ty * TS)
	if surface != "":
		b.set_meta("surface", surface)   # read by _update_surfaces()
	world.add_child(b)
	var sz := Vector2(tw * TS, th * TS)
	var cs := CollisionShape2D.new()
	var r := RectangleShape2D.new()
	r.size = sz
	cs.shape = r
	cs.position = sz / 2.0
	b.add_child(cs)
	# moon-rock visuals (the level color only gives the gray a faint tint)
	var rock := MoonRock.new()
	rock.sz = sz
	rock.surface = surface
	if surface == "ice":
		rock.base = Color(0.62, 0.8, 0.92)
	elif surface == "bounce":
		rock.base = Color(0.66, 0.46, 0.76)
	else:
		rock.base = Color(0.56, 0.56, 0.59).lerp(col, 0.15)
	rock.seed_val = (tx * 73856093) ^ (ty * 19349663) ^ (tw * 83492791)
	rock.px = float(PIXEL)
	b.add_child(rock)

# Draws one block of cratered moon rock. Drawn once and cached by the engine.
# surface "ice" adds frost and shine, "bounce" adds a springy top with arrows.
class MoonRock extends Node2D:
	var sz := Vector2.ZERO
	var base := Color(0.56, 0.56, 0.59)
	var seed_val := 1
	var px := 3.0
	var surface := ""

	func _draw() -> void:
		var rng := RandomNumberGenerator.new()
		rng.seed = seed_val
		var area_tiles := sz.x * sz.y / 1024.0
		var dust_h := 6.0

		# 1) base rock
		draw_rect(Rect2(Vector2.ZERO, sz), base)

		# 2) darker / lighter angular rock chunks
		var chunks := int(area_tiles * 1.5) + 2
		for i in chunks:
			var rad := rng.randf_range(8.0, 22.0)
			rad = minf(rad, minf(sz.x, sz.y) * 0.45)
			if rad < 5.0:
				continue
			var c := Vector2(
				rng.randf_range(rad, sz.x - rad),
				rng.randf_range(rad, sz.y - rad))
			var n := rng.randi_range(5, 7)
			var pts := PackedVector2Array()
			for k in n:
				var a := k * TAU / n + rng.randf_range(-0.3, 0.3)
				pts.append(c + Vector2(cos(a), sin(a)) * rad * rng.randf_range(0.6, 1.0))
			var shade := rng.randf_range(-0.1, 0.07)
			var chunk_col := base.lightened(shade) if shade > 0.0 else base.darkened(-shade)
			draw_colored_polygon(pts, chunk_col)

		# 3) pebble speckles, snapped to the pixel grid
		var specks := int(area_tiles * 7.0) + 6
		for i in specks:
			var sx := floorf(rng.randf_range(0.0, sz.x - px) / px) * px
			var sy := floorf(rng.randf_range(0.0, sz.y - px) / px) * px
			var sc := base.lightened(0.25) if rng.randf() < 0.5 else base.darkened(0.3)
			draw_rect(Rect2(sx, sy, px, px), sc)

		# 4) craters: lit rim underneath, dark pit on top
		var craters := int(sz.x / 90.0) + (1 if sz.y > 64.0 else 0)
		for i in craters:
			var rx := rng.randf_range(10.0, 24.0)
			var ry := rx * 0.4
			if sz.x < rx * 2.0 + 8.0 or sz.y < dust_h + ry * 2.0 + 10.0:
				continue
			var cx := rng.randf_range(rx + 4.0, sz.x - rx - 4.0)
			var cy_min := dust_h + ry + 5.0
			var cy_max := minf(sz.y - ry - 3.0, 90.0)
			if cy_max <= cy_min:
				continue
			var cy := rng.randf_range(cy_min, cy_max)
			draw_colored_polygon(_ellipse(Vector2(cx, cy + 2.0), rx, ry), base.lightened(0.3))
			draw_colored_polygon(_ellipse(Vector2(cx, cy), rx * 0.88, ry * 0.85), base.darkened(0.35))
			draw_colored_polygon(_ellipse(Vector2(cx, cy + ry * 0.2), rx * 0.55, ry * 0.5), base.darkened(0.2))

		# 5) pale dust layer on top with a ragged lower edge
		var dust := base.lightened(0.45)
		draw_rect(Rect2(0.0, 0.0, sz.x, dust_h), dust)
		draw_rect(Rect2(0.0, 0.0, sz.x, px), base.lightened(0.65))
		var step := px * 2.0
		var x := 0.0
		while x < sz.x:
			var w := minf(step, sz.x - x)
			var drop := float(rng.randi_range(0, 2)) * px
			if drop > 0.0:
				draw_rect(Rect2(x, dust_h, w, drop), dust.darkened(0.1))
			x += step

		# 6) special surfaces
		if surface == "ice":
			# glassy tint, bright frosty top edge, diagonal-ish shine marks
			draw_rect(Rect2(Vector2.ZERO, sz), Color(0.6, 0.85, 1.0, 0.18))
			draw_rect(Rect2(0.0, 0.0, sz.x, px * 2.0), Color(0.88, 0.98, 1.0))
			var shines := int(sz.x / 48.0) + 1
			for i in shines:
				var sx := floorf(rng.randf_range(0.0, maxf(sz.x - px * 6.0, 1.0)) / px) * px
				var sy := dust_h + px * float(rng.randi_range(1, 4))
				draw_rect(Rect2(sx, sy, px * 4.0, px), Color(1, 1, 1, 0.6))
				draw_rect(Rect2(sx + px * 4.0, sy + px, px * 2.0, px), Color(1, 1, 1, 0.4))
		elif surface == "bounce":
			# springy pink top and rows of white up-arrows
			draw_rect(Rect2(0.0, 0.0, sz.x, px * 2.0), Color(1.0, 0.45, 0.75))
			draw_rect(Rect2(0.0, px * 2.0, sz.x, px), Color(0.75, 0.25, 0.6))
			var ax := px * 4.0
			while ax + px * 6.0 < sz.x:
				var top := dust_h + px * 3.0
				if top + px * 3.0 < sz.y:
					var arrow := PackedVector2Array([
						Vector2(ax, top + px * 3.0),
						Vector2(ax + px * 3.0, top),
						Vector2(ax + px * 6.0, top + px * 3.0)])
					draw_polyline(arrow, Color(1, 1, 1, 0.85), px)
				ax += px * 10.0

	func _ellipse(c: Vector2, rx: float, ry: float) -> PackedVector2Array:
		var pts := PackedVector2Array()
		for i in 16:
			var a := i * TAU / 16.0
			pts.append(c + Vector2(cos(a) * rx, sin(a) * ry))
		return pts

func _spike(tx: int, ty: int) -> void:
	var a := Area2D.new()
	a.position = Vector2(tx * TS, (ty - 1) * TS)
	world.add_child(a)
	var p := Polygon2D.new()
	p.polygon = PackedVector2Array([Vector2(0, 32), Vector2(16, 2), Vector2(32, 32)])
	p.color = Color(0.9, 0.2, 0.25)
	a.add_child(p)
	var cs := CollisionShape2D.new()
	var r := RectangleShape2D.new()
	r.size = Vector2(18, 18)
	cs.shape = r
	cs.position = Vector2(16, 23)
	a.add_child(cs)
	a.body_entered.connect(_hurt_body)

func _hurt_body(b: Node) -> void:
	if b.has_method("hurt"):
		b.hurt()

func _enemy(tx: int, w: int, h: int, rng: RandomNumberGenerator) -> void:
	var enemy_script = load("res://enemy.gd")
	var e = enemy_script.new()
	e.left = tx * TS + 13.0
	e.right = (tx + w) * TS - 13.0
	e.position = Vector2(rng.randf_range(e.left, e.right), h * TS - 13.0)
	e.speed = 50.0 + level * 8.0
	world.add_child(e)

func _goal(tx: int, h: int) -> void:
	var a := Area2D.new()
	a.position = Vector2(tx * TS, (h - 3) * TS)
	world.add_child(a)
	var cs := CollisionShape2D.new()
	var r := RectangleShape2D.new()
	r.size = Vector2(TS, 3 * TS)
	cs.shape = r
	cs.position = r.size / 2.0
	a.add_child(cs)
	var pole := ColorRect.new()
	pole.size = Vector2(4, 96)
	pole.position = Vector2(14, 0)
	a.add_child(pole)
	var flag := ColorRect.new()
	flag.size = Vector2(24, 20)
	flag.position = Vector2(18, 0)
	flag.color = Color(1, 0.85, 0.2)
	a.add_child(flag)
	a.body_entered.connect(_on_goal)
