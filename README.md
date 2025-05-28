import pygame
import sys
import math
from pygame.locals import *

pygame.init()

WIDTH, HEIGHT = 1500, 800
screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption('Simulação de Plano Inclinado com Rampa Plana')

WHITE = (255, 255, 255)
BLACK = (0, 0, 0)
RED = (255, 0, 0)
GRAY = (200, 200, 200)

font = pygame.font.SysFont('Arial', 20)
title_font = pygame.font.SysFont('Arial', 24, bold=True)

# G2 - variáveis iniciais da simulação
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

# G1 - variáveis para o plano horizontal (rampa)
flat_length = 0
flat_pos = 0
on_flat = False

# G1 - caixas de input para parâmetros
input_boxes = {
    "angle": {"label": "Ângulo (graus):", "value": "", "rect": pygame.Rect(300, 200, 200, 30)},
    "mass": {"label": "Massa (kg):", "value": "", "rect": pygame.Rect(300, 260, 200, 30)},
    "length": {"label": "Comprimento (m):", "value": "", "rect": pygame.Rect(300, 320, 200, 30)}
}
active_box = None

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

def draw_simulation():
    screen.fill(WHITE)
    angle_rad = math.radians(angle)
    incline_length_px = 400
    incline_height_px = incline_length_px * math.sin(angle_rad)
    incline_width_px = incline_length_px * math.cos(angle_rad)
    base_x = 100
    base_y = 150

    # E - Desenha o plano inclinado
    pygame.draw.polygon(screen, GRAY, [
        (base_x, base_y),
        (base_x + incline_width_px, base_y + incline_height_px),
        (base_x, base_y + incline_height_px)
    ])

    # G1 - Desenha a rampa plana
    flat_start_x = base_x + incline_width_px
    flat_start_y = base_y + incline_height_px
    flat_length_px = (flat_length / length) * incline_length_px * 2  

    pygame.draw.rect(screen, GRAY, (flat_start_x, flat_start_y - 15, flat_length_px, 30))

    # E - Desenha a bola na posição correta
    if not on_flat:
        obj_x = base_x + (object_pos / length) * incline_width_px
        obj_y = base_y + (object_pos / length) * incline_height_px
    else:
        obj_x = flat_start_x + flat_pos * (flat_length_px / flat_length)
        obj_y = flat_start_y

    pygame.draw.circle(screen, RED, (int(obj_x), int(obj_y)), 15)

    # G2 - Calcula forças no plano inclinado
    weight = mass * g
    normal_force_incline = weight * math.cos(angle_rad)
    parallel_force = weight * math.sin(angle_rad)
    friction_force_incline = mu * normal_force_incline

    # G2 - Calcula forças no plano horizontal
    normal_force_flat = weight
    friction_force_flat = mu * normal_force_flat

    info_x = WIDTH - 300
    info_y = 50

    # E - Informações gerais
    screen.blit(font.render(f"Ângulo: {angle}°", True, BLACK), (info_x, info_y))
    screen.blit(font.render(f"Massa: {mass} kg", True, BLACK), (info_x, info_y + 30))
    screen.blit(font.render(f"Comprimento: {length} m", True, BLACK), (info_x, info_y + 60))

    # G1 - Exibe forças do plano inclinado sempre
    screen.blit(font.render(f"[Inclinado] Força Normal: {normal_force_incline:.2f} N", True, BLACK), (info_x, info_y + 90))
    screen.blit(font.render(f"[Inclinado] Força Paralela: {parallel_force:.2f} N", True, BLACK), (info_x, info_y + 120))
    screen.blit(font.render(f"[Inclinado] Força de Atrito: {friction_force_incline:.2f} N", True, BLACK), (info_x, info_y + 150))

    # G1 - Exibe forças do plano horizontal sempre
    screen.blit(font.render(f"[Horizontal] Força Normal: {normal_force_flat:.2f} N", True, BLACK), (info_x, info_y + 180))
    screen.blit(font.render(f"[Horizontal] Força de Atrito: {friction_force_flat:.2f} N", True, BLACK), (info_x, info_y + 210))

    # E - Velocidade e tempo
    screen.blit(font.render(f"Velocidade: {velocity:.2f} m/s", True, BLACK), (info_x, info_y + 240))
    screen.blit(font.render(f"Tempo: {time:.2f} s", True, BLACK), (info_x, info_y + 270))

    if not simulation_running:
        screen.blit(font.render(f"Velocidade Final: {final_velocity:.2f} m/s", True, BLACK), (info_x, info_y + 300))

    # G2 - Mensagem para reiniciar
    reset_text = font.render("Pressione SPACE para reiniciar", True, RED)
    screen.blit(reset_text, (WIDTH // 2 - reset_text.get_width() // 2, HEIGHT - 50))

    pygame.display.update()

def calculate_physics(dt):
    global object_pos, velocity, time, final_velocity, simulation_running, on_flat, flat_pos

    angle_rad = math.radians(angle)
    weight = mass * g

    if not on_flat:
        # G1 - Forças e movimento no plano inclinado
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
            flat_pos = 0  # inicia movimento na rampa plana

    else:
        # E - Movimento na rampa plana com atrito
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

                                # G2 - inicia simulação com valores
                                input_mode = False
                                simulation_running = True
                                object_pos = 0
                                velocity = 0
                                final_velocity = 0
                                time = 0
                                on_flat = False
                                flat_pos = 0
                                flat_length = length  # comprimento da rampa plana

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
                    # E - Resetar tudo
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

pygame.quit()
sys.exit()
