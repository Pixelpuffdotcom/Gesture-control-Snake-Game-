# Gesture-control-Snake-Game-
#A Python-based "Snake Game" using Pygame, OpenCV, and Mediapipe. The snake is controlled via hand gestures, with food appearing randomly on the screen. The game tracks and displays the score, high score, and #features a firework effect for new high scores. The game ends upon collision with boundaries or itself.
import pygame
import random
import cv2
import mediapipe as mp
import math

# Initialize Pygame
pygame.init()

# Game window setup
WIDTH, HEIGHT = 1000, 800
screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption("Textured Snake Game")

# Colors
BLACK = (0, 0, 0)
WHITE = (255, 255, 255)
RED = (255, 0, 0)
SNAKE_HEAD_COLOR = (0, 100, 0)
SNAKE_BODY_COLOR = (0, 150, 0)
DARK_GREEN_PATCH = (0, 80, 0)
BACKGROUND_COLOR = (30, 50, 30)

# Game settings
snake_block = 20
base_speed = 5
HIGH_SCORE_FILE = "highscore.txt"

def initialize_camera():
    cap = cv2.VideoCapture(0)
    mp_hands = mp.solutions.hands
    hands = mp_hands.Hands(
        static_image_mode=False,
        max_num_hands=1,
        min_detection_confidence=0.8,
        min_tracking_confidence=0.8
    )
    return cap, hands

# Font setup
font = pygame.font.SysFont("Arial", 50)
clock = pygame.time.Clock()

# Texture patterns
body_texture = pygame.Surface((snake_block, snake_block))
body_texture.fill(SNAKE_BODY_COLOR)
pygame.draw.ellipse(body_texture, DARK_GREEN_PATCH, 
                  (3, 3, snake_block-6, snake_block-6))

head_texture = pygame.Surface((snake_block, snake_block), pygame.SRCALPHA)
pygame.draw.circle(head_texture, SNAKE_HEAD_COLOR, 
                 (snake_block//2, snake_block//2), snake_block//2)

def get_high_score():
    try:
        with open(HIGH_SCORE_FILE, 'r') as f:
            return int(f.read())
    except:
        return 0

def save_high_score(score):
    with open(HIGH_SCORE_FILE, 'w') as f:
        f.write(str(score))

def create_firework():
    return [{
        'pos': [WIDTH//4, HEIGHT//4],
        'color': (random.randint(100,255), random.randint(100,255), random.randint(100,255)),
        'vel': [random.uniform(-8,8), random.uniform(-25,5)],
        'lifetime': random.randint(70, 100),
        'age': 0
    } for _ in range(200)]

def update_firework(firework):
    for fw in firework:
        fw['vel'][1] += 0.4
        fw['pos'][0] += fw['vel'][0]
        fw['pos'][1] += fw['vel'][1]
        fw['age'] += 1
    return [fw for fw in firework if fw['age'] < fw['lifetime']]

def draw_firework(firework):
    for fw in firework:
        alpha = 255 * (1 - fw['age']/fw['lifetime'])
        size = int(8 * (1 - fw['age']/fw['lifetime']))
        s = pygame.Surface((16,16), pygame.SRCALPHA)
        pygame.draw.circle(s, (*fw['color'], int(alpha)), (8,8), size)
        screen.blit(s, (int(fw['pos'][0])-8, int(fw['pos'][1])-8))

def draw_snake(snake_list):
    for i, block in enumerate(snake_list):
        if i == len(snake_list) - 1:
            head_x, head_y = block
            center = (head_x + snake_block//2, head_y + snake_block//2)
            pygame.draw.circle(screen, SNAKE_HEAD_COLOR, center, snake_block//2)
            eye_radius = snake_block//8
            eye_offset_x = snake_block//4
            eye_offset_y = snake_block//4
            pygame.draw.circle(screen, WHITE, (center[0]-eye_offset_x, center[1]-eye_offset_y), eye_radius)
            pygame.draw.circle(screen, WHITE, (center[0]+eye_offset_x, center[1]-eye_offset_y), eye_radius)
            tongue_start = (center[0], center[1] + snake_block//2 - 2)
            tongue_end = (center[0], center[1] + snake_block//2 + 4)
            pygame.draw.line(screen, RED, tongue_start, tongue_end, 2)
        else:
            pygame.draw.ellipse(screen, SNAKE_BODY_COLOR, (block[0], block[1], snake_block, snake_block))

def draw_background():
    screen.fill(BACKGROUND_COLOR)
    # Add subtle grid texture
    grid_size = 50
    for x in range(0, WIDTH, grid_size):
        for y in range(0, HEIGHT, grid_size):
            if (x//grid_size + y//grid_size) % 2 == 0:
                pygame.draw.rect(screen, (25, 45, 25), 
                               (x, y, grid_size, grid_size), 1)

def spawn_food():
    return [
        round(random.randrange(0, WIDTH-snake_block)/snake_block)*snake_block,
        round(random.randrange(0, HEIGHT-snake_block)/snake_block)*snake_block
    ]

def game_loop():
    cap, hands = initialize_camera()
    high_score = get_high_score()
    game_over = False
    game_close = False
    fireworks = []

    # Snake initialization
    x, y = WIDTH//2, HEIGHT//2
    dx, dy = 0, 0
    snake = []
    snake_length = 1
    foods = [spawn_food() for _ in range(3)]

    # Gesture control variables
    prev_direction = None
    direction_lock = False

    while not game_over:
        ret, frame = cap.read()
        if ret:
            rgb_frame = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
            results = hands.process(rgb_frame)
            
            if results.multi_hand_landmarks:
                hand = results.multi_hand_landmarks[0]
                wrist = hand.landmark[mp.solutions.hands.HandLandmark.WRIST]
                index_tip = hand.landmark[mp.solutions.hands.HandLandmark.INDEX_FINGER_TIP]
                
                dir_x = wrist.x - index_tip.x
                dir_y = index_tip.y - wrist.y
                
                vector_length = math.sqrt(dir_x**2 + dir_y**2)
                
                if vector_length > 0.1:
                    norm_x = dir_x / vector_length
                    norm_y = dir_y / vector_length
                    
                    angle = math.degrees(math.atan2(norm_y, norm_x))
                    
                    if -45 <= angle < 45:    # Right
                        new_dir = 'right'
                    elif 45 <= angle < 135:  # Down
                        new_dir = 'down'
                    elif -135 <= angle < -45: # Up
                        new_dir = 'up'
                    else:                    # Left
                        new_dir = 'left'

                    if new_dir != prev_direction and not direction_lock:
                        if new_dir == 'left' and dx == 0:
                            dx, dy = -snake_block, 0
                        elif new_dir == 'right' and dx == 0:
                            dx, dy = snake_block, 0
                        elif new_dir == 'up' and dy == 0:
                            dx, dy = 0, -snake_block
                        elif new_dir == 'down' and dy == 0:
                            dx, dy = 0, snake_block
                        prev_direction = new_dir
                        direction_lock = True

        direction_lock = False

        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                game_over = True

        # Boundary check
        if x < 0 or x >= WIDTH or y < 0 or y >= HEIGHT:
            game_close = True

        # Self-collision check
        for segment in snake[:-1]:
            if segment == [x, y]:
                game_close = True

        if game_close:
            current_score = snake_length - 1
            new_high = False
            if current_score > high_score:
                save_high_score(current_score)
                fireworks = create_firework()
                high_score = current_score
                new_high = True
            
            while game_close:
                draw_background()
                if fireworks:
                    fireworks = update_firework(fireworks)
                    draw_firework(fireworks)
                
                # Game over text
                screen.blit(font.render("NEW HIGH SCORE!" if new_high else "GAME OVER", 
                                      True, RED), (WIDTH//4, HEIGHT//3))
                screen.blit(font.render(f"Score: {current_score}", True, WHITE), 
                          (WIDTH//3, HEIGHT//2))
                screen.blit(font.render("High Score: " + str(high_score), True, WHITE),
                          (WIDTH//3, HEIGHT//2 + 50))
                screen.blit(font.render("Press C to Play Again / Q to Quit", True, WHITE), 
                          (WIDTH//4, HEIGHT//1.5))
                
                pygame.display.update()

                for event in pygame.event.get():
                    if event.type == pygame.QUIT:
                        game_over = True
                        game_close = False
                    if event.type == pygame.KEYDOWN:
                        if event.key == pygame.K_q:
                            game_over = True
                            game_close = False
                        if event.key == pygame.K_c:
                            cap.release()
                            hands.close()
                            game_loop()
                            return

        # Update snake position
        x += dx
        y += dy
        
        # Drawing
        draw_background()

        # Food handling
        for food in foods:
            pygame.draw.circle(screen, RED, (food[0]+10, food[1]+10), 10)
            if x == food[0] and y == food[1]:
                foods.remove(food)
                foods.append(spawn_food())
                snake_length += 1

        # Snake body management
        snake.append([x, y])
        if len(snake) > snake_length:
            del snake[0]

        draw_snake(snake)
        pygame.display.update()
        clock.tick(base_speed + snake_length//2)

    cap.release()
    hands.close()
    pygame.quit()

if __name__ == "__main__":
    while True:
        game_loop()
