# space-game
extends Node3D

class RadarWidget extends Control:
	var blips: Array[Vector2] = []

	func set_blips(new_blips: Array[Vector2]) -> void:
		blips = new_blips
		queue_redraw()

	func _draw() -> void:
		var center := size * 0.5
		var radius: float = minf(size.x, size.y) * 0.44
		var cyan := Color(0.15, 0.85, 1.0, 0.8)

		draw_circle(center, radius, Color(0.02, 0.08, 0.12, 0.88))
		draw_arc(center, radius, 0.0, TAU, 64, cyan, 2.0)
		draw_arc(center, radius * 0.66, 0.0, TAU, 64, Color(0.15, 0.65, 0.78, 0.45), 1.0)
		draw_arc(center, radius * 0.33, 0.0, TAU, 64, Color(0.15, 0.65, 0.78, 0.35), 1.0)
		draw_line(Vector2(center.x - radius, center.y), Vector2(center.x + radius, center.y), Color(0.15, 0.65, 0.78, 0.35), 1.0)
		draw_line(Vector2(center.x, center.y - radius), Vector2(center.x, center.y + radius), Color(0.15, 0.65, 0.78, 0.35), 1.0)

		for blip in blips:
			var point: Vector2 = center + Vector2(blip.x, -blip.y) * radius
			draw_circle(point, 4.0, Color(0.1, 1.0, 0.95, 0.95))
			draw_circle(point, 7.0, Color(0.1, 1.0, 0.95, 0.25))

		draw_circle(center, 5.0, Color(1.0, 1.0, 1.0, 0.95))
		draw_circle(center, 9.0, Color(0.2, 0.8, 1.0, 0.3))


# ------------------------------------------------------------------ world
const WORLD_WIDTH := 400.0
const WORLD_DEPTH := 250.0
const TERRAIN_RESOLUTION_X := 400
const TERRAIN_RESOLUTION_Z := 250
const WORLD_EDGE_MARGIN := 1.5
const SPAWN_POSITION := Vector3(-150.0, 5.0, 0.0)
const DUNE_PATCH_ORIGIN := Vector2(-100.0, -50.0)
const DUNE_PATCH_WIDTH := 200.0
const DUNE_PATCH_DEPTH := 100.0
const DUNE_HEIGHT_SAMPLE_SPACING := 1.0

# ------------------------------------------------------------------ graphics
# 0 = performance, 1 = balanced, 2 = high. Lower this if the game runs slowly.
const GRAPHICS_QUALITY := 1
# Set to false to fall back to a plain StandardMaterial3D if the custom shader gives trouble.
const USE_CUSTOM_TERRAIN_SHADER := true
const TERRAIN_BRIGHTNESS := 1.2
const PHYSICS_TICKS_PER_SECOND := 120
const UI_UPDATE_INTERVAL := 1.0 / 30.0

# ------------------------------------------------------------------ movement
const WALK_SPEED := 2.0
const SPRINT_SPEED := 3.6
const JUMP_FORCE := 7.5
const GRAVITY := 18.0
const JETPACK_THRUST := 26.0
const JETPACK_DURATION := 5.0
const JETPACK_RECHARGE_TIME := 15.0
const MOUSE_SENSITIVITY := 0.003
const ACCELERATION := 8.0
const BRAKING := 12.0
const AIR_CONTROL := 0.55
const AIR_DRAG := 2.5
const COYOTE_TIME := 0.12
const JUMP_BUFFER_TIME := 0.12

# Slopes up to this angle are walkable (and get slower when going uphill).
# Anything steeper can be climbed by pushing into it.
const FLOOR_MAX_ANGLE_DEG := 55.0
const UPHILL_MIN_SPEED_FACTOR := 0.6

# Animation cycle rates are the original ones scaled to the new walk/sprint speeds.
const WALK_ANIM_RATE := 2.0 * (WALK_SPEED / 3.2)
const RUN_ANIM_RATE := 1.5 * (SPRINT_SPEED / 6.0)

# ------------------------------------------------------------------ climbing
const CLIMB_SPEED := 3.2
const CLIMB_STICK_SPEED := 2.0
const CLIMB_ENTRY_PUSH_DOT := 0.55
const CLIMB_ENTRY_TIME := 0.15
const CLIMB_LOST_GRACE := 0.25
const CLIMB_COOLDOWN := 0.25
const CLIMB_JUMP_AWAY := 4.5

# ------------------------------------------------------------------ character visuals
# If the astronaut's soles are slightly above/below the ground in the rest pose, tune this (metres).
const MODEL_FOOT_OFFSET := 0.0
const MAX_FOOT_DROP := 0.45
const GROUND_TILT_AMOUNT := 0.15

const TERRAIN_DIFFUSE := "res://assets/codex_upgrade/rocky_terrain_diff_2k.jpg"
const TERRAIN_NORMAL := "res://assets/codex_upgrade/rocky_terrain_nor_dx_2k.jpg"
const TERRAIN_ROUGHNESS := "res://assets/codex_upgrade/rocky_terrain_rough_2k.jpg"
const MARS_MAP := "res://assets/codex_upgrade/mars_jpl.jpg"
const MARS_GALE_HEIGHT_MAP := "res://assets/codex_upgrade/mars_gale_10ha_height.bin"
const MARS_DUNE_HEIGHT_MAP := "res://assets/codex_upgrade/mars_bagnold_2ha_height.bin"
const MARS_DUNE_ALBEDO := "res://assets/codex_upgrade/mars_bagnold_2ha_albedo.png"
const ASTRONAUT_MODEL := "res://assets/codex_upgrade/astronaut_rigged.glb"
const TON_618_MODEL := "res://assets/codex_upgrade/black-hole-ton618.glb"

# Input actions. Only created if they don't exist, so anything you set up in
# Project Settings > Input Map (including gamepad buttons) is kept.
const DEFAULT_KEY_BINDINGS := {
	"move_forward": KEY_W,
	"move_back": KEY_S,
	"move_left": KEY_A,
	"move_right": KEY_D,
	"sprint": KEY_SHIFT,
	"jump": KEY_SPACE,
	"jetpack": KEY_E,
}

# Triplanar terrain shader: textures are projected from the world axes, so steep
# slopes and cliffs don't stretch. Two texture scales are blended on flat ground
# to hide tiling, and steep faces get a darker, cooler rock tint.
const TERRAIN_SHADER_CODE := """
shader_type spatial;
render_mode diffuse_burley, specular_schlick_ggx;

uniform sampler2D albedo_tex : source_color, filter_linear_mipmap_anisotropic, repeat_enable;
uniform sampler2D normal_tex : hint_normal, filter_linear_mipmap_anisotropic, repeat_enable;
uniform sampler2D rough_tex : hint_default_white, filter_linear_mipmap_anisotropic, repeat_enable;
uniform sampler2D mars_macro_tex : source_color, filter_linear_mipmap_anisotropic, repeat_enable;
uniform sampler2D dune_albedo_tex : source_color, filter_linear_mipmap_anisotropic, repeat_disable;
uniform vec2 dune_origin = vec2(-100.0, -50.0);
uniform vec2 dune_size = vec2(200.0, 100.0);
uniform vec4 tint : source_color = vec4(1.0);
uniform float tile_size = 6.0;
uniform float far_tile_scale = 0.17;
uniform float mars_macro_scale = 0.008;
uniform float albedo_gain = 1.4;
uniform float normal_strength = 0.75;
uniform float blend_sharpness = 6.0;
uniform bool flip_green = false;
uniform vec3 cliff_tint : source_color = vec3(0.62, 0.60, 0.72);

varying vec3 w_pos;
varying vec3 w_normal;

vec3 unpack_normal(vec4 tex) {
	vec3 n = tex.xyz * 2.0 - 1.0;
	if (flip_green) {
		n.y = -n.y;
	}
	n.xy *= normal_strength;
	return n;
}

void vertex() {
	w_pos = (MODEL_MATRIX * vec4(VERTEX, 1.0)).xyz;
	w_normal = normalize(MODEL_NORMAL_MATRIX * NORMAL);
}

void fragment() {
	vec3 wn = normalize(w_normal);
	vec3 w = pow(abs(wn), vec3(blend_sharpness));
	w /= (w.x + w.y + w.z + 0.0001);

	float s_near = 1.0 / tile_size;
	float s_far = s_near * far_tile_scale;

	vec2 uv_x = w_pos.zy * s_near;
	vec2 uv_y = w_pos.xz * s_near;
	vec2 uv_z = w_pos.xy * s_near;
	vec2 uv_y_far = w_pos.xz * s_far + vec2(0.37, 0.61);

	vec3 col_x = texture(albedo_tex, uv_x).rgb;
	vec3 col_z = texture(albedo_tex, uv_z).rgb;
	vec3 col_y = mix(texture(albedo_tex, uv_y).rgb, texture(albedo_tex, uv_y_far).rgb, 0.45);
	vec3 tex_col = col_x * w.x + col_y * w.y + col_z * w.z;
	vec3 mars_macro = texture(mars_macro_tex, w_pos.xz * mars_macro_scale + vec2(0.31, 0.17)).rgb;
	tex_col *= mix(vec3(1.0), mars_macro * 1.35, 0.38);

	float cliff = 1.0 - smoothstep(0.55, 0.85, wn.y);
	float dune_mask = clamp(1.0 - COLOR.a, 0.0, 1.0);
	vec3 base = tex_col * COLOR.rgb * tint.rgb * albedo_gain;
	vec3 rock_albedo = mix(base, base * cliff_tint, cliff);
	vec2 dune_uv = clamp((w_pos.xz - dune_origin) / dune_size, vec2(0.0), vec2(1.0));
	vec3 dune_albedo = texture(dune_albedo_tex, dune_uv).rgb;
	ALBEDO = mix(rock_albedo, dune_albedo, dune_mask);

	// Triplanar normal mapping (whiteout blend), done in world space.
	vec3 n_x = unpack_normal(texture(normal_tex, uv_x));
	vec3 n_z = unpack_normal(texture(normal_tex, uv_z));
	vec3 n_y = mix(unpack_normal(texture(normal_tex, uv_y)), unpack_normal(texture(normal_tex, uv_y_far)), 0.4);
	n_x = vec3(n_x.xy + wn.zy, abs(n_x.z) * wn.x);
	n_y = vec3(n_y.xy + wn.xz, abs(n_y.z) * wn.y);
	n_z = vec3(n_z.xy + wn.xy, abs(n_z.z) * wn.z);
	vec3 world_n = normalize(n_x.zyx * w.x + n_y.xzy * w.y + n_z.xyz * w.z);
	vec3 terrain_n = normalize(mix(world_n, wn, dune_mask * 0.85));
	NORMAL = normalize((VIEW_MATRIX * vec4(terrain_n, 0.0)).xyz);

	ROUGHNESS = mix(clamp(texture(rough_tex, uv_y).r, 0.6, 1.0), 0.96, dune_mask);
	METALLIC = mix(0.04, 0.0, dune_mask);
	SPECULAR = 0.35;
}
"""

# ------------------------------------------------------------------ state
var player: CharacterBody3D
var astronaut_model: Node3D
var astronaut_animation: AnimationPlayer
var skeleton: Skeleton3D
var camera_pivot: Node3D
var player_camera: Camera3D
var terrain_noise := FastNoiseLite.new()
var mars_height_data := PackedFloat32Array()
var mars_dune_data := PackedFloat32Array()

var terrain_shader: Shader
var tex_diffuse: Texture2D
var tex_normal: Texture2D
var tex_roughness: Texture2D
var tex_dune_albedo: Texture2D

var effect_time := 0.0
var ui_timer := 0.0
var fps_timer := 0.0

var starfield: MultiMeshInstance3D
var space_sky_root: Node3D
var accretion_disk_node: Node3D
var red_giant_material: StandardMaterial3D
var red_giant_glow: Sprite3D
var tidal_stream: MeshInstance3D
var galaxy_textures: Array[Texture2D] = []

var message_timer := 0.0

var jetpack_fuel := JETPACK_DURATION
var jetpack_recharge_timer := 0.0

# Movement helpers
var coyote_timer := 0.0
var jump_buffer_timer := 0.0

# Climbing
var climbing := false
var wall_normal := Vector3.ZERO
var wall_contact_time := 0.0
var climb_lost_timer := 0.0
var climb_cooldown := 0.0
var climb_move_amount := 0.0
var climb_hint_shown := false

# Character visuals
var model_yaw := 0.0
var model_quat := Quaternion.IDENTITY
var ground_normal := Vector3.UP
var model_height_offset := 0.0
var rest_pose_applied := false
var anim_idle := ""
var anim_walk := ""
var anim_run := ""
var anim_climb := ""
var anim_air := ""

# HUD
var title_label: Label
var jetpack_bar: ProgressBar
var jetpack_value: Label
var message_label: Label

var game_started := false


func _ready() -> void:
	randomize()
	Engine.physics_ticks_per_second = PHYSICS_TICKS_PER_SECOND
	setup_input_map()
	setup_noise()
	load_mars_height_map()
	create_environment()
	create_terrain()
	create_player()
	create_space_sky()
	create_world_objects()
	create_hud()
	Input.mouse_mode = Input.MOUSE_MODE_CAPTURED
	game_started = true


func setup_input_map() -> void:
	for action in DEFAULT_KEY_BINDINGS:
		if InputMap.has_action(action):
			continue
		InputMap.add_action(action)
		var key_event := InputEventKey.new()
		key_event.physical_keycode = DEFAULT_KEY_BINDINGS[action]
		InputMap.action_add_event(action, key_event)


func setup_noise() -> void:
	terrain_noise.seed = randi()
	terrain_noise.noise_type = FastNoiseLite.TYPE_SIMPLEX
	terrain_noise.frequency = 0.035
	terrain_noise.fractal_octaves = 5
	terrain_noise.fractal_gain = 0.5
	terrain_noise.fractal_lacunarity = 2.1


func load_mars_height_map() -> void:
	var file := FileAccess.open(MARS_GALE_HEIGHT_MAP, FileAccess.READ)
	if file == null:
		push_warning("Gale Crater height data not found; using procedural fallback terrain.")
	else:
		var expected_samples := (TERRAIN_RESOLUTION_X + 1) * (TERRAIN_RESOLUTION_Z + 1)
		if file.get_length() == expected_samples * 4:
			mars_height_data.resize(expected_samples)
			for i in range(expected_samples):
				mars_height_data[i] = file.get_float()
		else:
			push_warning("Gale Crater height data has an unexpected size; using procedural fallback terrain.")

	var dune_file := FileAccess.open(MARS_DUNE_HEIGHT_MAP, FileAccess.READ)
	if dune_file == null:
		push_warning("Bagnold dune data not found; the 2 hectare sand patch will be omitted.")
		return
	var dune_samples := 201 * 101
	if dune_file.get_length() != dune_samples * 4:
		push_warning("Bagnold dune data has an unexpected size; the sand patch will be omitted.")
		return
	mars_dune_data.resize(dune_samples)
	for i in range(dune_samples):
		mars_dune_data[i] = dune_file.get_float()
	tex_dune_albedo = load(MARS_DUNE_ALBEDO) as Texture2D


func create_environment() -> void:
	var world := WorldEnvironment.new()
	var env := Environment.new()
	env.background_mode = Environment.BG_COLOR
	env.background_color = Color(0.0015, 0.002, 0.008)
	env.ambient_light_source = Environment.AMBIENT_SOURCE_COLOR
	env.ambient_light_color = Color(0.25, 0.29, 0.40)
	env.ambient_light_energy = 0.52

	# Look: ACES tonemapping, slight contrast/saturation lift, bloom on bright scene details.
	env.tonemap_mode = Environment.TONE_MAPPER_ACES
	env.adjustment_enabled = true
	env.adjustment_contrast = 1.08
	env.adjustment_saturation = 1.12

	# A very thin haze gives the terrain depth. Sky, stars and planets ignore it.
	env.fog_enabled = true
	env.fog_light_color = Color(0.10, 0.12, 0.22)
	env.fog_density = 0.0016
	env.fog_sky_affect = 0.0

	var sun := DirectionalLight3D.new()
	sun.rotation_degrees = Vector3(-42.0, -28.0, 0.0)
	sun.light_color = Color(0.78, 0.82, 1.0)
	sun.light_energy = 1.35
	sun.shadow_enabled = true
	sun.shadow_normal_bias = 1.5
	sun.shadow_blur = 1.4

	var viewport := get_viewport()
	if GRAPHICS_QUALITY <= 0:
		env.glow_enabled = false
		env.ssao_enabled = false
		sun.directional_shadow_mode = DirectionalLight3D.SHADOW_PARALLEL_2_SPLITS
		sun.directional_shadow_max_distance = 80.0
		viewport.msaa_3d = Viewport.MSAA_DISABLED
		viewport.screen_space_aa = Viewport.SCREEN_SPACE_AA_FXAA
	elif GRAPHICS_QUALITY == 1:
		env.glow_enabled = true
		env.glow_intensity = 0.7
		env.glow_bloom = 0.04
		env.glow_blend_mode = Environment.GLOW_BLEND_MODE_SOFTLIGHT
		env.ssao_enabled = true
		env.ssao_radius = 1.6
		env.ssao_intensity = 2.0
		sun.directional_shadow_mode = DirectionalLight3D.SHADOW_PARALLEL_4_SPLITS
		sun.directional_shadow_max_distance = 110.0
		viewport.msaa_3d = Viewport.MSAA_2X
	else:
		env.glow_enabled = true
		env.glow_intensity = 0.8
		env.glow_bloom = 0.05
		env.glow_blend_mode = Environment.GLOW_BLEND_MODE_SOFTLIGHT
		env.ssao_enabled = true
		env.ssao_radius = 1.8
		env.ssao_intensity = 2.4
		env.ssil_enabled = true
		sun.directional_shadow_mode = DirectionalLight3D.SHADOW_PARALLEL_4_SPLITS
		sun.directional_shadow_max_distance = 150.0
		viewport.msaa_3d = Viewport.MSAA_4X

	world.environment = env
	add_child(world)
	add_child(sun)


func get_terrain_height(x: float, z: float) -> float:
	var terrain_height: float
	if mars_height_data.size() == (TERRAIN_RESOLUTION_X + 1) * (TERRAIN_RESOLUTION_Z + 1):
		var u := clampf(x + WORLD_WIDTH * 0.5, 0.0, WORLD_WIDTH)
		var v := clampf(z + WORLD_DEPTH * 0.5, 0.0, WORLD_DEPTH)
		terrain_height = sample_height_bilinear(mars_height_data, TERRAIN_RESOLUTION_X + 1, TERRAIN_RESOLUTION_Z + 1, u, v)
	else:
		var broad := terrain_noise.get_noise_2d(x, z) * 8.0
		var detail := terrain_noise.get_noise_2d(x * 2.8, z * 2.8) * 1.8
		var ridges := absf(terrain_noise.get_noise_2d(x * 0.55 + 100.0, z * 0.55)) * 2.0
		terrain_height = broad + detail + ridges - 0.7

	# Replace exactly 200 m x 100 m of the Gale surface with real HiRISE dune relief.
	if mars_dune_data.size() == 201 * 101:
		var local_x := x - DUNE_PATCH_ORIGIN.x
		var local_z := z - DUNE_PATCH_ORIGIN.y
		if local_x >= 0.0 and local_x <= DUNE_PATCH_WIDTH and local_z >= 0.0 and local_z <= DUNE_PATCH_DEPTH:
			var dune_u := local_x / DUNE_HEIGHT_SAMPLE_SPACING
			var dune_v := local_z / DUNE_HEIGHT_SAMPLE_SPACING
			var dune_relief := sample_height_bilinear(mars_dune_data, 201, 101, dune_u, dune_v)
			var edge_distance := minf(minf(local_x, DUNE_PATCH_WIDTH - local_x), minf(local_z, DUNE_PATCH_DEPTH - local_z))
			var edge_blend := clampf(edge_distance / 8.0, 0.0, 1.0)
			edge_blend = edge_blend * edge_blend * (3.0 - 2.0 * edge_blend)
			terrain_height += dune_relief * edge_blend
	return terrain_height


func sample_height_bilinear(data: PackedFloat32Array, grid_width: int, grid_depth: int, u: float, v: float) -> float:
	var x0 := clampi(floori(u), 0, grid_width - 1)
	var z0 := clampi(floori(v), 0, grid_depth - 1)
	var x1 := mini(x0 + 1, grid_width - 1)
	var z1 := mini(z0 + 1, grid_depth - 1)
	var fx := clampf(u - float(x0), 0.0, 1.0)
	var fz := clampf(v - float(z0), 0.0, 1.0)
	var top := lerpf(data[z0 * grid_width + x0], data[z0 * grid_width + x1], fx)
	var bottom := lerpf(data[z1 * grid_width + x0], data[z1 * grid_width + x1], fx)
	return lerpf(top, bottom, fz)


func get_terrain_color(x: float, z: float, height: float) -> Color:
	var color_noise := terrain_noise.get_noise_2d(x * 0.18 + 500.0, z * 0.18)
	var rock_noise := terrain_noise.get_noise_2d(x * 0.45, z * 0.45 + 300.0)
	var dune_noise := terrain_noise.get_noise_2d(x * 0.08 - 180.0, z * 0.08 + 240.0)
	var base_color: Color
	if height > 6.0:
		base_color = Color(0.62, 0.31, 0.21).lerp(
			Color(0.82, 0.56, 0.39),
			clampf((height - 6.0) / 10.0, 0.0, 0.8)
		)
	elif rock_noise > 0.35:
		base_color = Color(0.43, 0.22, 0.17)
	elif dune_noise > 0.15:
		base_color = Color(0.72, 0.40, 0.28).lerp(Color(0.84, 0.59, 0.42), clampf(dune_noise, 0.0, 1.0) * 0.45)
	elif color_noise > 0.35:
		base_color = Color(0.68, 0.34, 0.24)
	else:
		base_color = Color(0.61, 0.30, 0.22)

	var local_x := x - DUNE_PATCH_ORIGIN.x
	var local_z := z - DUNE_PATCH_ORIGIN.y
	if local_x >= 0.0 and local_x <= DUNE_PATCH_WIDTH and local_z >= 0.0 and local_z <= DUNE_PATCH_DEPTH:
		var edge_distance := minf(minf(local_x, DUNE_PATCH_WIDTH - local_x), minf(local_z, DUNE_PATCH_DEPTH - local_z))
		var dune_blend := clampf(edge_distance / 8.0, 0.0, 1.0)
		dune_blend = dune_blend * dune_blend * (3.0 - 2.0 * dune_blend)
		var sand_edge_tint := base_color.lerp(Color(0.50, 0.31, 0.23), dune_blend)
		return Color(sand_edge_tint.r, sand_edge_tint.g, sand_edge_tint.b, 1.0 - dune_blend)
	return base_color


func make_terrain_material(tint: Color = Color.WHITE) -> Material:
	if tex_diffuse == null:
		tex_diffuse = load(TERRAIN_DIFFUSE) as Texture2D
		tex_normal = load(TERRAIN_NORMAL) as Texture2D
		tex_roughness = load(TERRAIN_ROUGHNESS) as Texture2D

	if USE_CUSTOM_TERRAIN_SHADER:
		if terrain_shader == null:
			terrain_shader = Shader.new()
			terrain_shader.code = TERRAIN_SHADER_CODE
		var shader_material := ShaderMaterial.new()
		shader_material.shader = terrain_shader
		shader_material.set_shader_parameter("albedo_tex", tex_diffuse)
		shader_material.set_shader_parameter("normal_tex", tex_normal)
		shader_material.set_shader_parameter("rough_tex", tex_roughness)
		shader_material.set_shader_parameter("mars_macro_tex", load(MARS_MAP) as Texture2D)
		if tex_dune_albedo == null:
			tex_dune_albedo = load(MARS_DUNE_ALBEDO) as Texture2D
		shader_material.set_shader_parameter("dune_albedo_tex", tex_dune_albedo)
		shader_material.set_shader_parameter("dune_origin", DUNE_PATCH_ORIGIN)
		shader_material.set_shader_parameter("dune_size", Vector2(DUNE_PATCH_WIDTH, DUNE_PATCH_DEPTH))
		shader_material.set_shader_parameter("tint", tint)
		shader_material.set_shader_parameter("albedo_gain", TERRAIN_BRIGHTNESS)
		return shader_material

	# Fallback: standard material with world-space triplanar mapping.
	var material := StandardMaterial3D.new()
	material.vertex_color_use_as_albedo = true
	material.albedo_color = tint
	material.albedo_texture = tex_diffuse
	material.normal_enabled = true
	material.normal_texture = tex_normal
	material.roughness_texture = tex_roughness
	material.roughness = 0.95
	material.metallic = 0.08
	material.uv1_triplanar = true
	material.uv1_world_triplanar = true
	material.uv1_scale = Vector3.ONE / 6.0
	material.texture_filter = BaseMaterial3D.TEXTURE_FILTER_LINEAR_WITH_MIPMAPS_ANISOTROPIC
	return material


func create_terrain() -> void:
	var verts_x := TERRAIN_RESOLUTION_X + 1
	var verts_z := TERRAIN_RESOLUTION_Z + 1
	var step_x := WORLD_WIDTH / float(TERRAIN_RESOLUTION_X)
	var step_z := WORLD_DEPTH / float(TERRAIN_RESOLUTION_Z)
	var half_x := WORLD_WIDTH * 0.5
	var half_z := WORLD_DEPTH * 0.5

	# Sample the terrain height once per grid point. Both the visual mesh and the
	# collision shape are built from this single grid.
	var heights := PackedFloat32Array()
	heights.resize(verts_x * verts_z)
	for zi in range(verts_z):
		for xi in range(verts_x):
			heights[zi * verts_x + xi] = get_terrain_height(-half_x + xi * step_x, -half_z + zi * step_z)

	var surface := SurfaceTool.new()
	surface.begin(Mesh.PRIMITIVE_TRIANGLES)
	for zi in range(verts_z):
		var z := -half_z + zi * step_z
		for xi in range(verts_x):
			var x := -half_x + xi * step_x
			var h := heights[zi * verts_x + xi]
			surface.set_color(get_terrain_color(x, z, h))
			surface.set_uv(Vector2(x / 6.0, z / 6.0))
			surface.add_vertex(Vector3(x, h, z))

	for zi in range(TERRAIN_RESOLUTION_Z):
		for xi in range(TERRAIN_RESOLUTION_X):
			var i00 := zi * verts_x + xi
			var i10 := i00 + 1
			var i01 := i00 + verts_x
			var i11 := i01 + 1
			surface.add_index(i00)
			surface.add_index(i10)
			surface.add_index(i01)
			surface.add_index(i10)
			surface.add_index(i11)
			surface.add_index(i01)

	surface.generate_normals()
	var terrain_mesh := surface.commit()

	var terrain := StaticBody3D.new()
	terrain.name = "AlienTerrain"
	terrain.collision_layer = 1
	terrain.collision_mask = 0

	var mesh_instance := MeshInstance3D.new()
	mesh_instance.mesh = terrain_mesh
	mesh_instance.material_override = make_terrain_material()
	terrain.add_child(mesh_instance)

	# HeightMapShape3D is much cheaper than a trimesh of the whole terrain.
	# Its samples are 1 unit apart, so the node is scaled to the real grid step.
	var height_shape := HeightMapShape3D.new()
	height_shape.map_width = verts_x
	height_shape.map_depth = verts_z
	height_shape.map_data = heights

	var collision := CollisionShape3D.new()
	collision.shape = height_shape
	collision.scale = Vector3(step_x, 1.0, step_z)
	terrain.add_child(collision)
	add_child(terrain)


func create_player() -> void:
	player = CharacterBody3D.new()
	player.name = "Astronaut"
	player.position = SPAWN_POSITION
	player.collision_layer = 2
	player.collision_mask = 1
	player.floor_max_angle = deg_to_rad(FLOOR_MAX_ANGLE_DEG)
	player.floor_snap_length = 0.6
	player.floor_stop_on_slope = true
	player.floor_constant_speed = true
	player.floor_block_on_wall = true
	player.max_slides = 6
	add_child(player)

	var collision := CollisionShape3D.new()
	var capsule := CapsuleShape3D.new()
	capsule.radius = 0.4
	capsule.height = 1.8
	collision.shape = capsule
	collision.position.y = 0.9
	player.add_child(collision)

	create_astronaut_model()

	camera_pivot = Node3D.new()
	camera_pivot.position.y = 1.5
	player.add_child(camera_pivot)

	# The spring arm pulls the camera in when terrain is between it and the player,
	# so the view never clips into hills while climbing.
	var spring_arm := SpringArm3D.new()
	spring_arm.spring_length = 5.0
	spring_arm.margin = 0.3
	spring_arm.collision_mask = 1
	spring_arm.position.y = 0.7
	var spring_shape := SphereShape3D.new()
	spring_shape.radius = 0.3
	spring_arm.shape = spring_shape
	camera_pivot.add_child(spring_arm)

	player_camera = Camera3D.new()
	player_camera.rotation_degrees.x = -8.0
	player_camera.far = 1400.0
	spring_arm.add_child(player_camera)
	player_camera.current = true


func create_astronaut_model() -> void:
	astronaut_model = Node3D.new()
	astronaut_model.name = "AstronautModel"
	player.add_child(astronaut_model)

	var model_scene := load(ASTRONAUT_MODEL) as PackedScene
	if model_scene == null:
		push_error("Could not load the astronaut model: " + ASTRONAUT_MODEL)
		return

	var suit_model := model_scene.instantiate()
	suit_model.name = "RiggedEVAExplorer"
	# The imported character's local forward axis faces +Z; the game faces -Z.
	suit_model.rotation.y = PI
	astronaut_model.add_child(suit_model)
	astronaut_animation = suit_model.find_child("AnimationPlayer", true, false) as AnimationPlayer

	var skeletons := suit_model.find_children("*", "Skeleton3D", true, false)
	if not skeletons.is_empty():
		skeleton = skeletons[0] as Skeleton3D

	setup_character_animations()


func find_animation(keywords: Array) -> String:
	if astronaut_animation == null:
		return ""
	for anim_name in astronaut_animation.get_animation_list():
		var lowered := String(anim_name).to_lower()
		for keyword in keywords:
			if lowered.contains(String(keyword)):
				return String(anim_name)
	return ""


func setup_character_animations() -> void:
	if astronaut_animation == null:
		return

	print("ASTRA animations found: ", astronaut_animation.get_animation_list())

	anim_walk = "walk" if astronaut_animation.has_animation("walk") else find_animation(["walk"])
	anim_run = "run" if astronaut_animation.has_animation("run") else find_animation(["run", "sprint", "jog"])
	anim_idle = find_animation(["idle", "stand"])
	anim_climb = find_animation(["climb"])
	anim_air = find_animation(["jump", "fall"])

	# Standing, climbing and air poses should loop instead of freezing on the last frame.
	for anim_name in [anim_idle, anim_climb]:
		if anim_name != "":
			astronaut_animation.get_animation(anim_name).loop_mode = Animation.LOOP_LINEAR


func create_space_sky() -> void:
	space_sky_root = Node3D.new()
	space_sky_root.name = "DeepSpaceBackdrop"
	space_sky_root.global_position = player.global_position
	add_child(space_sky_root)

	starfield = MultiMeshInstance3D.new()
	starfield.name = "DeepSpaceStars"
	starfield.cast_shadow = GeometryInstance3D.SHADOW_CASTING_SETTING_OFF

	var star_mesh := SphereMesh.new()
	star_mesh.radius = 0.24
	star_mesh.height = 0.48
	star_mesh.radial_segments = 6
	star_mesh.rings = 4

	var star_material := StandardMaterial3D.new()
	star_material.shading_mode = BaseMaterial3D.SHADING_MODE_UNSHADED
	star_material.albedo_color = Color(0.88, 0.91, 1.0)
	star_material.vertex_color_use_as_albedo = true
	star_material.emission_enabled = true
	star_material.emission = Color(0.72, 0.80, 1.0)
	star_material.emission_energy_multiplier = 1.25
	star_material.disable_fog = true
	star_mesh.material = star_material

	var multi := MultiMesh.new()
	multi.transform_format = MultiMesh.TRANSFORM_3D
	multi.use_colors = true
	multi.mesh = star_mesh
	multi.instance_count = 1800

	for i in range(multi.instance_count):
		var direction := Vector3(
			randf_range(-1.0, 1.0),
			randf_range(-1.0, 1.0),
			randf_range(-1.0, 1.0)
		).normalized()

		var distance := randf_range(300.0, 480.0)
		var star_size := randf_range(0.35, 1.05)
		var star_tint := Color(1.0, 0.86, 0.70).lerp(Color(0.63, 0.76, 1.0), randf())
		if randf() < 0.76:
			star_size *= 0.55
		multi.set_instance_color(i, star_tint)
		var star_transform := Transform3D(
			Basis.IDENTITY.scaled(Vector3.ONE * star_size),
			direction * distance
		)
		multi.set_instance_transform(i, star_transform)

	starfield.multimesh = multi
	space_sky_root.add_child(starfield)

	for style in range(4):
		galaxy_textures.append(make_galaxy_texture(style))
	create_galaxy_field()
	create_ton618_scene()


func create_galaxy_field() -> void:
	for style in range(4):
		var galaxy_field := MultiMeshInstance3D.new()
		galaxy_field.name = "DistantGalaxies_%d" % style
		galaxy_field.cast_shadow = GeometryInstance3D.SHADOW_CASTING_SETTING_OFF
		var quad := QuadMesh.new()
		quad.size = Vector2.ONE
		var material := StandardMaterial3D.new()
		material.shading_mode = BaseMaterial3D.SHADING_MODE_UNSHADED
		material.transparency = BaseMaterial3D.TRANSPARENCY_ALPHA
		material.albedo_texture = galaxy_textures[style]
		material.vertex_color_use_as_albedo = true
		material.emission_enabled = true
		material.emission = Color(0.82, 0.87, 1.0)
		material.emission_energy_multiplier = 0.9
		material.billboard_mode = BaseMaterial3D.BILLBOARD_ENABLED
		material.billboard_keep_scale = true
		material.disable_fog = true
		material.cull_mode = BaseMaterial3D.CULL_DISABLED
		quad.material = material

		var instances := MultiMesh.new()
		instances.transform_format = MultiMesh.TRANSFORM_3D
		instances.use_colors = true
		instances.mesh = quad
		instances.instance_count = 55
		for i in range(instances.instance_count):
			var direction := Vector3(
				randf_range(-1.0, 1.0),
				randf_range(-1.0, 1.0),
				randf_range(-1.0, 1.0)
			).normalized()
			var distance := randf_range(500.0, 1120.0)
			var galaxy_width := randf_range(10.0, 28.0)
			var galaxy_basis := Basis.IDENTITY.rotated(Vector3.BACK, randf_range(0.0, TAU))
			galaxy_basis = galaxy_basis.scaled(Vector3(galaxy_width, galaxy_width * 0.5, 1.0))
			instances.set_instance_transform(i, Transform3D(galaxy_basis, direction * distance))
			var brightness := randf_range(0.55, 1.0)
			instances.set_instance_color(i, Color(brightness, brightness, brightness, randf_range(0.7, 1.0)))
		galaxy_field.multimesh = instances
		space_sky_root.add_child(galaxy_field)


func make_galaxy_texture(style: int) -> Texture2D:
	var image := Image.create(256, 128, false, Image.FORMAT_RGBA8)
	var flatten := 0.48
	if style == 2:
		flatten = 0.22
	elif style == 1:
		flatten = 0.68

	var arm_color := Color(0.52, 0.67, 1.0)
	var core_color := Color(1.0, 0.83, 0.57)
	match style:
		1:
			arm_color = Color(0.74, 0.66, 0.92)
			core_color = Color(1.0, 0.77, 0.49)
		2:
			arm_color = Color(0.91, 0.47, 0.33)
			core_color = Color(1.0, 0.89, 0.67)
		3:
			arm_color = Color(0.66, 0.48, 1.0)
			core_color = Color(1.0, 0.72, 0.58)

	for y in range(128):
		for x in range(256):
			var u := (float(x) - 127.5) / 108.0
			var v := (float(y) - 63.5) / 48.0
			var stretched_v := v / flatten
			var radius := sqrt(u * u + stretched_v * stretched_v)
			if radius > 1.2:
				image.set_pixel(x, y, Color(0.0, 0.0, 0.0, 0.0))
				continue

			var angle := atan2(stretched_v, u)
			var spiral_wave := 0.5 + 0.5 * cos(angle * 2.0 - log(maxf(radius, 0.035)) * 4.2 + float(style))
			var arms := pow(maxf(spiral_wave, 0.0), 5.0)
			if style == 1:
				arms *= 0.12
			elif style == 3:
				arms = pow(maxf(0.5 + 0.5 * cos(angle * 2.0 + radius * 8.0), 0.0), 3.0)
			var core := exp(-radius * radius * 17.0)
			var disk_light := exp(-radius * 2.6) * (0.10 + 0.72 * arms)
			var brightness := clampf(core * 0.95 + disk_light, 0.0, 1.0)
			var rim_t := clampf((radius - 0.82) / 0.38, 0.0, 1.0)
			rim_t = rim_t * rim_t * (3.0 - 2.0 * rim_t)
			var alpha := clampf(brightness * 1.6, 0.0, 0.92) * (1.0 - rim_t)
			var color := arm_color.lerp(core_color, clampf(core * 1.5, 0.0, 1.0)) * brightness
			image.set_pixel(x, y, Color(color.r, color.g, color.b, alpha))

	return ImageTexture.create_from_image(image)


func create_ton618_scene() -> void:
	var scene := Node3D.new()
	scene.name = "TON_618_Tidal_Disruption_Visualization"
	scene.position = Vector3(0.0, 20.0, -320.0)
	space_sky_root.add_child(scene)
	var model_scene := load(TON_618_MODEL) as PackedScene
	if model_scene == null:
		push_error("TON 618 model could not be loaded: " + TON_618_MODEL)
		return
	var black_hole_model := model_scene.instantiate() as Node3D
	black_hole_model.name = "TON_618_Imported_Black_Hole_Model"
	black_hole_model.scale = Vector3.ONE * 420.0
	black_hole_model.position.y = -155.4
	scene.add_child(black_hole_model)
	# The supplied CC0 model includes a small presentation plinth. Hide its base
	# light so the black hole reads as an object in space, not as a display stand.
	var display_base := black_hole_model.find_child("black-hole_base_ring_glow", true, false)
	if display_base is Node3D:
		display_base.visible = false
	accretion_disk_node = black_hole_model

	var giant := MeshInstance3D.new()
	giant.name = "RedGiantStar"
	giant.position = Vector3(-150.0, 0.0, -4.0)
	var giant_mesh := SphereMesh.new()
	giant_mesh.radius = 38.0
	giant_mesh.height = 76.0
	giant_mesh.radial_segments = 80
	giant_mesh.rings = 40
	giant.mesh = giant_mesh
	red_giant_material = StandardMaterial3D.new()
	red_giant_material.shading_mode = BaseMaterial3D.SHADING_MODE_UNSHADED
	red_giant_material.albedo_color = Color(0.92, 0.16, 0.035)
	red_giant_material.emission_enabled = true
	red_giant_material.emission = Color(1.0, 0.11, 0.015)
	red_giant_material.emission_energy_multiplier = 1.8
	red_giant_material.disable_fog = true
	giant.material_override = red_giant_material
	giant.cast_shadow = GeometryInstance3D.SHADOW_CASTING_SETTING_OFF
	scene.add_child(giant)

	red_giant_glow = Sprite3D.new()
	red_giant_glow.name = "RedGiantCorona"
	red_giant_glow.texture = make_radial_glow_texture(Color(1.0, 0.18, 0.025))
	red_giant_glow.position = giant.position
	red_giant_glow.pixel_size = 0.75
	red_giant_glow.shaded = false
	red_giant_glow.billboard = BaseMaterial3D.BILLBOARD_ENABLED
	red_giant_glow.modulate = Color(1.0, 0.62, 0.35, 0.72)
	scene.add_child(red_giant_glow)

	tidal_stream = MeshInstance3D.new()
	tidal_stream.name = "RedGiantTidalStream"
	tidal_stream.mesh = create_tidal_stream_mesh()
	var stream_material := StandardMaterial3D.new()
	stream_material.shading_mode = BaseMaterial3D.SHADING_MODE_UNSHADED
	stream_material.vertex_color_use_as_albedo = true
	stream_material.transparency = BaseMaterial3D.TRANSPARENCY_ALPHA
	stream_material.emission_enabled = true
	stream_material.emission = Color(1.0, 0.20, 0.035)
	stream_material.emission_energy_multiplier = 2.0
	stream_material.disable_fog = true
	tidal_stream.material_override = stream_material
	tidal_stream.cast_shadow = GeometryInstance3D.SHADOW_CASTING_SETTING_OFF
	scene.add_child(tidal_stream)

	var label := Label3D.new()
	label.text = "TON 618"
	label.position = Vector3(0.0, 112.0, 0.0)
	label.billboard = BaseMaterial3D.BILLBOARD_ENABLED
	label.font_size = 38
	label.pixel_size = 0.09
	label.modulate = Color(0.85, 0.88, 1.0)
	label.outline_size = 5
	label.outline_modulate = Color(0.005, 0.004, 0.018, 0.95)
	scene.add_child(label)

func create_tidal_stream_mesh() -> ArrayMesh:
	var surface := SurfaceTool.new()
	surface.begin(Mesh.PRIMITIVE_TRIANGLES)
	var count := 48
	var start := Vector3(-111.0, 0.0, -4.0)
	var control_a := Vector3(-84.0, 0.0, 30.0)
	var control_b := Vector3(-53.0, 0.0, -27.0)
	var finish := Vector3(-24.0, 0.0, 0.0)
	for i in range(count):
		var t0 := float(i) / count
		var t1 := float(i + 1) / count
		var p0 := cubic_bezier(start, control_a, control_b, finish, t0)
		var p1 := cubic_bezier(start, control_a, control_b, finish, t1)
		var width0 := lerpf(10.0, 0.8, t0) * (0.82 + 0.18 * sin(t0 * TAU * 5.0))
		var width1 := lerpf(10.0, 0.8, t1) * (0.82 + 0.18 * sin(t1 * TAU * 5.0))
		var c0 := Color(0.82, 0.055, 0.018, 0.48).lerp(Color(1.0, 0.68, 0.26, 0.96), t0)
		var c1 := Color(0.82, 0.055, 0.018, 0.48).lerp(Color(1.0, 0.68, 0.26, 0.96), t1)
		var v00 := p0 + Vector3(0.0, 0.0, width0)
		var v01 := p0 - Vector3(0.0, 0.0, width0)
		var v10 := p1 + Vector3(0.0, 0.0, width1)
		var v11 := p1 - Vector3(0.0, 0.0, width1)
		for vertex in [[v00, c0], [v01, c0], [v10, c1], [v10, c1], [v01, c0], [v11, c1]]:
			surface.set_color(vertex[1])
			surface.add_vertex(vertex[0])
	return surface.commit()


func cubic_bezier(p0: Vector3, p1: Vector3, p2: Vector3, p3: Vector3, t: float) -> Vector3:
	var inverse := 1.0 - t
	return p0 * inverse * inverse * inverse + p1 * 3.0 * inverse * inverse * t + p2 * 3.0 * inverse * t * t + p3 * t * t * t


func make_radial_glow_texture(glow_color: Color) -> Texture2D:
	var image := Image.create(128, 128, false, Image.FORMAT_RGBA8)
	for y in range(128):
		for x in range(128):
			var radius := Vector2(float(x) - 63.5, float(y) - 63.5).length() / 63.5
			var alpha := pow(clampf(1.0 - radius, 0.0, 1.0), 2.2) * 0.8
			image.set_pixel(x, y, Color(glow_color.r, glow_color.g, glow_color.b, alpha))
	return ImageTexture.create_from_image(image)


func create_rocks() -> void:
	for i in range(250):
		var x := randf_range(-WORLD_WIDTH * 0.48, WORLD_WIDTH * 0.48)
		var z := randf_range(-WORLD_DEPTH * 0.46, WORLD_DEPTH * 0.46)
		var in_dune_patch := (
			x >= DUNE_PATCH_ORIGIN.x
			and x <= DUNE_PATCH_ORIGIN.x + DUNE_PATCH_WIDTH
			and z >= DUNE_PATCH_ORIGIN.y
			and z <= DUNE_PATCH_ORIGIN.y + DUNE_PATCH_DEPTH
		)
		if Vector2(x - SPAWN_POSITION.x, z - SPAWN_POSITION.z).length() < 14.0 or in_dune_patch:
			continue

		var radius := randf_range(0.5, 1.5)
		var height := randf_range(0.6, 2.0)
		var rock_scale := Vector3(randf_range(0.8, 1.6), randf_range(0.5, 1.4), randf_range(0.8, 1.6))

		var rock := StaticBody3D.new()
		rock.collision_layer = 1
		rock.collision_mask = 0
		rock.position = Vector3(x, get_terrain_height(x, z) + 0.3, z)

		var mesh_instance := MeshInstance3D.new()
		var rock_mesh := SphereMesh.new()
		rock_mesh.radius = radius
		rock_mesh.height = height
		mesh_instance.mesh = rock_mesh
		mesh_instance.scale = rock_scale
		# Rocks use the same triplanar rock shader as the terrain, just tinted.
		var rock_tint := Color(randf_range(0.42, 0.62), randf_range(0.18, 0.34), randf_range(0.10, 0.20))
		mesh_instance.material_override = make_terrain_material(rock_tint)
		rock.add_child(mesh_instance)

		# Physics shapes shouldn't be non-uniformly scaled, so use a sphere whose
		# radius averages the rock's horizontal and vertical half-sizes.
		var horizontal_radius := radius * (rock_scale.x + rock_scale.z) * 0.5
		var vertical_radius := height * 0.5 * rock_scale.y
		var shape := SphereShape3D.new()
		shape.radius = (horizontal_radius + vertical_radius) * 0.5
		var collision := CollisionShape3D.new()
		collision.shape = shape
		rock.add_child(collision)

		add_child(rock)


func create_world_objects() -> void:
	create_rocks()


func create_card(parent: Node, panel_position: Vector2, panel_size: Vector2) -> PanelContainer:
	var panel := PanelContainer.new()
	panel.position = panel_position
	panel.custom_minimum_size = panel_size

	var style := StyleBoxFlat.new()
	style.bg_color = Color(0.015, 0.035, 0.075, 0.88)
	style.border_color = Color(0.12, 0.65, 0.82, 0.7)
	style.set_border_width_all(1)
	style.set_corner_radius_all(12)
	style.content_margin_left = 14.0
	style.content_margin_right = 14.0
	style.content_margin_top = 12.0
	style.content_margin_bottom = 12.0
	panel.add_theme_stylebox_override("panel", style)
	parent.add_child(panel)
	return panel


func make_progress_bar(parent: Control, color: Color) -> ProgressBar:
	var bar := ProgressBar.new()
	bar.custom_minimum_size = Vector2(220.0, 13.0)
	bar.min_value = 0.0
	bar.max_value = 100.0
	bar.show_percentage = false

	var background := StyleBoxFlat.new()
	background.bg_color = Color(0.04, 0.08, 0.13, 1.0)
	background.set_corner_radius_all(6)
	bar.add_theme_stylebox_override("background", background)

	var fill := StyleBoxFlat.new()
	fill.bg_color = color
	fill.set_corner_radius_all(6)
	bar.add_theme_stylebox_override("fill", fill)
	parent.add_child(bar)
	return bar


func create_hud() -> void:
	var canvas := CanvasLayer.new()
	canvas.layer = 5
	add_child(canvas)

	var stats_panel := create_card(canvas, Vector2(22.0, 20.0), Vector2(290.0, 110.0))
	var stats_box := VBoxContainer.new()
	stats_box.add_theme_constant_override("separation", 7)
	stats_panel.add_child(stats_box)
	title_label = Label.new()
	title_label.text = "ASTRA  /  EXPLORATION"
	title_label.add_theme_font_size_override("font_size", 18)
	title_label.add_theme_color_override("font_color", Color(0.45, 0.9, 1.0))
	stats_box.add_child(title_label)
	jetpack_value = Label.new()
	stats_box.add_child(jetpack_value)
	jetpack_bar = make_progress_bar(stats_box, Color(0.7, 0.45, 1.0))

	var data_panel := create_card(canvas, Vector2(-322.0, 20.0), Vector2(302.0, 150.0))
	data_panel.anchor_left = 1.0
	data_panel.anchor_right = 1.0
	var data_box := VBoxContainer.new()
	data_box.add_theme_constant_override("separation", 4)
	data_panel.add_child(data_box)
	var data_title := Label.new()
	data_title.text = "TON 618  /  QUASAR"
	data_title.add_theme_font_size_override("font_size", 14)
	data_title.add_theme_color_override("font_color", Color(0.45, 0.9, 1.0))
	data_box.add_child(data_title)
	var data_text := Label.new()
	data_text.text = "SUPERMASSIVE BLACK HOLE\nRED GIANT TIDAL STREAM\nGLOWING ACCRETION DISK\n\nIllustrative deep-space scene"
	data_text.add_theme_font_size_override("font_size", 12)
	data_text.add_theme_color_override("font_color", Color(0.78, 0.85, 0.95))
	data_box.add_child(data_text)

	var controls_panel := create_card(canvas, Vector2(22.0, -170.0), Vector2(310.0, 148.0))
	controls_panel.anchor_top = 1.0
	controls_panel.anchor_bottom = 1.0
	var controls_box := VBoxContainer.new()
	controls_box.add_theme_constant_override("separation", 4)
	controls_panel.add_child(controls_box)
	var controls_title := Label.new()
	controls_title.text = "FIELD CONTROLS"
	controls_title.add_theme_font_size_override("font_size", 15)
	controls_title.add_theme_color_override("font_color", Color(0.45, 0.9, 1.0))
	controls_box.add_child(controls_title)
	var controls_text := Label.new()
	controls_text.text = "WASD Move   SHIFT Sprint\nSPACE Jump   E Jetpack\nESC Release mouse\nPush into a steep slope to CLIMB"
	controls_text.add_theme_font_size_override("font_size", 13)
	controls_text.add_theme_color_override("font_color", Color(0.78, 0.85, 0.95))
	controls_box.add_child(controls_text)

	message_label = Label.new()
	message_label.set_anchors_preset(Control.PRESET_CENTER_BOTTOM)
	message_label.position = Vector2(-250.0, -82.0)
	message_label.size = Vector2(500.0, 42.0)
	message_label.horizontal_alignment = HORIZONTAL_ALIGNMENT_CENTER
	message_label.add_theme_font_size_override("font_size", 20)
	message_label.add_theme_color_override("font_color", Color(0.4, 0.92, 1.0))
	canvas.add_child(message_label)
	update_hud()


func update_hud(delta: float = 0.0) -> void:
	if not is_instance_valid(jetpack_bar):
		return
	var jetpack_percent := 0.0
	if jetpack_recharge_timer > 0.0:
		jetpack_value.text = "JETPACK   RECHARGING  %02.0f s" % jetpack_recharge_timer
		jetpack_percent = 100.0 * (1.0 - jetpack_recharge_timer / JETPACK_RECHARGE_TIME)
	else:
		jetpack_value.text = "JETPACK   FUEL  %03.0f%%" % (100.0 * jetpack_fuel / JETPACK_DURATION)
		jetpack_percent = 100.0 * jetpack_fuel / JETPACK_DURATION
	var blend := 1.0 if delta <= 0.0 else minf(delta * 6.0, 1.0)
	jetpack_bar.value = lerpf(jetpack_bar.value, jetpack_percent, blend)

func show_message(message: String) -> void:
	message_timer = 3.0
	if is_instance_valid(message_label):
		message_label.text = message


func respawn_player() -> void:
	player.position = SPAWN_POSITION
	player.velocity = Vector3.ZERO
	climbing = false
	show_message("RETURNED TO LANDING ZONE")


func _input(event: InputEvent) -> void:
	if not game_started:
		return
	if event.is_action_pressed("ui_cancel"):
		Input.mouse_mode = Input.MOUSE_MODE_VISIBLE
	if event is InputEventMouseButton and event.pressed and event.button_index == MOUSE_BUTTON_LEFT:
		Input.mouse_mode = Input.MOUSE_MODE_CAPTURED
	if event is InputEventMouseMotion and Input.mouse_mode == Input.MOUSE_MODE_CAPTURED:
		camera_pivot.rotate_y(-event.relative.x * MOUSE_SENSITIVITY)
		camera_pivot.rotation.x = clampf(camera_pivot.rotation.x - event.relative.y * MOUSE_SENSITIVITY, deg_to_rad(-60.0), deg_to_rad(60.0))

# Rendered-frame update: refresh the exploration HUD and frame-rate display.
func _process(delta: float) -> void:
	if not game_started:
		return
	ui_timer += delta
	if ui_timer >= UI_UPDATE_INTERVAL:
		update_hud(ui_timer)
		ui_timer = 0.0
	fps_timer += delta
	if fps_timer >= 0.5:
		fps_timer = 0.0
		if is_instance_valid(title_label):
			title_label.text = "ASTRA  /  EXPLORATION        %d FPS" % int(Engine.get_frames_per_second())

func _physics_process(delta: float) -> void:
	if not game_started:
		return

	effect_time += delta
	if is_instance_valid(space_sky_root):
		space_sky_root.global_position = player.global_position
	if is_instance_valid(accretion_disk_node):
		accretion_disk_node.rotate_y(delta * 0.08)
	if is_instance_valid(red_giant_material):
		red_giant_material.emission_energy_multiplier = 1.8 + 0.24 * sin(effect_time * 2.0)
	if is_instance_valid(red_giant_glow):
		red_giant_glow.modulate.a = 0.68 + 0.10 * sin(effect_time * 1.6)
	if is_instance_valid(tidal_stream):
		tidal_stream.modulate.a = 0.82 + 0.18 * sin(effect_time * 2.4)

	# --- input
	var direction := Input.get_vector("move_left", "move_right", "move_forward", "move_back")
	var has_input := direction.length() > 0.0

	var forward := -camera_pivot.global_transform.basis.z
	var right := camera_pivot.global_transform.basis.x
	forward.y = 0.0
	right.y = 0.0
	forward = forward.normalized()
	right = right.normalized()
	var movement := (forward * -direction.y + right * direction.x).normalized()

	if Input.is_action_just_pressed("jump"):
		jump_buffer_timer = JUMP_BUFFER_TIME
	else:
		jump_buffer_timer = maxf(jump_buffer_timer - delta, 0.0)

	var on_floor := player.is_on_floor()
	if on_floor:
		coyote_timer = COYOTE_TIME
	else:
		coyote_timer = maxf(coyote_timer - delta, 0.0)

	var jetpack_wanted := (
		Input.is_action_pressed("jetpack")
		and jetpack_fuel > 0.0
		and jetpack_recharge_timer <= 0.0
	)

	# --- climbing / jetpack / walking
	update_climb_state(delta, direction, movement, jetpack_wanted)
	update_jetpack(delta)

	if climbing:
		process_climbing(direction)
	else:
		process_walking(delta, direction, movement, player.is_on_floor())

	player.move_and_slide()
	if player.is_on_wall():
		wall_normal = player.get_wall_normal()

	# Keep movement inside the expanded 10 hectare main terrain.
	var limit_x := WORLD_WIDTH * 0.5 - WORLD_EDGE_MARGIN
	var limit_z := WORLD_DEPTH * 0.5 - WORLD_EDGE_MARGIN
	player.position.x = clampf(player.position.x, -limit_x, limit_x)
	player.position.z = clampf(player.position.z, -limit_z, limit_z)
	update_model_orientation(delta, movement, has_input)
	update_character_animation(has_input)

	if message_timer > 0.0:
		message_timer -= delta
		if message_timer <= 0.0:
			message_label.text = ""


# ---------------------------------------------------------------- walking

func process_walking(delta: float, direction: Vector2, movement: Vector3, on_floor: bool) -> void:
	var has_input := direction.length() > 0.0

	if not on_floor:
		player.velocity.y -= GRAVITY * delta
	elif player.velocity.y < 0.0:
		player.velocity.y = 0.0

	var sprinting := Input.is_action_pressed("sprint") and has_input
	var speed := SPRINT_SPEED if sprinting else WALK_SPEED

	# Going uphill on a steep (but still walkable) slope is slower.
	if on_floor and has_input:
		var floor_normal := player.get_floor_normal()
		var slope_angle := acos(clampf(floor_normal.y, -1.0, 1.0))
		var downhill := Vector3(floor_normal.x, 0.0, floor_normal.z)
		if downhill.length() > 0.001 and movement.dot(downhill.normalized()) < -0.2:
			var effort := smoothstep(deg_to_rad(20.0), deg_to_rad(FLOOR_MAX_ANGLE_DEG), slope_angle)
			speed *= lerpf(1.0, UPHILL_MIN_SPEED_FACTOR, effort)

	var target_velocity := movement * speed
	var move_rate := 0.0
	if on_floor:
		move_rate = ACCELERATION if has_input else BRAKING
	else:
		move_rate = ACCELERATION * AIR_CONTROL if has_input else AIR_DRAG

	player.velocity.x = move_toward(player.velocity.x, target_velocity.x, move_rate * delta)
	player.velocity.z = move_toward(player.velocity.z, target_velocity.z, move_rate * delta)

	# Jump: buffered presses and a short grace period after leaving the ground make jumping feel responsive.
	var wants_jump := jump_buffer_timer > 0.0 or (Input.is_action_pressed("jump") and on_floor)
	if wants_jump and coyote_timer > 0.0:
		player.velocity.y = JUMP_FORCE
		jump_buffer_timer = 0.0
		coyote_timer = 0.0


# ---------------------------------------------------------------- climbing

func get_surface_up(surface_normal: Vector3) -> Vector3:
	# The direction that points "up the slope" along a surface.
	var up_dir := Vector3.UP - surface_normal * surface_normal.dot(Vector3.UP)
	if up_dir.length() < 0.05:
		return Vector3.UP
	return up_dir.normalized()


func update_climb_state(delta: float, direction: Vector2, movement: Vector3, jetpack_wanted: bool) -> void:
	climb_cooldown = maxf(climb_cooldown - delta, 0.0)
	var on_floor := player.is_on_floor()
	var on_wall := player.is_on_wall()

	if climbing:
		# Jump off the wall.
		if Input.is_action_just_pressed("jump"):
			player.velocity = wall_normal * CLIMB_JUMP_AWAY + Vector3.UP * JUMP_FORCE
			jump_buffer_timer = 0.0
			coyote_timer = 0.0
			stop_climbing()
			return
		# Using the jetpack releases the wall.
		if jetpack_wanted:
			stop_climbing()
			return

		if on_wall:
			climb_lost_timer = 0.0
		else:
			climb_lost_timer += delta

		# Lost the surface (ledge/crest reached), or climbed down to walkable ground.
		var climbing_down := direction.y > 0.1
		if climb_lost_timer > CLIMB_LOST_GRACE or (on_floor and (not on_wall or climbing_down)):
			stop_climbing()
		return

	# Not climbing: grab a steep slope if the player pushes straight into it.
	var pushing_into_wall := false
	if on_wall and movement.length() > 0.0:
		var flat_normal := Vector3(wall_normal.x, 0.0, wall_normal.z)
		if flat_normal.length() > 0.05:
			pushing_into_wall = movement.dot(-flat_normal.normalized()) > CLIMB_ENTRY_PUSH_DOT

	if pushing_into_wall and not jetpack_wanted and climb_cooldown <= 0.0:
		wall_contact_time += delta
		if wall_contact_time >= CLIMB_ENTRY_TIME:
			start_climbing()
	else:
		wall_contact_time = 0.0


func start_climbing() -> void:
	climbing = true
	wall_contact_time = 0.0
	climb_lost_timer = 0.0
	if not climb_hint_shown:
		climb_hint_shown = true
		show_message("CLIMBING  -  SPACE: JUMP OFF   E: JETPACK")


func stop_climbing() -> void:
	climbing = false
	wall_contact_time = 0.0
	climb_lost_timer = 0.0
	climb_cooldown = CLIMB_COOLDOWN
	climb_move_amount = 0.0


func process_climbing(direction: Vector2) -> void:
	var surface_normal := wall_normal
	var up_dir := get_surface_up(surface_normal)

	# A/D move along the surface relative to the camera; W/S move up/down the slope.
	var camera_right := camera_pivot.global_transform.basis.x
	camera_right.y = 0.0
	camera_right = camera_right.normalized()
	var side := camera_right - surface_normal * surface_normal.dot(camera_right)
	if side.length() < 0.2:
		side = up_dir.cross(surface_normal)
	side = side.normalized()

	var speed := CLIMB_SPEED * (1.6 if Input.is_action_pressed("sprint") else 1.0)
	climb_move_amount = direction.length()
	# A small push into the surface keeps the astronaut in contact with it.
	player.velocity = (up_dir * -direction.y + side * direction.x) * speed - surface_normal * CLIMB_STICK_SPEED


# ---------------------------------------------------------------- character visuals

func sample_ground() -> Dictionary:
	var space_state := get_world_3d().direct_space_state
	var from := player.global_position + Vector3.UP * 1.0
	var to := player.global_position + Vector3.DOWN * 1.5
	var query := PhysicsRayQueryParameters3D.create(from, to, 1)
	return space_state.intersect_ray(query)


func update_model_orientation(delta: float, movement: Vector3, has_input: bool) -> void:
	if astronaut_model == null:
		return

	var blend := 1.0 - exp(-14.0 * delta)
	var target_quat := Quaternion.IDENTITY
	var target_offset := 0.0

	if climbing:
		# Chest toward the slope, head up the slope.
		var up_dir := get_surface_up(wall_normal)
		var side := up_dir.cross(wall_normal).normalized()
		target_quat = Basis(side, up_dir, wall_normal).get_rotation_quaternion()
		model_yaw = atan2(wall_normal.x, wall_normal.z)
		ground_normal = Vector3.UP
	else:
		if has_input:
			var target_angle := atan2(-movement.x, -movement.z)
			model_yaw = lerp_angle(model_yaw, target_angle, minf(12.0 * delta, 1.0))

		var target_normal := Vector3.UP
		if player.is_on_floor():
			var hit := sample_ground()
			if not hit.is_empty():
				target_normal = hit["normal"]
				var hit_position: Vector3 = hit["position"]
				# The capsule rests on a slope with its curved bottom, so on slopes the
				# origin floats slightly above the surface. Lower the model to compensate.
				target_offset = clampf(hit_position.y - player.global_position.y, -MAX_FOOT_DROP, 0.0)

		ground_normal = ground_normal.lerp(target_normal, blend).normalized()
		var tilt := Quaternion.IDENTITY.slerp(Quaternion(Vector3.UP, ground_normal), GROUND_TILT_AMOUNT)
		target_quat = tilt * Basis(Vector3.UP, model_yaw).get_rotation_quaternion()

	model_quat = model_quat.slerp(target_quat, blend)
	astronaut_model.basis = Basis(model_quat)
	model_height_offset = lerpf(model_height_offset, target_offset, blend)
	astronaut_model.position.y = model_height_offset + MODEL_FOOT_OFFSET


func hold_rest_pose() -> void:
	# No suitable animation: stop the player and put every bone back in its rest pose.
	# (Stopping alone leaves the skeleton frozen in whatever frame was playing, which is
	# what made one foot hang in the air when standing still.)
	if rest_pose_applied:
		return
	if astronaut_animation.is_playing():
		astronaut_animation.stop()
	if skeleton != null:
		skeleton.reset_bone_poses()
	rest_pose_applied = true


func update_character_animation(has_input: bool) -> void:
	if astronaut_animation == null:
		return

	var on_floor := player.is_on_floor()
	var planar_speed := Vector2(player.velocity.x, player.velocity.z).length()
	var desired := ""
	var speed_scale := 1.0

	if climbing:
		desired = anim_climb if anim_climb != "" else anim_walk
		speed_scale = climb_move_amount * 1.4
	elif has_input and on_floor and planar_speed > 0.25:
		var running := Input.is_action_pressed("sprint") and anim_run != ""
		desired = anim_run if running else anim_walk
		var reference_speed := SPRINT_SPEED if running else WALK_SPEED
		var reference_rate := RUN_ANIM_RATE if running else WALK_ANIM_RATE
		speed_scale = reference_rate * clampf(planar_speed / reference_speed, 0.25, 1.0)
	elif not on_floor and anim_air != "":
		desired = anim_air
	else:
		desired = anim_idle

	if desired == "":
		hold_rest_pose()
		return

	rest_pose_applied = false
	if astronaut_animation.current_animation != desired or not astronaut_animation.is_playing():
		astronaut_animation.play(desired, 0.15)
	astronaut_animation.speed_scale = speed_scale


# ---------------------------------------------------------------- jetpack / effects

func update_jetpack(delta: float) -> void:
	if jetpack_recharge_timer > 0.0:
		jetpack_recharge_timer = maxf(jetpack_recharge_timer - delta, 0.0)
		if jetpack_recharge_timer == 0.0:
			jetpack_fuel = JETPACK_DURATION
			show_message("JETPACK RECHARGED")

	var using_jetpack := (
		Input.is_action_pressed("jetpack")
		and jetpack_fuel > 0.0
		and jetpack_recharge_timer <= 0.0
	)

	if using_jetpack:
		player.velocity.y = minf(player.velocity.y + JETPACK_THRUST * delta, 12.0)
		jetpack_fuel = maxf(jetpack_fuel - delta, 0.0)

		if jetpack_fuel <= 0.0:
			jetpack_recharge_timer = JETPACK_RECHARGE_TIME
			show_message("JETPACK EMPTY - RECHARGING")
