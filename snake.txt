DATAS SEGMENT
    MAXLEN  EQU 100 ; const maxlen
    snake_x    DB MAXLEN DUP(0) ; x cooridnates of snake pixles
    snake_y    DB MAXLEN DUP(0) ; y cooridnates of snake pixles
    snake_len  DB 4 ; starting length for snake
    direction  DB 2 ; snake directed to left by default
    ; up = 0 
    ; down = 1
    ; left = 2
    ; right = 3
    food_x     DB 50  ; starting X coordinate for food
    food_y     DB 12  ; starting  y coordinate for food
    ATTR_SNAKE DB 0Ah ; snake color (currently light green)
    ATTR_FOOD  DB 0Ch ; food color (currently light red)
    ATTR_BG    DB 07h ; background color (currently light grey)
    new_head_x DB 0   ; new X cooridnate for snake head
    new_head_y DB 0   ; new Y cooridnate for snake head
    tail_x     DB 0   ; starting x coordinate for tail
    tail_y     DB 0   ; starting y coordinate for tail
    msg_gameover DB 'GAME OVER','$'  ;  

DATAS ENDS

    
    
CODES SEGMENT
ASSUME CS:CODES, DS:DATAS ; Determine both data segment and code segment offsets

START:
    mov ax,  DATAS ; ds can only read from another register
    mov ds , ax 
    ; set text mode 3 (80x25 text, 16 colors)
    mov ax, 3
    int 10h

    ; starting values for snake pixels (x,y) coords 
    mov  [snake_len], 4
    mov  [snake_x + 0], 38
    mov [snake_x + 1], 39
    mov [snake_x + 2], 40
    mov [snake_x + 3], 41
    mov [snake_y + 0], 12
    mov [snake_y + 1], 12
    mov [snake_y + 2], 12
    mov [snake_y + 3], 12   
   ; intit food  
    mov [food_x], 50
    mov [food_y], 10
    ; call procedures from our code
    call clear_screen
    call draw_food
    call draw_snake_all

game_loop:
    call delay_tick

    ; keyboard handling
    mov ah, 1
    int 16h ; read user input key , store at al
    jz .no_key   ; continue game if no key pressed

    mov ah, 0
    int 16h               
    ; if esc pressed terminate game 
    cmp al, 1Bh ; 1bh = esc ascii val
    je .do_exit_now

    ; read current direction to avoid reversing into self
    mov dl, [direction]

    ; Up (scan 48h)
    cmp ah, 48h
    je .set_up
    ; Down (scan 50h)
    cmp ah, 50h
    je .set_down
    ; Left (scan 4Bh)
    cmp ah, 4Bh
    je .set_left
    ; Right (scan 4Dh)
    cmp ah, 4Dh
    je .set_right

    jmp .no_key
    ; ingore key if pressed in direction of movement (ssnake)
.set_up:
    cmp dl, 1            ; if current direction = DOWN (1) -> ignore
    je .no_key
    mov [direction], 0
    jmp .no_key

.set_down:
    cmp dl, 0            ; if current direction = UP (0) -> ignore
    je .no_key
    mov [direction], 1
    jmp .no_key

.set_left:
    cmp dl, 3            ; if current direction = RIGHT (3) -> ignore
    je .no_key
    mov [direction], 2
    jmp .no_key

.set_right:
    cmp dl, 2            ; if current direction = LEFT (2) -> ignore
    je .no_key
    mov [direction], 3
    jmp .no_key

.no_key:
    call compute_new_head
    call move_snake
    call check_self_collision
    call draw_food
    jmp game_loop

.do_exit_now:
    mov ax, 4C00h
    int 21h
    ; Exit the program 

; ----------------
compute_new_head PROC
    push ax bx cx dx si di ; save registers state coz it will change during proc

    mov al, [snake_x] ; col
    mov bl, al
    mov al, [snake_y] ; row
    mov bh, al

    mov al, [direction]
    cmp al, 0
    je cnh_up
    cmp al, 1
    je cnh_down
    cmp al, 2
    je cnh_left
    cmp al, 3
    je cnh_right
    jmp cnh_store

    ; cnh  = Compute New Head
    
    ; y coords
cnh_up:    dec bh
           jmp cnh_store
cnh_down:  inc bh
           jmp cnh_store
    ; x coords
cnh_left:  dec bl
           jmp cnh_store
cnh_right: inc bl

cnh_store:
    ; check wall hit on right or left
    cmp bl, 80
    je cnh_hit_wall
    cmp bl, 0FFH
    je cnh_hit_wall 

    ;check wall hit on top or bottom
    cmp bh, 25
    je cnh_hit_wall
    cmp bh, 0FFH
    je cnh_hit_wall
    jmp cnh_ok
    
    ; hit wall -> game over
cnh_hit_wall:
    call game_over

cnh_ok:
    mov [new_head_x], bl
    mov [new_head_y], bh


    pop di si dx cx bx ax
    ret
compute_new_head ENDP

move_snake PROC
    push ax bx cx dx si di bp

    mov cx, 0
    mov cl, [snake_len]
    ; if length <= 1 -> handle inline (avoid long conditional jump)
    cmp cx, 1
    jg ms_continue

    ; inline for length==0 or 1: just store head and draw
    mov al, [new_head_x]
    lea bx, snake_x
    mov [bx], al
    mov al, [new_head_y]
    lea bx, snake_y
    mov [bx], al
    ; draw head
    lea bx, snake_x
    mov dl, [bx + 0]
    lea bx, snake_y
    mov dh, [bx + 0]
    mov al, 'O'
    mov bl, [ATTR_SNAKE]
    call WriteCell
    pop bp di si dx cx bx ax
    ret
    ; ms : move snake
ms_continue:
    ; compute tail index = length - 1 in SI
    mov si, cx
    dec si

    ; save tail coords
    lea bx, snake_x
    mov dl, [bx + si]
    lea bx, snake_y
    mov dh, [bx + si]
    mov [tail_x], dl
    mov [tail_y], dh

    ; shift: for idx = length-1 downto 1
    mov di, cx
    dec di    ; di = length-1

ms_shift_loop:
    cmp di, 0
    je ms_shift_done

    ; use SI as source (src) = di-1 and DI as destination (dst) = di
    mov si, di
    dec si            ; SI = source index (di-1)

    ; copy x: snake_x[di] = snake_x[si]
    lea bx, snake_x
    mov al, [bx + si]
    mov [bx + di], al

    ; copy y: snake_y[di] = snake_y[si]
    lea bx, snake_y
    mov al, [bx + si]
    mov [bx + di], al

    dec di
    jmp ms_shift_loop

ms_shift_done:
    ; store new head into index 0
    mov al, [new_head_x]
    lea bx, snake_x
    mov [bx + 0], al

    mov al, [new_head_y]
    lea bx, snake_y
    mov [bx + 0], al
    ; check eat
    mov al, [new_head_x]
    cmp al, [food_x]
    jne ms_not_eat
    mov al, [new_head_y]
    cmp al, [food_y]
    jne ms_not_eat
    call clear_screen
    inc byte ptr [snake_len]
    call place_food
    jmp ms_draw_head

ms_not_eat:
    mov dl, [tail_x]
    mov dh, [tail_y]
    mov al, ' '
    mov bl, [ATTR_BG]
    call WriteCell

ms_draw_head:
    lea bx, snake_x
    mov dl, [bx + 0]
    lea bx, snake_y
    mov dh, [bx + 0]
    mov al, 'O'
    mov bl, [ATTR_SNAKE]
    call WriteCell

    pop bp di si dx cx bx ax
    ret
move_snake ENDP

place_food PROC
    push ax
    push bx
    push cx
    push dx
    push si
    push di

.try_again:
    ; random column (0..79) using DL from BIOS tick
    mov ah, 0
    int 1Ah              ; BIOS tick in DX (we use DL)
    mov al, dl
    mov ah, 0           ; AX = DL (0..255)
    mov bl, 80
    div bl               ; AL = quotient, AH = remainder (0..79)
    mov [food_x], ah     ; store remainder -> column 0..79

    ; random row (0..24) using another tick call
    mov ah, 0
    int 1Ah
    mov al, dl
    mov ah, 0           ; AX = DL
    mov bl, 25
    div bl               ; AH = remainder 0..24
    mov [food_y], ah     ; store remainder -> row 0..24

    ; --- ensure food not placed on snake ---
    mov cx, 0
    mov cl, [snake_len]  ; CX = snake_len (as byte in CL)
    mov si, 0

.chkloop:
    cmp si, cx
    jge .placed          ; checked all segments -> ok
    lea bx, snake_x
    mov al, [bx + si]    ; al = snake_x[si]
    cmp al, [food_x]
    jne .nextchk
    lea bx, snake_y
    mov al, [bx + si]    ; al = snake_y[si]
    cmp al, [food_y]
    je .try_again        ; collision -> pick new food
.nextchk:
    inc si
    jmp .chkloop

.placed:
    pop di
    pop si
    pop dx
    pop cx
    pop bx
    pop ax
    ret
place_food ENDP

check_self_collision PROC
    push ax bx cx dx si di
    lea bx, snake_x ; put offset of snake_x[0] in bx accroding to ds 
    mov al, [bx]
    ; store snake[0] , al 
    lea bx, snake_y
    mov ah, [bx]
    mov cx, 0
    mov cl, [snake_len] ; read last element of snake_x
    mov si, 1
    ; csc : check self collision
csc_loop:
    cmp si, cx
    jge csc_done
    lea bx, snake_x
    mov dl, [bx + si]; store the element which index is si in dl 
    cmp dl, al ; check of head crashed into any part of snake body (x coord)
    jne csc_next
    lea bx, snake_y
    mov dh, [bx + si]
    cmp dh, ah ; check of head crashed into any part of snake body (y coord)
    jne csc_next
    call game_over ; self collision happened
csc_next:
    inc si
    jmp csc_loop ; back to start of loop 
csc_done:
    pop di si dx cx bx ax
    ret
check_self_collision ENDP

; -----------------------
draw_food PROC
    push ax bx cx dx si di
    mov dl, [food_x]
    mov dh, [food_y]
    mov al, '*'
    mov bl, [ATTR_FOOD]
    call WriteCell
    pop di si dx cx bx ax
    ret
draw_food ENDP

draw_snake_all PROC
    push ax bx cx dx si di
    mov cx, 0
    mov cl, [snake_len]
    mov si, 0
ds_loop:
    cmp si, cx
    jge ds_done
    lea bx, snake_x
    mov dl, [bx + si]
    lea bx, snake_y
    mov dh, [bx + si]
    mov al, 'O'
    mov bl, [ATTR_SNAKE]
    call WriteCell
    inc si
    jmp ds_loop
ds_done:
    pop di si dx cx bx ax
    ret
draw_snake_all ENDP

; AL = char, BL = attr, DH = row, DL = col
WriteCell PROC
    push cx dx si di es ax bx

    ; compute offset = (row*80 + col) * 2
    mov ax, 0
    mov al, dh      ; AL = row
    mov bx, ax      ; BX = row
    shl ax, 6       ; AX = row*64
    mov cx, bx
    shl cx, 4       ; CX = row*16
    add ax, cx      ; AX = row*80

    mov bl, dl      ; BL = col
    mov bh, 0
    add ax, bx      ; AX = row*80 + col
    shl ax, 1       ; *2 -> byte offset
    mov di, ax

    mov ax, 0B800h
    mov es, ax
    pop bx ax

    ; write char and attribute to video memory
    mov  es:[di], al
    mov  es:[di+1], bl

    pop es di si dx cx
    ret
WriteCell ENDP

clear_screen PROC
    push ax bx cx dx si di es
    mov ax,0B800h
    mov es,ax
    mov di,0
    mov cx, 80*25
    mov al, ' '
    mov ah, [ATTR_BG]
cs_loop:
    mov es:[di], al
    mov es:[di+1], ah
    add di, 2
    loop cs_loop
    pop es di si dx cx bx ax
    ret
clear_screen ENDP


  ;  55ms delay due to value at cx 
delay_tick PROC
    push ax bx cx dx si di
    mov ah,0
    int 1Ah ; get current time , store it also in cx:dx ( it's a big number )
    mov bx, dx
dt_wait:
    mov ah,0
    int 1Ah
    cmp dx, bx
    je dt_wait
    mov cx, 0C000h ; 55ms delay
dt_busy:
    loop dt_busy ; check if cx is zero , if zero don't loop
    pop di si dx cx bx ax
    ret
delay_tick ENDP

game_over PROC
    lea dx ,msg_gameover
    mov ah ,9 
    int 21h ; check dx stored value , go to this address , then print till $ 
    mov ax,4C00h
    int 21h
game_over ENDP

CODES ENDS
END START
