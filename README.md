# PROJECT-TRIV5
# world.py
from ursina import *

def create_world():
    # Ground plane
    Entity(model='plane', scale=(100, 1, 100), color=color.dark_gray, collider='box')
    # Simple Sky
    Sky()
    # Adding a light for better visibility
    DirectionalLight(y=2, z=3, shadows=True)
# ai_brain.py
import random
from ursina import distance

class AIBot:
    def __init__(self):
        self.type = random.choice(["AGGRESSIVE", "DEFENSIVE"])

    def update(self, player, enemy):
        dist = distance(player.position, enemy.position)

        if self.type == "AGGRESSIVE":
            if dist > 3:
                return "CHASE"
            return "ATTACK"

        elif self.type == "DEFENSIVE":
            if dist < 5:
                return "RETREAT"
            return "CHASE"
# enemy.py
from ursina import *
from ai_brain import AIBot
import random

class Enemy(Entity):
    def __init__(self):
        super().__init__(
            model='cube', 
            color=color.red, 
            scale=(1, 2, 1), 
            position=(random.randint(-20, 20), 1, random.randint(-20, 20)),
            collider='box'
        )
        self.health = 50
        self.brain = AIBot()

    def update_logic(self, player):
        action = self.brain.update(player, self)

        if action == "CHASE":
            self.look_at(player)
            self.rotation_x = 0  # Keep them upright
            self.position += self.forward * time.dt * 3

        elif action == "RETREAT":
            self.look_at(player)
            self.rotation_x = 0
            self.position -= self.forward * time.dt * 3

        # Damage player on contact
        if distance(self.position, player.position) < 1.5:
            player.health -= 10 * time.dt
            
# player.py
from ursina import *

class Player(Entity):
    def __init__(self):
        super().__init__(
            model='cube', 
            color=color.blue, 
            scale=(1, 2, 1), 
            position=(0, 1, 0), 
            collider='box'
        )
        self.speed = 8
        self.health = 100
        
        # Setup Camera
        camera.parent = self
        camera.position = (0, 1.5, -6)
        camera.rotation_x = 10
        mouse.visible = False

    def update(self):
        # Basic movement
        direction = Vec3(
            held_keys['d'] - held_keys['a'],
            0,
            held_keys['w'] - held_keys['s']
        ).normalized()
        
        self.position += direction * time.dt * self.speed

# ui.py      
from ursina import *

class UI:
    def __init__(self, player):
        self.player = player
        self.health_bar_bg = Entity(parent=camera.ui, model='quad', color=color.black66, scale=(0.41, 0.03), position=(-0.5, 0.4))
        self.health_bar = Entity(parent=camera.ui, model='quad', color=color.green, scale=(0.4, 0.02), position=(-0.5, 0.4))
        self.health_text = Text(text="HP", position=(-0.7, 0.45), scale=1.2)

    def update_ui(self):
        self.health_bar.scale_x = 0.4 * (max(0, self.player.health) / 100)
        # Change bar color if health is low
        if self.player.health < 30:
            self.health_bar.color = color.red
            

# main.py
from ursina import *
from player import Player
from enemy import Enemy
from ui import UI
from world import create_world

app = Ursina()

# Initialize Game
create_world()
player = Player()
ui = UI(player)
enemies = [Enemy() for _ in range(8)]
bullets = []

def input(key):
    if key == 'left mouse down':
        # Create bullet
        bullet = Entity(model='sphere', scale=0.3, color=color.yellow, 
                        position=player.position + Vec3(0,1.5,0) + player.forward,
                        collider='sphere')
        bullet.direction = player.forward
        bullets.append(bullet)
        destroy(bullet, delay=2) # Auto-cleanup after 2 seconds

def update():
    player.update()
    ui.update_ui()

    # Update Enemies
    for e in enemies:
        e.update_logic(player)

    # Update Bullets & Collisions
    for b in bullets[:]:
        b.position += b.direction * time.dt * 40
        
        # Check collision with enemies
        hit_info = b.intersects()
        if hit_info.hit:
            if isinstance(hit_info.entity, Enemy):
                hit_info.entity.health -= 20
                if hit_info.entity.health <= 0:
                    enemies.remove(hit_info.entity)
                    destroy(hit_info.entity)
                
                if b in bullets:
                    bullets.remove(b)
                    destroy(b)

    # Game Over Check
    if player.health <= 0:
        Text(text="GAME OVER", origin=(0,0), scale=3, color=color.red)
        application.pause()

app.run()










            
