# Main.gd — Todo el juego en un solo script. Sencillo y funcional.
extends Node2D

# ── COLORES ──
const C_PURPLE  = Color(0.45, 0.0,  0.75, 1.0)
const C_BLACK   = Color(0.08, 0.05, 0.10, 1.0)
const C_YELLOW  = Color(0.95, 0.85, 0.05, 1.0)
const C_GREEN   = Color(0.10, 0.75, 0.20, 1.0)
const C_WHITE   = Color(0.95, 0.95, 0.95, 1.0)
const C_DARK    = Color(0.12, 0.08, 0.18, 1.0)

# ── CAMPO ──
const FIELD_LEFT   = 80.0
const FIELD_RIGHT  = 1200.0
const FIELD_TOP    = 100.0
const FIELD_BOT    = 620.0
const GOAL_H       = 140.0   # alto portería
const GOAL_DEPTH   = 40.0    # profundidad portería
const GOAL_Y_TOP   = 290.0
const GOAL_Y_BOT   = 430.0

# ── CONSTANTES JUEGO ──
const PLAYER_SPEED   = 260.0
const BALL_FRICTION  = 0.985
const KICK_FORCE     = 700.0
const BULLET_SPEED   = 620.0
const STUN_TIME      = 1.2
const SHOOT_COOLDOWN = 0.45
const MAX_SCORE      = 5
const MATCH_TIME     = 180.0
const POWERUP_INTERVAL = 9.0

# ── ESTADO ──
var score       = [0, 0]
var match_time  = MATCH_TIME
var active      = false
var goal_cooldown = 0.0
var powerup_timer = 0.0
var game_over   = false
var winner_team = -1

# ── JUGADORES ──
var players = []   # Array de dicts
# dict: {pos, vel, team, stun, stun_t, shoot_cd, weapon, ammo, frozen, freeze_t}

# ── PELOTA ──
var ball_pos = Vector2(640, 360)
var ball_vel = Vector2.ZERO

# ── BALAS ──
var bullets = []
# dict: {pos, vel, team, weapon, life}

# ── POWER-UPS en campo ──
var powerups_on_field = []
# dict: {pos, weapon}

# ── ARMAS ──
const WEAPONS = {
	"pistola":     {"emoji":"P", "color": C_WHITE,  "speed":620.0, "stun":1.2, "cd":0.45, "ammo":12, "effect":"normal"},
	"escopeta":    {"emoji":"E", "color": C_YELLOW, "speed":500.0, "stun":1.8, "cd":1.1,  "ammo":6,  "effect":"spread"},
	"cohete":      {"emoji":"C", "color":Color(1,0.3,0.1), "speed":380.0,"stun":2.5,"cd":2.0,"ammo":3,"effect":"explode"},
	"congelador":  {"emoji":"F", "color":Color(0.4,0.9,1), "speed":450.0,"stun":3.0,"cd":0.9,"ammo":8,"effect":"freeze"},
}
const WEAPON_ORDER = ["pistola","escopeta","cohete","congelador"]

# ── EFECTOS VISUALES ──
var explosions = []   # {pos, radius, life, max_life}
var goal_flash = 0.0
var goal_flash_team = 0

# ── UI ──
var score_label: Label
var timer_label: Label
var p1_hud: Label
var p2_hud: Label
var center_msg: Label
var controls_label: Label

func _ready():
	_build_ui()
	_start_match()

func _start_match():
	score = [0, 0]
	match_time = MATCH_TIME
	active = true
	game_over = false
	goal_cooldown = 0.0
	powerup_timer = 0.0
	bullets.clear()
	powerups_on_field.clear()
	explosions.clear()

	players = [
		_make_player(Vector2(280, 360), 0),
		_make_player(Vector2(1000, 360), 1),
	]
	ball_pos = Vector2(640, 360)
	ball_vel = Vector2.ZERO
	center_msg.text = ""

func _make_player(pos: Vector2, team: int) -> Dictionary:
	return {
		"pos": pos, "vel": Vector2.ZERO,
		"team": team, "stun": false, "stun_t": 0.0,
		"shoot_cd": 0.0, "weapon": "pistola", "ammo": 12,
		"frozen": false, "freeze_t": 0.0,
		"facing": Vector2(1 if team == 0 else -1, 0),
	}

# ────────────────────────────────────────────
func _physics_process(delta: float):
	if game_over:
		if Input.is_action_just_pressed("p1_shoot") or Input.is_action_just_pressed("p2_shoot"):
			_start_match()
		return

	if not active: return

	goal_cooldown -= delta
	if goal_cooldown > 0.0: return   # pausa breve tras gol

	match_time -= delta
	if match_time <= 0.0:
		match_time = 0.0
		_end_match()
		return

	powerup_timer += delta
	if powerup_timer >= POWERUP_INTERVAL:
		powerup_timer = 0.0
		_spawn_powerup()

	_update_players(delta)
	_update_ball(delta)
	_update_bullets(delta)
	_update_powerups()
	_check_goal()
	_update_explosions(delta)
	_update_hud()
	if goal_flash > 0.0:
		goal_flash -= delta

func _update_players(delta: float):
	for i in 2:
		var p = players[i]
		var prefix = "p%d_" % (i + 1)

		# Timers
		if p["stun_t"] > 0.0:
			p["stun_t"] -= delta
			if p["stun_t"] <= 0.0: p["stun"] = false
		if p["freeze_t"] > 0.0:
			p["freeze_t"] -= delta
			if p["freeze_t"] <= 0.0: p["frozen"] = false
		if p["shoot_cd"] > 0.0:
			p["shoot_cd"] -= delta

		if p["stun"] or p["frozen"]:
			p["vel"] = p["vel"].lerp(Vector2.ZERO, delta * 10.0)
			p["pos"] += p["vel"] * delta
			_clamp_player(p)
			continue

		# Movimiento
		var dir = Vector2.ZERO
		if Input.is_action_pressed(prefix + "up"):    dir.y -= 1
		if Input.is_action_pressed(prefix + "down"):  dir.y += 1
		if Input.is_action_pressed(prefix + "left"):  dir.x -= 1
		if Input.is_action_pressed(prefix + "right"): dir.x += 1
		if dir != Vector2.ZERO:
			dir = dir.normalized()
			p["facing"] = dir

		p["vel"] = dir * PLAYER_SPEED
		p["pos"] += p["vel"] * delta
		_clamp_player(p)

		# Disparo
		if Input.is_action_just_pressed(prefix + "shoot") and p["shoot_cd"] <= 0.0 and p["ammo"] > 0:
			_shoot(p)

		# Patada
		if Input.is_action_just_pressed(prefix + "kick"):
			_kick(p)

		# Auto-empuje suave al tocar pelota
		var dist = p["pos"].distance_to(ball_pos)
		if dist < 30.0:
			var push = (ball_pos - p["pos"]).normalized()
			ball_vel += push * 120.0

func _clamp_player(p: Dictionary):
	p["pos"].x = clamp(p["pos"].x, FIELD_LEFT + 15, FIELD_RIGHT - 15)
	p["pos"].y = clamp(p["pos"].y, FIELD_TOP + 15, FIELD_BOT - 15)

func _shoot(p: Dictionary):
	var w = WEAPONS[p["weapon"]]
	p["shoot_cd"] = w["cd"]
	p["ammo"] -= 1
	if p["ammo"] <= 0:
		p["weapon"] = "pistola"
		p["ammo"] = WEAPONS["pistola"]["ammo"]

	var dirs = []
	if w["effect"] == "spread":
		for a in [-18, -9, 0, 9, 18]:
			dirs.append(p["facing"].rotated(deg_to_rad(a)))
	else:
		dirs.append(p["facing"])

	for d in dirs:
		bullets.append({
			"pos": p["pos"] + d * 28.0,
			"vel": d * w["speed"],
			"team": p["team"],
			"weapon": p["weapon"],
			"life": 2.0,
		})

func _kick(p: Dictionary):
	if p["pos"].distance_to(ball_pos) < 60.0:
		var dir = (ball_pos - p["pos"]).normalized()
		ball_vel += dir * KICK_FORCE

func _update_ball(delta: float):
	ball_pos += ball_vel * delta
	ball_vel *= BALL_FRICTION

	# Tope superior e inferior
	if ball_pos.y < FIELD_TOP + 14:
		ball_pos.y = FIELD_TOP + 14
		ball_vel.y = abs(ball_vel.y) * 0.7
	if ball_pos.y > FIELD_BOT - 14:
		ball_pos.y = FIELD_BOT - 14
		ball_vel.y = -abs(ball_vel.y) * 0.7

	# Lateral izquierdo (fuera de portería)
	if ball_pos.x < FIELD_LEFT + 14:
		if ball_pos.y < GOAL_Y_TOP or ball_pos.y > GOAL_Y_BOT:
			ball_pos.x = FIELD_LEFT + 14
			ball_vel.x = abs(ball_vel.x) * 0.7
	# Lateral derecho
	if ball_pos.x > FIELD_RIGHT - 14:
		if ball_pos.y < GOAL_Y_TOP or ball_pos.y > GOAL_Y_BOT:
			ball_pos.x = FIELD_RIGHT - 14
			ball_vel.x = -abs(ball_vel.x) * 0.7

	# Límite profundidad portería
	if ball_pos.x < FIELD_LEFT - GOAL_DEPTH:
		ball_pos.x = FIELD_LEFT - GOAL_DEPTH
		ball_vel.x = abs(ball_vel.x) * 0.5
	if ball_pos.x > FIELD_RIGHT + GOAL_DEPTH:
		ball_pos.x = FIELD_RIGHT + GOAL_DEPTH
		ball_vel.x = -abs(ball_vel.x) * 0.5

	if ball_vel.length() > 950.0:
		ball_vel = ball_vel.normalized() * 950.0

func _update_bullets(delta: float):
	var to_remove = []
	for b in bullets:
		b["pos"] += b["vel"] * delta
		b["life"] -= delta

		# Fuera de pantalla
		if b["life"] <= 0 or b["pos"].x < 0 or b["pos"].x > 1280 or b["pos"].y < 0 or b["pos"].y > 720:
			to_remove.append(b)
			continue

		var w = WEAPONS[b["weapon"]]

		# Hit pelota
		if b["pos"].distance_to(ball_pos) < 20.0:
			if w["effect"] == "explode":
				ball_vel += (ball_pos - b["pos"]).normalized() * 900.0
				_add_explosion(b["pos"])
			else:
				ball_vel += b["vel"].normalized() * 430.0
			to_remove.append(b)
			continue

		# Hit jugador rival
		var hit = false
		for p in players:
			if p["team"] == b["team"]: continue
			if b["pos"].distance_to(p["pos"]) < 22.0:
				match w["effect"]:
					"freeze":
						p["frozen"] = true
						p["freeze_t"] = w["stun"]
					_:
						p["stun"] = true
						p["stun_t"] = w["stun"]
				if w["effect"] == "explode":
					_add_explosion(b["pos"])
				to_remove.append(b)
				hit = true
				break
		if hit: continue

		# Paredes
		if b["pos"].y < FIELD_TOP or b["pos"].y > FIELD_BOT:
			to_remove.append(b)

	for b in to_remove:
		bullets.erase(b)

func _update_powerups():
	var to_remove = []
	for pw in powerups_on_field:
		for p in players:
			if p["pos"].distance_to(pw["pos"]) < 28.0:
				p["weapon"] = pw["weapon"]
				p["ammo"] = WEAPONS[pw["weapon"]]["ammo"]
				to_remove.append(pw)
				break
	for pw in to_remove:
		powerups_on_field.erase(pw)

func _check_goal():
	# GOL equipo 1 (pelota entra portería izquierda)
	if ball_pos.x < FIELD_LEFT - 10 and ball_pos.y > GOAL_Y_TOP and ball_pos.y < GOAL_Y_BOT:
		_score_goal(1)
	# GOL equipo 0 (pelota entra portería derecha)
	if ball_pos.x > FIELD_RIGHT + 10 and ball_pos.y > GOAL_Y_TOP and ball_pos.y < GOAL_Y_BOT:
		_score_goal(0)

func _score_goal(team: int):
	score[team] += 1
	goal_flash = 1.5
	goal_flash_team = team
	goal_cooldown = 2.0
	center_msg.text = "⚽ GOL — %s!" % ("MORADO" if team == 0 else "VERDE")
	center_msg.modulate = C_PURPLE if team == 0 else C_GREEN

	# Reset posiciones
	ball_pos = Vector2(640, 360)
	ball_vel = Vector2.ZERO
	players[0]["pos"] = Vector2(280, 360)
	players[1]["pos"] = Vector2(1000, 360)
	for p in players:
		p["vel"] = Vector2.ZERO
		p["stun"] = false
		p["frozen"] = false

	await get_tree().create_timer(1.8).timeout
	center_msg.text = ""

	if score[team] >= MAX_SCORE:
		_end_match()

func _end_match():
	active = false
	game_over = true
	if score[0] > score[1]:
		winner_team = 0
		center_msg.text = "GANA MORADO! %d - %d\n\nF o K para reiniciar" % [score[0], score[1]]
		center_msg.modulate = C_PURPLE
	elif score[1] > score[0]:
		winner_team = 1
		center_msg.text = "GANA VERDE! %d - %d\n\nF o K para reiniciar" % [score[0], score[1]]
		center_msg.modulate = C_GREEN
	else:
		winner_team = -1
		center_msg.text = "EMPATE! %d - %d\n\nF o K para reiniciar" % [score[0], score[1]]
		center_msg.modulate = C_YELLOW

func _spawn_powerup():
	if powerups_on_field.size() >= 3: return
	var weapons = ["escopeta", "cohete", "congelador"]
	var w = weapons[randi() % weapons.size()]
	var pos = Vector2(
		randf_range(FIELD_LEFT + 80, FIELD_RIGHT - 80),
		randf_range(FIELD_TOP + 60, FIELD_BOT - 60)
	)
	powerups_on_field.append({"pos": pos, "weapon": w})

func _add_explosion(pos: Vector2):
	explosions.append({"pos": pos, "radius": 5.0, "life": 0.35, "max_life": 0.35})

func _update_explosions(delta: float):
	var to_remove = []
	for e in explosions:
		e["life"] -= delta
		e["radius"] = remap(e["life"], 0.0, e["max_life"], 80.0, 5.0)
		if e["life"] <= 0.0: to_remove.append(e)
	for e in to_remove: explosions.erase(e)

func _update_hud():
	score_label.text = "%d  —  %d" % [score[0], score[1]]
	var m = int(match_time) / 60
	var s = int(match_time) % 60
	timer_label.text = "%d:%02d" % [m, s]
	if match_time < 30: timer_label.modulate = C_YELLOW
	else: timer_label.modulate = C_WHITE

	var p1 = players[0]
	var p2 = players[1]
	var w1 = WEAPONS[p1["weapon"]]
	var w2 = WEAPONS[p2["weapon"]]
	p1_hud.text = "%s [%s] x%d%s" % [w1["emoji"], p1["weapon"], p1["ammo"], "  FROZEN" if p1["frozen"] else ("  STUN" if p1["stun"] else "")]
	p2_hud.text = "%s%s x%d [%s] %s" % ["FROZEN  " if p2["frozen"] else ("STUN  " if p2["stun"] else ""), p2["ammo"], w2["emoji"], p2["weapon"], ""]

# ── DIBUJO ────────────────────────────────────
func _draw():
	_draw_field()
	_draw_goals()
	_draw_powerups_on_field()
	_draw_ball()
	_draw_players()
	_draw_bullets()
	_draw_explosions()
	if goal_flash > 0.0:
		var col = C_PURPLE if goal_flash_team == 0 else C_GREEN
		col.a = goal_flash * 0.25
		draw_rect(Rect2(0, 0, 1280, 720), col)

func _draw_field():
	# Fondo negro
	draw_rect(Rect2(0, 0, 1280, 720), C_BLACK)
	# Césped interior (morado muy oscuro)
	draw_rect(Rect2(FIELD_LEFT, FIELD_TOP, FIELD_RIGHT - FIELD_LEFT, FIELD_BOT - FIELD_TOP), Color(0.12, 0.04, 0.20, 1.0))
	# Franjas de hierba
	for i in range(6):
		var stripe_x = FIELD_LEFT + i * ((FIELD_RIGHT - FIELD_LEFT) / 6.0)
		var stripe_w = (FIELD_RIGHT - FIELD_LEFT) / 6.0
		if i % 2 == 0:
			draw_rect(Rect2(stripe_x, FIELD_TOP, stripe_w, FIELD_BOT - FIELD_TOP), Color(0.10, 0.03, 0.18, 1.0))
	# Borde del campo (amarillo)
	draw_rect(Rect2(FIELD_LEFT, FIELD_TOP, FIELD_RIGHT - FIELD_LEFT, FIELD_BOT - FIELD_TOP), C_YELLOW, false, 3.0)
	# Línea central
	draw_line(Vector2(640, FIELD_TOP), Vector2(640, FIELD_BOT), C_YELLOW, 2.0)
	# Círculo central
	draw_arc(Vector2(640, 360), 70, 0, TAU, 40, C_YELLOW, 2.0)
	# Punto central
	draw_circle(Vector2(640, 360), 5, C_YELLOW)
	# Áreas de penalti
	draw_rect(Rect2(FIELD_LEFT, 240, 120, 240), C_YELLOW, false, 1.5)
	draw_rect(Rect2(FIELD_RIGHT - 120, 240, 120, 240), C_YELLOW, false, 1.5)

func _draw_goals():
	# Portería izquierda (morado)
	var gl_rect = Rect2(FIELD_LEFT - GOAL_DEPTH, GOAL_Y_TOP, GOAL_DEPTH, GOAL_H)
	draw_rect(gl_rect, Color(0.45, 0.0, 0.75, 0.3))
	draw_rect(gl_rect, C_PURPLE, false, 3.0)
	# Posts
	draw_line(Vector2(FIELD_LEFT, GOAL_Y_TOP), Vector2(FIELD_LEFT - GOAL_DEPTH, GOAL_Y_TOP), C_WHITE, 4.0)
	draw_line(Vector2(FIELD_LEFT, GOAL_Y_BOT), Vector2(FIELD_LEFT - GOAL_DEPTH, GOAL_Y_BOT), C_WHITE, 4.0)
	draw_line(Vector2(FIELD_LEFT - GOAL_DEPTH, GOAL_Y_TOP), Vector2(FIELD_LEFT - GOAL_DEPTH, GOAL_Y_BOT), C_WHITE, 4.0)

	# Portería derecha (verde)
	var gr_rect = Rect2(FIELD_RIGHT, GOAL_Y_TOP, GOAL_DEPTH, GOAL_H)
	draw_rect(gr_rect, Color(0.10, 0.75, 0.20, 0.3))
	draw_rect(gr_rect, C_GREEN, false, 3.0)
	draw_line(Vector2(FIELD_RIGHT, GOAL_Y_TOP), Vector2(FIELD_RIGHT + GOAL_DEPTH, GOAL_Y_TOP), C_WHITE, 4.0)
	draw_line(Vector2(FIELD_RIGHT, GOAL_Y_BOT), Vector2(FIELD_RIGHT + GOAL_DEPTH, GOAL_Y_BOT), C_WHITE, 4.0)
	draw_line(Vector2(FIELD_RIGHT + GOAL_DEPTH, GOAL_Y_TOP), Vector2(FIELD_RIGHT + GOAL_DEPTH, GOAL_Y_BOT), C_WHITE, 4.0)

func _draw_players():
	for i in 2:
		var p = players[i]
		var col = C_PURPLE if p["team"] == 0 else C_GREEN
		if p["frozen"]:  col = Color(0.4, 0.9, 1.0)
		elif p["stun"]:  col = C_YELLOW

		# Cuerpo
		draw_circle(p["pos"], 18, col)
		# Borde negro
		draw_arc(p["pos"], 18, 0, TAU, 20, C_BLACK, 2.5)
		# Número
		# Indicador de dirección
		var tip = p["pos"] + p["facing"] * 24.0
		draw_line(p["pos"], tip, C_WHITE, 2.5)
		draw_circle(tip, 4, C_WHITE)

		# Arma encima
		if p["weapon"] != "pistola":
			var w = WEAPONS[p["weapon"]]
			var wp = p["pos"] + Vector2(0, -28)
			draw_circle(wp, 8, w["color"])
			draw_arc(wp, 8, 0, TAU, 12, C_BLACK, 1.5)

func _draw_ball():
	# Sombra
	draw_circle(ball_pos + Vector2(3, 3), 14, Color(0, 0, 0, 0.35))
	# Pelota blanca
	draw_circle(ball_pos, 14, C_WHITE)
	# Manchas negras (estilo fútbol)
	draw_circle(ball_pos, 5, C_BLACK)
	draw_circle(ball_pos + Vector2(8, 5), 3, C_BLACK)
	draw_circle(ball_pos + Vector2(-8, 5), 3, C_BLACK)
	draw_circle(ball_pos + Vector2(0, -9), 3, C_BLACK)
	# Borde
	draw_arc(ball_pos, 14, 0, TAU, 20, C_BLACK, 1.5)

func _draw_bullets():
	for b in bullets:
		var w = WEAPONS[b["weapon"]]
		var col = w["color"]
		draw_circle(b["pos"], 6, col)
		# Trail
		var trail_end = b["pos"] - b["vel"].normalized() * 18.0
		var tc = col; tc.a = 0.4
		draw_line(b["pos"], trail_end, tc, 3.0)

func _draw_powerups_on_field():
	for pw in powerups_on_field:
		var w = WEAPONS[pw["weapon"]]
		# Glow
		var gc = w["color"]; gc.a = 0.25
		draw_circle(pw["pos"], 22, gc)
		# Cuerpo
		draw_circle(pw["pos"], 14, w["color"])
		draw_arc(pw["pos"], 14, 0, TAU, 16, C_BLACK, 2.0)

func _draw_explosions():
	for e in explosions:
		var t = e["life"] / e["max_life"]
		var col = Color(1.0, 0.5, 0.1, t * 0.8)
		draw_circle(e["pos"], e["radius"], col)
		var col2 = Color(1.0, 0.9, 0.3, t * 0.5)
		draw_circle(e["pos"], e["radius"] * 0.5, col2)

# ── UI ─────────────────────────────────────────
func _build_ui():
	var canvas = CanvasLayer.new()
	add_child(canvas)

	# Barra superior negra
	var top_bar = ColorRect.new()
	top_bar.color = Color(0.06, 0.02, 0.12, 0.92)
	top_bar.set_anchors_preset(Control.PRESET_TOP_WIDE)
	top_bar.custom_minimum_size = Vector2(0, 60)
	top_bar.size = Vector2(1280, 60)
	canvas.add_child(top_bar)

	# P1 label (morado, izquierda)
	var p1_tag = Label.new()
	p1_tag.text = "MORADO"
	p1_tag.position = Vector2(16, 8)
	p1_tag.add_theme_font_size_override("font_size", 13)
	p1_tag.add_theme_color_override("font_color", C_PURPLE)
	canvas.add_child(p1_tag)

	p1_hud = Label.new()
	p1_hud.text = "P [pistola] x12"
	p1_hud.position = Vector2(16, 28)
	p1_hud.add_theme_font_size_override("font_size", 14)
	p1_hud.add_theme_color_override("font_color", C_WHITE)
	canvas.add_child(p1_hud)

	# Marcador (centro)
	score_label = Label.new()
	score_label.text = "0  —  0"
	score_label.add_theme_font_size_override("font_size", 36)
	score_label.add_theme_color_override("font_color", C_YELLOW)
	score_label.set_anchors_preset(Control.PRESET_TOP_WIDE)
	score_label.horizontal_alignment = HORIZONTAL_ALIGNMENT_CENTER
	score_label.position = Vector2(0, 4)
	canvas.add_child(score_label)

	timer_label = Label.new()
	timer_label.text = "3:00"
	timer_label.add_theme_font_size_override("font_size", 16)
	timer_label.add_theme_color_override("font_color", C_WHITE)
	timer_label.set_anchors_preset(Control.PRESET_TOP_WIDE)
	timer_label.horizontal_alignment = HORIZONTAL_ALIGNMENT_CENTER
	timer_label.position = Vector2(0, 42)
	canvas.add_child(timer_label)

	# P2 label (verde, derecha)
	var p2_tag = Label.new()
	p2_tag.text = "VERDE"
	p2_tag.set_anchors_preset(Control.PRESET_TOP_RIGHT)
	p2_tag.position = Vector2(-130, 8)
	p2_tag.add_theme_font_size_override("font_size", 13)
	p2_tag.add_theme_color_override("font_color", C_GREEN)
	canvas.add_child(p2_tag)

	p2_hud = Label.new()
	p2_hud.text = "x12 [pistola] P"
	p2_hud.set_anchors_preset(Control.PRESET_TOP_RIGHT)
	p2_hud.position = Vector2(-220, 28)
	p2_hud.add_theme_font_size_override("font_size", 14)
	p2_hud.add_theme_color_override("font_color", C_WHITE)
	canvas.add_child(p2_hud)

	# Mensaje central (goles, fin de partido)
	center_msg = Label.new()
	center_msg.text = ""
	center_msg.add_theme_font_size_override("font_size", 38)
	center_msg.add_theme_color_override("font_color", C_YELLOW)
	center_msg.set_anchors_preset(Control.PRESET_CENTER)
	center_msg.horizontal_alignment = HORIZONTAL_ALIGNMENT_CENTER
	center_msg.position = Vector2(-400, -50)
	center_msg.size = Vector2(800, 120)
	center_msg.autowrap_mode = TextServer.AUTOWRAP_WORD
	canvas.add_child(center_msg)

	# Controles abajo
	controls_label = Label.new()
	controls_label.text = "MORADO: WASD mover · F disparar · G patear     |     VERDE: Flechas mover · K disparar · L patear"
	controls_label.add_theme_font_size_override("font_size", 12)
	controls_label.add_theme_color_override("font_color", Color(0.6, 0.6, 0.6, 0.7))
	controls_label.set_anchors_preset(Control.PRESET_BOTTOM_WIDE)
	controls_label.horizontal_alignment = HORIZONTAL_ALIGNMENT_CENTER
	controls_label.position = Vector2(0, -22)
	canvas.add_child(controls_label)

	# Leyenda power-ups
	var pw_legend = Label.new()
	pw_legend.text = "Power-ups:  E=Escopeta  C=Cohete  F=Congelador"
	pw_legend.add_theme_font_size_override("font_size", 12)
	pw_legend.add_theme_color_override("font_color", Color(0.7, 0.7, 0.4, 0.65))
	pw_legend.set_anchors_preset(Control.PRESET_BOTTOM_WIDE)
	pw_legend.horizontal_alignment = HORIZONTAL_ALIGNMENT_CENTER
	pw_legend.position = Vector2(0, -38)
	canvas.add_child(pw_legend)

	# Forzar redibujado en cada frame
	set_process(true)

func _process(_delta: float):
	queue_redraw()
