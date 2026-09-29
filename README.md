# Terminal-snake-retro-engine
The terminal-based Snake game architecture is structured around a decoupled Model-View-Controller (MVC) pattern that splits low-level platform input handling, core arcade physics, and text-buffer frame rendering.
import os
import sys
import time
import random
import threading
from typing import List, Tuple, Dict, Any, Optional

# =====================================================================
# 1. PLATFORM CONFIGURATION & WINDOWS/POSIX COMPATIBILITY LAYER
# =====================================================================

if os.name == 'nt':
    import msvcrt
else:
    import tty
    import termios

class NonBlockingInput:
    """Captures keyboard events instantly without freezing execution frames."""
    def __init__(self):
        self.old_settings = None
        if os.name != 'nt':
            self.old_settings = termios.tcgetattr(sys.stdin)

    def get_char(self) -> Optional[str]:
        if os.name == 'nt':
            if msvcrt.kbhit():
                try:
                    ch = msvcrt.getch().decode('utf-8').lower()
                    return ch
                except UnicodeDecodeError:
                    return None
            return None
        else:
            try:
                tty.setraw(sys.stdin.fileno())
                if sys.stdin in select.select([sys.stdin], [], [], 0)[0]:
                    return sys.stdin.read(1).lower()
                return None
            except Exception:
                return None
            finally:
                if self.old_settings:
                    termios.tcsetattr(sys.stdin, termios.TCSADRAIN, self.old_settings)


# For non-Windows cross-platform input selection handling fallback
if os.name != 'nt':
    import select


# =====================================================================
# 2. CORE GAME STATE MANAGEMENT ENGINE
# =====================================================================

class SnakeEngine:
    """Manages coordinate vectors, physics checks, and score keeping."""
    def __init__(self, width: int = 20, height: int = 15):
        self.width = width
        self.height = height
        self.reset_game()

    def reset_game(self):
        # Start at the center boundary coordinate
        self.snake: List[Tuple[int, int]] = [
            (self.height // 2, self.width // 2),
            (self.height // 2, (self.width // 2) - 1),
            (self.height // 2, (self.width // 2) - 2)
        ]
        # Direction mapping vectors: (y, x)
        self.direction = (0, 1)  # Initially traveling Right
        self.food_position: Tuple[int, int] = (0, 0)
        self.score = 0
        self.game_over = False
        self.spawn_food()

    def spawn_food(self):
        """Generates item location ensuring it does not collide with the snake's body."""
        while True:
            y = random.randint(0, self.height - 1)
            x = random.randint(0, self.width - 1)
            if (y, x) not in self.snake:
                self.food_position = (y, x)
                break

    def change_direction(self, keystroke: str):
        """Maps keys while preventing the snake from folding back into itself."""
        # mapping controls: w=up, s=down, a=left, d=right
        if keystroke == 'w' and self.direction != (1, 0):
            self.direction = (-1, 0)
        elif keystroke == 's' and self.direction != (-1, 0):
            self.direction = (1, 0)
        elif keystroke == 'a' and self.direction != (0, 1):
            self.direction = (0, -1)
        elif keystroke == 'd' and self.direction != (0, -1):
            self.direction = (0, 1)

    def process_physics_tick(self) -> bool:
        """Calculates trajectory movements. Returns False if a fatal collision happens."""
        if self.game_over:
            return False

        head_y, head_x = self.snake[0]
        dir_y, dir_x = self.direction
        
        # Determine target tile
        new_head = (head_y + dir_y, head_x + dir_x)

        # Collision Check: Boundary Walls
        if not (0 <= new_head[0] < self.height and 0 <= new_head[1] < self.width):
            self.game_over = True
            return False

        # Collision Check: Own Tail
        if new_head in self.snake:
            self.game_over = True
            return False

        # Prepend new head coordinates
        self.snake.insert(0, new_head)

        # Logic Branch: Eating food objective vs normal step tracking
        if new_head == self.food_position:
            self.score += 10
            self.spawn_food()
        else:
            # Delete last cell segment to simulate standard fluid motion forward
            self.snake.pop()
            
        return True


# =====================================================================
# 3. RENDER ENGINE & EXECUTIVE GAME CONTROLLER
# =====================================================================

class TerminalGraphicsController:
    """Handles buffer clears and structures drawing elements cleanly in the terminal."""
    def __init__(self, engine: SnakeEngine):
        self.engine = engine
        self.input_handler = NonBlockingInput()

    def clear_screen(self):
        """Clears terminal screen efficiently across operating systems."""
        os.system('cls' if os.name == 'nt' else 'clear')

    def render_frame(self) -> str:
        """Generates standard text strings representing the graphical screen layout buffer."""
        buffer = []
        # Draw dynamic boundary roof
        buffer.append("╔" + "══" * self.engine.width + "╗\n")

        for y in range(self.engine.height):
            row = ["║"]
            for x in range(self.engine.width):
                if (y, x) == self.engine.snake[0]:
                    row.append("🐸")  # Snake Head rendering symbol
                elif (y, x) in self.engine.snake:
                    row.append("🟢")  # Body section rendering symbol
                elif (y, x) == self.engine.food_position:
                    row.append("🍎")  # Food item target symbol
                else:
                    row.append("  ")  # Empty landscape tile
            row.append("║\n")
            buffer.append("".join(row))

        # Draw structural base frame
        buffer.append("╚" + "══" * self.engine.width + "╝\n")
        buffer.append(f" SCORE: {self.engine.score} points | [W A S D]: Steer | [Q]: Quit Game\n")
        
        return "".join(buffer)

    def start_game_loop(self):
        """Main loop that links input, physical checks, frame rendering, and frame timing."""
        self.engine.reset_game()
        
        print("🎮 PREPARING TERMINAL CANVAS... PRESS ENTER TO START.")
        input()
        
        # Adaptive difficulty framework tracking speed variables
        base_tick_speed = 0.15 

        while not self.engine.game_over:
            # Ingest input events safely
            key = self.input_handler.get_char()
            if key == 'q':
                break
            if key:
                self.engine.change_direction(key)

            # Advance game physics calculations
            self.engine.process_physics_tick()

            # Render updated display outputs
            self.clear_screen()
            sys.stdout.write(self.render_frame())
            sys.stdout.flush()

            # Maintain stable update intervals
            time.sleep(base_tick_speed)

        # Terminate cleanly with a game-over interface screen
        self.clear_screen()
        print("\n" + "=" * 45)
        print("                 💀 GAME OVER 💀                 ")
        print(f"       FINAL SCORE ACQUIRED: {self.engine.score} POINTS")
        print("=" * 45 + "\n")


# =====================================================================
# 4. RUNTIME SYSTEM INTENT TRIGGER
# =====================================================================

if __name__ == "__main__":
    # Create the internal logic grid with a standard dimension mapping layout
    snake_game = SnakeEngine(width=22, height=16)
    
    # Initialize interface rendering container wrapper
    terminal_app = TerminalGraphicsController(snake_game)
    
    # Run application runtime pipeline safely
    try:
        terminal_app.start_game_loop()
    except KeyboardInterrupt:
        print("\nApplication closed via system intervention signal.")
