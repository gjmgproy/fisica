import pygame
import sys
import math
from pygame.locals import *

pygame.init()

# E - define a janela principal
WIDTH, HEIGHT = 1500, 800
screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption('Simulação de Plano Inclinado com Rampa Plana')

# G1 - define cores e fontes
WHITE = (255, 255, 255)
BLACK = (0, 0, 0)
RED = (255, 0, 0)
GRAY = (200, 200, 200)

font = pygame.font.SysFont('Arial', 20)
title_font = pygame.font.SysFont('Arial', 24, bold=True)

# G2 - define variáveis físicas e estados iniciais
angle = 0
mass = 0
length = 0
g = 9.81
mu = 0.2

input_mode = True
simulation_running = False
object_pos = 0
velocity = 0
final_velocity = 0
time = 0

# G1 - define variáveis da rampa plana
flat_length = 0
flat_pos = 0
on_flat = False

# E - define caixas de input para os parâmetros
input_boxes = {
    "angle": {"label": "Ângulo (graus):", "value": "", "rect": pygame.Rect(300, 200, 200, 30)},
    "mass": {"label": "Massa (kg):", "value": "", "rect": pygame.Rect(300, 260, 200, 30)},
    "length": {"label": "Comprimento (m):", "value": "", "rect": pygame.Rect(300, 320, 200, 30)}
}
active_box = None

# G2 - desenha a tela de entrada de dados
def draw_input_screen():
    screen.fill(GRAY)
    title = title_font.render("Insira os parâmetros da simulação", True, BLACK)
    screen.blit(title, (WIDTH // 2 - title.get_width() // 2, 100))

    for key, box in input_boxes.items():
        label = font.render(box["label"], True, BLACK)
        screen.blit(label, (box["rect"].x - 180, box["rect"].y + 5))
        pygame.draw.rect(screen, RED if active_box == key else BLACK, box["rect"], 2)
        text_surface = font.render(box["value"], True, BLACK)
        screen.blit(text_surface, (box["rect"].x + 5, box["rect"].y + 5))

    info_text = font.render("Pressione ENTER para iniciar a simulação", True, RED)
    screen.blit(info_text, (WIDTH // 2 - info_text.get_width() // 2, 400))

    pygame.display.update()

# E - desenha a simulação na tela
def draw_simulation():
    screen.fill(WHITE)
    angle_rad = math.radians(angle)
    incline_length_px = 400
    incline_height_px = incline_length_px * math.sin(angle_rad)
    incline_width_px = incline_length_px * math.cos(angle_rad)
    base_x = 100
    base_y = 150

    # G1 - desenha o plano inclinado
    pygame.draw.polygon(screen, GRAY, [
        (base_x, base_y),
        (base_x + incline_width_px, base_y + incline_height_px),
        (base_x, base_y + incline_height_px)
    ])

    # G2 - desenha a rampa plana
    flat_start_x = base_x + incline_width_px
    flat_start_y = base_y + incline_height_px
    flat_length_px = (flat_length / length) * incline_length_px * 2

    pygame.draw.rect(screen, GRAY, (flat_start_x, flat_start_y - 15, flat_length_px, 30))

    # E - calcula posição do objeto
    if not on_flat:
        obj_x = base_x + (object_pos / length) * incline_width_px
        obj_y = base_y + (object_pos / length) * incline_height_px
    else:
        obj_x = flat_start_x + flat_pos * (flat_length_px / flat_length)
        obj_y = flat_start_y

    pygame.draw.circle(screen, RED, (int(obj_x), int(obj_y)), 15)

    # G1 - mostra informações físicas na tela
    weight = mass * g
    normal_force = weight * math.cos(angle_rad) if not on_flat else weight
    parallel_force = weight * math.sin(angle_rad) if not on_flat else 0
    friction_force = mu * normal_force

    info_x = WIDTH - 260
    info_y = 50
    screen.blit(font.render(f"Ângulo: {angle}°", True, BLACK), (info_x, info_y))
    screen.blit(font.render(f"Massa: {mass} kg", True, BLACK), (info_x, info_y + 30))
    screen.blit(font.render(f"Comprimento: {length} m", True, BLACK), (info_x, info_y + 60))
    screen.blit(font.render(f"Força Normal: {normal_force:.2f} N", True, BLACK), (info_x, info_y + 90))
    screen.blit(font.render(f"Força Paralela: {parallel_force:.2f} N", True, BLACK), (info_x, info_y + 120))
    screen.blit(font.render(f"Força de Atrito: {friction_force:.2f} N", True, BLACK), (info_x, info_y + 150))
    screen.blit(font.render(f"Velocidade: {velocity:.2f} m/s", True, BLACK), (info_x, info_y + 180))
    screen.blit(font.render(f"Tempo: {time:.2f} s", True, BLACK), (info_x, info_y + 210))

    if not simulation_running:
        screen.blit(font.render(f"Velocidade Final: {final_velocity:.2f} m/s", True, BLACK), (info_x, info_y + 240))

    # G2 - mostra instrução para reiniciar
    reset_text = font.render("Pressione SPACE para reiniciar", True, RED)
    screen.blit(reset_text, (WIDTH // 2 - reset_text.get_width() // 2, HEIGHT - 50))

    pygame.display.update()

# E - calcula a física da simulação
def calculate_physics(dt):
    global object_pos, velocity, time, final_velocity, simulation_running, on_flat, flat_pos

    if not on_flat:
        angle_rad = math.radians(angle)
        weight = mass * g
        normal_force = weight * math.cos(angle_rad)
        parallel_force = weight * math.sin(angle_rad)
        friction_force = mu * normal_force

        if velocity > 0 or parallel_force > friction_force:
            net_force = parallel_force - friction_force
        else:
            net_force = 0

        acceleration = net_force / mass
        velocity += acceleration * dt
        object_pos += velocity * dt

        if object_pos >= length:
            object_pos = length
            final_velocity = velocity
            on_flat = True
            flat_pos = 0

    else:
        weight = mass * g
        normal_force = weight 
        friction_force = mu * normal_force

        if velocity > 0:
            net_force = -friction_force
            acceleration = net_force / mass
            velocity += acceleration * dt
            if velocity < 0:
                velocity = 0
            flat_pos += velocity * dt
        else:
            velocity = 0
            simulation_running = False

    time += dt

# G1 - inicializa o loop principal do jogo
clock = pygame.time.Clock()
running = True

while running:
    dt = clock.tick(60) / 1000.0

    for event in pygame.event.get():
        if event.type == QUIT:
            running = False

        elif input_mode:
            if event.type == MOUSEBUTTONDOWN:
                for key, box in input_boxes.items():
                    if box["rect"].collidepoint(event.pos):
                        active_box = key
                        break

            elif event.type == KEYDOWN:
                if active_box:
                    if event.key == K_RETURN:
                        if all(input_boxes[k]["value"] for k in input_boxes):
                            try:
                                angle = float(input_boxes["angle"]["value"])
                                mass = float(input_boxes["mass"]["value"])
                                length = float(input_boxes["length"]["value"])

                                input_mode = False
                                simulation_running = True
                                object_pos = 0
                                velocity = 0
                                final_velocity = 0
                                time = 0
                                on_flat = False
                                flat_pos = 0
                                flat_length = length / 2

                            except ValueError:
                                pass
                    elif event.key == K_BACKSPACE:
                        input_boxes[active_box]["value"] = input_boxes[active_box]["value"][:-1]
                    else:
                        char = event.unicode
                        if char.isdigit() or char == '.' or (char == '-' and len(input_boxes[active_box]["value"]) == 0):
                            input_boxes[active_box]["value"] += char

        else:
            if event.type == KEYDOWN:
                if event.key == K_SPACE:
                    # G2 - reinicia a simulação
                    input_mode = True
                    simulation_running = False
                    object_pos = 0
                    velocity = 0
                    final_velocity = 0
                    time = 0
                    on_flat = False
                    flat_pos = 0
                    for k in input_boxes:
                        input_boxes[k]["value"] = ""
                    active_box = None

    if input_mode:
        draw_input_screen()
    else:
        if simulation_running:
            calculate_physics(dt)
        draw_simulation()

# E - encerra o programa
pygame.quit()
sys.exit()
