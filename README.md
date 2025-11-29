# escape_game_full_2minigames.py
import pygame, sys, random, math
from collections import deque

pygame.init()

# ---------------- Config ----------------
WIDTH, HEIGHT = 1200, 720
FPS = 60
MAP_W, MAP_H = 2800, 2000
PLAYER_SIZE = 48

# ---------------- Helpers ----------------
def clamp(x, a, b): return max(a, min(b, x))

# ---------------- Display & Clock ----------------
screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption("Escape Game - 2 Minigames")
clock = pygame.time.Clock()

# ---------------- Camera ----------------
cam_x, cam_y = 0.0, 0.0
CAM_SMOOTH = 0.12
def update_camera(target_rect):
    global cam_x, cam_y
    tx = target_rect.centerx - WIDTH//2
    ty = target_rect.centery - HEIGHT//2
    cam_x += (tx - cam_x) * CAM_SMOOTH
    cam_y += (ty - cam_y) * CAM_SMOOTH
    cam_x = clamp(cam_x, 0, MAP_W - WIDTH)
    cam_y = clamp(cam_y, 0, MAP_H - HEIGHT)

def world_to_screen(x, y):
    return int(x - cam_x), int(y - cam_y)

# ---------------- Fonts ----------------
FONT = pygame.font.Font(None, 26)
BIG = pygame.font.Font(None, 56)
SMALL = pygame.font.Font(None, 20)

# ---------------- Environment ----------------
player = pygame.Rect(150, 150, PLAYER_SIZE, PLAYER_SIZE) # ukuran pemain di atur di sini (width x high)

# Walls/buildings (Rect list)
walls = []
walls.append(pygame.Rect(1000, 300, 700, 450))
walls.append(pygame.Rect(300, 800, 900, 420))
walls.append(pygame.Rect(1700, 1400, 600, 320))
walls.extend([pygame.Rect(1200,900,400,20), pygame.Rect(600,1500,700,20)])

# Trees / rocks / animals (visual + collidable)
trees = []
rocks = []
animals = []
for _ in range(40):
    x = random.randint(80, MAP_W-120); y = random.randint(80, MAP_H-120)
    w = random.randint(36,80); h = w + random.randint(10,40)
    trees.append(pygame.Rect(x,y,w,h))
for _ in range(20):
    x = random.randint(80, MAP_W-120); y = random.randint(80, MAP_H-120)
    rocks.append(pygame.Rect(x,y, random.randint(24,48), random.randint(16,32)))
for _ in range(8):
    x = random.randint(120, MAP_W-160); y = random.randint(120, MAP_H-160)
    animals.append({
        "rect": pygame.Rect(x,y,36,28),
        "dir": random.choice(["left","right","idle"]),
        "speed": random.uniform(0.3,1.0),
        "timer": random.randint(20,200)
    })

# ---------------- Tasks / Objects ----------------
# Only two stations now: Wires and Code (Number)
task_stations = [
    {"rect": pygame.Rect(600,450,96,96), "name":"Wires", "completed": False},
    {"rect": pygame.Rect(1800,380,96,96), "name":"Code",  "completed": False},
]

key_rect = pygame.Rect(2500, 300, 28, 28)
key_visible = False
has_key = False
exit_door = pygame.Rect(2680, 1700, 96, 160)

# ---------------- Player movement & animation ----------------
player_dir = "down" # starting position ( means facing down)
frame = 0.0
FRAME_SPEED = 0.20
anim_groups = {}

def make_frames_for_direction(base_color, eye_offset_x=0, eye_offset_y=0):    ##############################################
    frames = []
    for i in range(4):
        s = pygame.Surface((PLAYER_SIZE, PLAYER_SIZE), pygame.SRCALPHA)
        pygame.draw.rect(s, base_color, (0,0,PLAYER_SIZE,PLAYER_SIZE), border_radius=8)
        eye_x = PLAYER_SIZE//2 - 6 + eye_offset_x + (i%2)*2
        eye_y = PLAYER_SIZE//3 + eye_offset_y + (i%2)
        pygame.draw.circle(s, (255,255,255), (eye_x, eye_y), 5)
        pygame.draw.circle(s, (255,255,255), (eye_x+12, eye_y), 5)
        frames.append(s)
    return frames

anim_groups["down"]  = make_frames_for_direction((20,120,255), 0, 0)
anim_groups["up"]    = make_frames_for_direction((0,160,220), 0, -2)
anim_groups["left"]  = make_frames_for_direction((10,100,200), -2, 0)
anim_groups["right"] = make_frames_for_direction((40,140,240), 2, 0)

BASE_SPEED = 4.0
RUN_MULT = 1.8
DIAG_NORMALIZE = True

# ---------------- Debug / UI toggles ----------------
debug_draw = False

# ---------------- Utility collision functions ----------------
def move_with_collisions(rect, dx, dy, obstacles):
    rect.x += dx
    collided = [o for o in obstacles if rect.colliderect(o)]
    for o in collided:
        if dx > 0: rect.right = o.left
        elif dx < 0: rect.left = o.right
    rect.y += dy
    collided = [o for o in obstacles if rect.colliderect(o)]
    for o in collided:
        if dy > 0: rect.bottom = o.top
        elif dy < 0: rect.top = o.bottom
    rect.x = clamp(rect.x, 0, MAP_W - rect.width)
    rect.y = clamp(rect.y, 0, MAP_H - rect.height)
    return rect

# ---------------- Drawing helpers ----------------
def draw_rounded_rect(surface, rect, color, radius=8, border=0, border_color=(0,0,0)):
    if border:
        pygame.draw.rect(surface, border_color, rect, border_radius=radius)
        inner = pygame.Rect(rect)
        inner.inflate_ip(-border*2, -border*2)
        pygame.draw.rect(surface, color, inner, border_radius=max(0, radius-border))
    else:
        pygame.draw.rect(surface, color, rect, border_radius=radius)

# ---------------- MINIGAME 1: Wires ----------------
class WiresMinigame:
    def __init__(self):
        self.left = [(-200, -80), (-200, -10), (-200, 60), (-200, 130)]
        self.right = [(200, -80), (200, -10), (200, 60), (200, 130)]
        self.colors = [(220,60,60), (80,220,80), (70,140,220), (240,220,70)]
        self.start()

    def start(self):                     
        order = list(range(4)); random.shuffle(order)
        self.right_colors = [ self.colors[i] for i in order ]
        self.assignments = [-1]*4
        self.dragging = None
        self.drag_pos = (0,0)
        self.popup_alpha = 0
        self.popup_open = True
        self.completed = False

    def update_popup_anim(self):              ################################################################
        if self.popup_open and self.popup_alpha < 255:
            self.popup_alpha = clamp(self.popup_alpha + 25, 0, 255)
        elif not self.popup_open and self.popup_alpha > 0:
            self.popup_alpha = clamp(self.popup_alpha - 25, 0, 255)

    def handle_event(self, event):
        cx, cy = WIDTH//2, HEIGHT//2
        if event.type == pygame.MOUSEBUTTONDOWN and event.button == 1:
            mx,my = event.pos
            for i,(lx,ly) in enumerate(self.left):
                rx = cx + lx; ry = cy + ly
                if (mx-rx)**2 + (my-ry)**2 < 26**2:
                    self.dragging = i; self.drag_pos = (mx,my); return
        if event.type == pygame.MOUSEMOTION and self.dragging is not None:
            self.drag_pos = event.pos
        if event.type == pygame.MOUSEBUTTONUP and event.button == 1 and self.dragging is not None:
            mx,my = event.pos
            matched = -1
            for j,(rx_off,ry_off) in enumerate(self.right):
                rx = cx + rx_off; ry = cy + ry_off
                if (mx-rx)**2 + (my-ry)**2 < 28**2:
                    if self.colors[self.dragging] == self.right_colors[j]:
                        matched = j
                    break
            if matched != -1:
                self.assignments[self.dragging] = matched
            else:
                self.assignments[self.dragging] = -1
            self.dragging = None
        if event.type == pygame.KEYDOWN and event.key == pygame.K_SPACE:
            if all(a != -1 for a in self.assignments):
                self.completed = True
                self.popup_open = False

    def draw(self, surf):
        if self.popup_alpha == 0: return
        popup = pygame.Surface((700,460), pygame.SRCALPHA)
        popup.fill((14,18,26, self.popup_alpha))
        rect = popup.get_rect(center=(WIDTH//2, HEIGHT//2))
        surf.blit(popup, rect.topleft)
        draw_rounded_rect(surf, rect.inflate(-10,-10), (22,28,38), radius=14)
        cx, cy = WIDTH//2, HEIGHT//2
        title = FONT.render("Fix the wires: Match color to color", True, (230,230,230))
        surf.blit(title, (cx - title.get_width()//2, cy - 200))
        for i,(lx,ly) in enumerate(self.left):
            x = cx + lx; y = cy + ly
            pygame.draw.circle(surf, self.colors[i], (x,y), 22); pygame.draw.circle(surf, (0,0,0), (x,y), 2, 2)
        for j,(rx_off,ry_off) in enumerate(self.right):
            x = cx + rx_off; y = cy + ry_off
            pygame.draw.circle(surf, self.right_colors[j], (x,y), 22); pygame.draw.circle(surf,(0,0,0),(x,y),2,2)
        for i,assigned in enumerate(self.assignments):
            if assigned != -1:
                lx,ly = self.left[i]; rx_off,ry_off = self.right[assigned]
                x1 = cx + lx; y1 = cy + ly; x2 = cx + rx_off; y2 = cy + ry_off
                pygame.draw.line(surf, self.colors[i], (x1,y1), (x2,y2), 8)
        if self.dragging is not None:
            lx,ly = self.left[self.dragging]; x1 = cx + lx; y1 = cy + ly
            mx,my = self.drag_pos
            pygame.draw.line(surf, self.colors[self.dragging], (x1,y1), (mx,my), 6)
        if all(a != -1 for a in self.assignments):
            inst = FONT.render("All wires matched! Press SPACE to finish", True, (200,200,200))
            surf.blit(inst, (cx - inst.get_width()//2, cy + 200))
        else:
            inst = FONT.render("Drag left wire onto matching right node", True, (200,200,200))
            surf.blit(inst, (cx - inst.get_width()//2, cy + 200))

# ---------------- MINIGAME 2: Number input (paper) ----------------
class NumberGame:
    def __init__(self):
        self.code = random.randint(1000, 9999)
        self.user = ""
        self.popup_alpha = 0
        self.popup_open = True
        self.completed = False

    def handle_event(self, event):
        if event.type == pygame.KEYDOWN:
            if event.key == pygame.K_BACKSPACE:
                self.user = self.user[:-1]
            elif event.key == pygame.K_RETURN:
                if self.user == str(self.code):
                    self.completed = True
                    self.popup_open = False
            else:
                name = pygame.key.name(event.key)
                if name.isdigit() and len(self.user) < 4:
                    self.user += name

    def update_popup_anim(self):
        if self.popup_open and self.popup_alpha < 255:
            self.popup_alpha = clamp(self.popup_alpha + 25, 0, 255)
        elif not self.popup_open and self.popup_alpha > 0:
            self.popup_alpha = clamp(self.popup_alpha - 25, 0, 255)

    def draw(self, surf):
        if self.popup_alpha == 0: return
        popup = pygame.Surface((560,260), pygame.SRCALPHA)
        popup.fill((12,14,20,self.popup_alpha))
        rect = popup.get_rect(center=(WIDTH//2, HEIGHT//2))
        surf.blit(popup, rect.topleft)
        draw_rounded_rect(surf, rect.inflate(-10,-10), (24,28,34), radius=12)
        cx, cy = rect.center
        title = BIG.render("Note: Read the paper", True, (230,230,230))
        surf.blit(title, (cx - title.get_width()//2, cy - 80))
        code_txt = FONT.render(f"Code on paper: {self.code}", True, (240,240,240))
        surf.blit(code_txt, (cx - code_txt.get_width()//2, cy - 20))
        input_txt = FONT.render(f"Enter code and press Enter: {self.user}", True, (200,200,200))
        surf.blit(input_txt, (cx - input_txt.get_width()//2, cy + 30))

# ---------------- Attach minigame instances ----------------
wires_game = WiresMinigame()
number_game = None

# ---------------- Main loop variables ----------------
running = True
state = "menu"   # menu, playing, win
menu_choice = 0
current_station = None   # station dict when popup open
active_game = None       # one of wires_game / number_game
debug_draw = False

# ---------------- Main loop ----------------
while running:
    dt = clock.tick(FPS)
    for event in pygame.event.get():
        if event.type == pygame.QUIT:
            running = False
            break

        if state == "menu":
            if event.type == pygame.KEYDOWN:
                if event.key in (pygame.K_UP, pygame.K_w): menu_choice = max(0, menu_choice-1)
                if event.key in (pygame.K_DOWN, pygame.K_s): menu_choice = min(1, menu_choice+1)
                if event.key == pygame.K_RETURN:
                    if menu_choice == 0:
                        state = "playing"; player.x, player.y = 150,150
                    else:
                        running = False

        elif state == "playing":
            # If popup / minigame active, route events to that game
            if active_game is not None:
                # wires uses mouse + space; number uses keyboard
                if isinstance(active_game, WiresMinigame):
                    active_game.handle_event(event)
                    if event.type == pygame.KEYDOWN and event.key == pygame.K_q:
                        # cancel wires progress
                        active_game.popup_open = False; active_game = None; current_station = None
                    if active_game.completed:
                        # mark station completed
                        if current_station:
                            current_station["completed"] = True
                        active_game.popup_open = False; active_game = None; current_station = None
                else:
                    # NumberGame handling
                    if event.type == pygame.KEYDOWN:
                        active_game.handle_event(event)
                        if event.type == pygame.KEYDOWN and event.key == pygame.K_q:
                            # cancel
                            active_game.popup_open = False; active_game = None; current_station = None
                        if active_game.completed:
                            if current_station:
                                current_station["completed"] = True
                            active_game.popup_open = False; active_game = None; current_station = None
                # continue to next event
                continue

            # No active popup: general world input
            if event.type == pygame.KEYDOWN:
                if event.key == pygame.K_ESCAPE:
                    state = "menu"
                elif event.key == pygame.K_e:
                    # interact with nearest incomplete station
                    found = None
                    for s in task_stations:
                        if player.colliderect(s["rect"]) and not s.get("completed", False):
                            found = s; break
                    if found:
                        current_station = found
                        idx = task_stations.index(found)
                        # open the minigame corresponding to station index
                        if idx == 0:
                            wires_game.start(); active_game = wires_game
                        elif idx == 1:
                            number_game = NumberGame(); active_game = number_game
                    # pick up key
                    if player.colliderect(key_rect) and key_visible and not has_key:
                        has_key = True; key_visible = False
                    # exit
                    if player.colliderect(exit_door) and has_key:
                        state = "win"
                elif event.key == pygame.K_F3:
                    debug_draw = not debug_draw

        elif state == "win":
            if event.type == pygame.KEYDOWN:
                state = "menu"
                # reset world
                for s in task_stations: s["completed"] = False
                key_visible = False; has_key = False; current_station = None
                wires_game = WiresMinigame(); number_game = None
                active_game = None; player.x, player.y = 150,150

    if not running:
        break

    # ---- Update (gameplay) ----
    if state == "playing" and active_game is None:
        keys = pygame.key.get_pressed()
        vx = vy = 0.0
        if keys[pygame.K_LEFT] or keys[pygame.K_a]: vx -= 1.0
        if keys[pygame.K_RIGHT] or keys[pygame.K_d]: vx += 1.0
        if keys[pygame.K_UP] or keys[pygame.K_w]: vy -= 1.0
        if keys[pygame.K_DOWN] or keys[pygame.K_s]: vy += 1.0
        if DIAG_NORMALIZE and vx != 0 and vy != 0:
            mag = math.sqrt(vx*vx + vy*vy); vx /= mag; vy /= mag
        speed = BASE_SPEED * (RUN_MULT if keys[pygame.K_LSHIFT] or keys[pygame.K_RSHIFT] else 1.0)
        dx = vx * speed; dy = vy * speed

        obstacles = []
        obstacles.extend(walls); obstacles.extend(trees); obstacles.extend(rocks)
        for a in animals: obstacles.append(a["rect"])

        player = move_with_collisions(player, dx, dy, obstacles)

        if vx > 0: player_dir = "right"
        elif vx < 0: player_dir = "left"
        if vy > 0: player_dir = "down"
        elif vy < 0: player_dir = "up"
        moving = (vx != 0 or vy != 0)
        if moving:
            frame += FRAME_SPEED
            if frame >= len(anim_groups[player_dir]): frame = 0.0
        else:
            frame = 0.0

        for a in animals:
            a["timer"] -= dt
            if a["timer"] <= 0:
                a["timer"] = random.randint(1000,5000)
                a["dir"] = random.choice(["left","right","idle"])
                a["speed"] = random.uniform(0.2,1.0)
            if a["dir"] == "left": a["rect"].x -= a["speed"]
            elif a["dir"] == "right": a["rect"].x += a["speed"]
            a["rect"].x = clamp(a["rect"].x, 60, MAP_W-60)

        update_camera(player)

        # spawn key once all tasks completed
        if not key_visible and not has_key and all(s.get("completed", False) for s in task_stations):
            key_visible = True
            key_rect.center = (MAP_W//2, MAP_H//2)

    # update popup anims if active
    if isinstance(active_game, WiresMinigame):
        active_game.update_popup_anim()
    elif isinstance(active_game, NumberGame):
        active_game.update_popup_anim()

    # ---- Draw world ----
    screen.fill((126, 200, 150))
    for x in range(0, MAP_W, 160):
        sx, _ = world_to_screen(x, 0); pygame.draw.line(screen, (120,170,130), (sx,0), (sx,HEIGHT), 1)
    for y in range(0, MAP_M, 160) if False else range(0, MAP_H, 160):
        _, sy = world_to_screen(0, y); pygame.draw.line(screen, (120,170,130), (0,sy), (WIDTH,sy), 1)

    for w in walls:
        sx, sy = world_to_screen(w.x, w.y); r = pygame.Rect(sx, sy, w.width, w.height)
        pygame.draw.rect(screen, (200,200,210), r); pygame.draw.rect(screen, (110,110,120), r, 3)

    for t in trees:
        sx, sy = world_to_screen(t.x, t.y)
        trunk = pygame.Rect(sx + t.width//3, sy + t.height//2, t.width//3, t.height//2)
        pygame.draw.rect(screen, (90,55,25), trunk)
        pygame.draw.ellipse(screen, (20,120,20), (sx, sy - t.height//3, t.width, t.height))

    for r_ in rocks:
        sx, sy = world_to_screen(r_.x, r_.y)
        pygame.draw.ellipse(screen, (140,120,110), (sx, sy, r_.width, r_.height))

    for a in animals:
        r = a["rect"]; sx, sy = world_to_screen(r.x, r.y)
        pygame.draw.ellipse(screen, (200,150,90), (sx, sy, r.width, r.height))
        pygame.draw.circle(screen, (10,10,10), (int(sx + r.width*0.7), int(sy + r.height*0.3)), 3)

    for s in task_stations:
        sx, sy = world_to_screen(s["rect"].x, s["rect"].y)
        col = (80,200,120) if s.get("completed", False) else (160,80,200)
        draw_rounded_rect(screen, pygame.Rect(sx, sy, s["rect"].width, s["rect"].height), col, radius=10)
        lab = SMALL.render(s["name"], True, (255,255,255)); screen.blit(lab, (sx + 6, sy + 6))

    if key_visible and not has_key:
        sx, sy = world_to_screen(key_rect.x, key_rect.y)
        pygame.draw.rect(screen, (255,230,80), (sx, sy, key_rect.width, key_rect.height))

    ex_sx, ex_sy = world_to_screen(exit_door.x, exit_door.y)
    draw_rounded_rect(screen, pygame.Rect(ex_sx, ex_sy, exit_door.width, exit_door.height),
                      (80,200,100) if has_key else (200,60,60), radius=8)
    lab = FONT.render("EXIT", True, (10,10,10)); screen.blit(lab, (ex_sx + exit_door.width//2 - lab.get_width()//2, ex_sy + exit_door.height//2 - lab.get_height()//2))

    sx, sy = world_to_screen(player.x, player.y)
    frames = anim_groups[player_dir]; img = frames[int(frame) % len(frames)]; screen.blit(img, (sx, sy))

    hud = pygame.Rect(12,12,360,110); draw_rounded_rect(screen, hud, (18,20,28), radius=10, border=2, border_color=(60,60,80))
    pos_text = FONT.render(f"Pos: {player.x},{player.y}", True, (200,200,200)); screen.blit(pos_text, (hud.x + 12, hud.y + 12))
    tasks_done = sum(1 for s in task_stations if s.get("completed", False))
    task_txt = FONT.render(f"Tasks done: {tasks_done}/{len(task_stations)}", True, (200,200,200)); screen.blit(task_txt, (hud.x + 12, hud.y + 36))
    inst = FONT.render("WASD/Arrows move • Shift run • E interact • Q close popup • F3 debug", True, (180,180,180)); screen.blit(inst, (hud.x + 12, hud.y + 64))

    if state == "playing" and active_game is None:
        for s in task_stations:
            if player.colliderect(s["rect"]) and not s.get("completed", False):
                sx, sy = world_to_screen(s["rect"].x, s["rect"].y)
                hint = FONT.render("Press E to interact", True, (240,240,240))
                bubble = pygame.Surface((hint.get_width()+20, hint.get_height()+12), pygame.SRCALPHA); bubble.fill((12,12,18,190))
                bx = sx + s["rect"].width//2 - bubble.get_width()//2; by = sy - bubble.get_height() - 8
                screen.blit(bubble, (bx,by)); screen.blit(hint, (bx+10,by+6))

    if key_visible and not has_key and player.colliderect(key_rect):
        sx, sy = world_to_screen(key_rect.x, key_rect.y)
        draw_rounded_rect(screen, pygame.Rect(sx-6, sy-28, 140, 26), (22,22,28), radius=8)
        screen.blit(FONT.render("Press E to pick up key", True, (230,230,230)), (sx+6, sy-24))

    if player.colliderect(exit_door):
        sx, sy = world_to_screen(exit_door.x, exit_door.y)
        if has_key:
            draw_rounded_rect(screen, pygame.Rect(sx-6, sy-28, 120, 26), (20,120,40), radius=8)
            screen.blit(FONT.render("Press E to Exit", True, (10,10,10)), (sx+12, sy-24))
        else:
            draw_rounded_rect(screen, pygame.Rect(sx-6, sy-28, 160, 26), (120,40,40), radius=8)
            screen.blit(FONT.render("Key required to exit", True, (255,230,230)), (sx+10, sy-24))

    # popup / active minigame draw
    if active_game is not None:
        overlay = pygame.Surface((WIDTH, HEIGHT), pygame.SRCALPHA); overlay.fill((8,8,12,200)); screen.blit(overlay, (0,0))
        if isinstance(active_game, WiresMinigame):
            active_game.draw(screen)
        elif isinstance(active_game, NumberGame):
            active_game.draw(screen)
        closer = FONT.render("Press Q to cancel (progress lost)", True, (220,220,220)); screen.blit(closer, (WIDTH//2 - closer.get_width()//2, HEIGHT - 40))

    # debug draw
    if debug_draw:
        pygame.draw.rect(screen, (255,0,0), ( -int(cam_x), -int(cam_y), 4, 4))
        for w in walls:
            sx, sy = world_to_screen(w.x, w.y); pygame.draw.rect(screen, (255,0,0), (sx, sy, w.width, w.height), 2)
        for t in trees:
            sx, sy = world_to_screen(t.x, t.y); pygame.draw.rect(screen, (255,120,0), (sx, sy, t.width, t.height), 1)
        for r_ in rocks:
            sx, sy = world_to_screen(r_.x, r_.y); pygame.draw.rect(screen, (180,100,60), (sx, sy, r_.width, r_.height), 1)
        for a in animals:
            r = a["rect"]; sx, sy = world_to_screen(r.x, r.y); pygame.draw.rect(screen, (0,255,0), (sx, sy, r.width, r.height), 1)
        sx, sy = world_to_screen(player.x, player.y); pygame.draw.rect(screen, (0,255,0), (sx, sy, player.width, player.height), 2)

    if state == "menu":
        overlay = pygame.Surface((WIDTH, HEIGHT), pygame.SRCALPHA); overlay.fill((6,10,14,210)); screen.blit(overlay, (0,0))
        title = BIG.render("Escape Game — 2 Minigames", True, (240,240,240)); screen.blit(title, (WIDTH//2 - title.get_width()//2, 120))
        choices = ["Start Game", "Quit"]
        for i, c in enumerate(choices):
            col = (255,255,255) if menu_choice==i else (170,170,170)
            txt = BIG.render(c, True, col) if menu_choice==i else FONT.render(c, True, col)
            screen.blit(txt, (WIDTH//2 - txt.get_width()//2, 320 + i*80))

    if state == "win":
        overlay = pygame.Surface((WIDTH, HEIGHT), pygame.SRCALPHA); overlay.fill((6,10,14,220)); screen.blit(overlay, (0,0))
        msg = BIG.render("YOU ESCAPED!", True, (255,245,200)); screen.blit(msg, (WIDTH//2 - msg.get_width()//2, HEIGHT//2 - 40))
        sub = FONT.render("Press any key to return to menu", True, (200,200,200)); screen.blit(sub, (WIDTH//2 - sub.get_width()//2, HEIGHT//2 + 40))

    pygame.display.flip()

pygame.quit()
sys.exit()
