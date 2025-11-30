# SnakeGame_assebly_TASM
# Retro snake game 🐍 implementation using TASM 8086 Assembly
= = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = =
About the project:
A classic Snake game implemented entirely in 8086 Assembly, running in text‑mode (Mode 3) using BIOS interrupts and direct video memory access.
The game logic, rendering, input handling, and collision detection are all written manually using low‑level assembly instructions.
= = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = =
⚙️Features
-Fully working Snake movement (up, down, left, right)
-Snake grows after eating food (*) 
-Food is placed randomly using BIOS timer ticks
-Self‑collision detection
-Wall‑collision detection (instant GAME OVER)
-Smooth frame‑based movement with custom delay routine (55 milliseconds) 
-Draws directly to 0xB800 text video memory
-Uses INT 10h & INT 16h for screen and input control
-Clean screen redraw and proper tail‑erasing logic
= = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = =
🎮 Game Controls
Key	Action
↑	Move Up
↓	Move Down
←	Move Left
→	Move Right
ESC	Exit Game
= = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = =
🕹️ Some_Notes_About_Game_Logic 
1.Snake_Movement
-The snake cannot reverse direction into itself (e.g., cannot go from left → right immediately).
2.Food System
-Food position uses the BIOS timer (INT 1Ah) for randomness.
-Food is re‑generated until it does not overlap the snake’s body.
3.Collisions
-Based on checking simple coordinate comparison of head vs body segments.
-Wall hit → game_over
-Self‑collision → game_over
= = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = =
🛠️ How to Build & Run
1.Environment 
-GUI turbo assmebler (https://shorturl.at/Vtfi9)
2.Build and Run
2.1-Simply create a new file
2.2-Open Snake.asm file using TASM
2.3-press F9 shrtcut to assmeble,build,and run code  
ENJOY !!
= = = = = = = = = = = = = = = = = = = = = = = = 
👨‍💻 Authors
-Yousef Ashraf
-Yousef Sarhan
-Yousef Moustafa 
