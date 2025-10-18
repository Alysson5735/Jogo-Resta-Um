#include "raylib.h"
#include <stdio.h>
#include <stdbool.h>
#include <math.h>

#define BOARD_SIZE 7
#define R 40
#define SPACING 95
#define SCREEN_SIZE 800

typedef struct {
    int x, y;
    bool state;
    bool valid;
} Part;

typedef struct {
    int X, Y;
    int boardX, boardY;
    bool state;
} Selection;

Part board[BOARD_SIZE][BOARD_SIZE];
int movimentos = 0;
bool jogoFinalizado = false;

int offsetX, offsetY;
Selection origem = {0};
bool aguardandoDestino = false;

double tempoInicio = 0;

//------------------------ FUNÇÕES ------------------------

void gerar_tabuleiro() {
    movimentos = 0;
    jogoFinalizado = false;
    aguardandoDestino = false;
    origem = (Selection){0};
    tempoInicio = GetTime();

    for(int i=0;i<BOARD_SIZE;i++){
        for(int j=0;j<BOARD_SIZE;j++){
            board[i][j].x = i;
            board[i][j].y = j;
            board[i][j].state = false;
            board[i][j].valid = false;
        }
    }

    // Posições válidas (cruz central)
    for(int i=0;i<BOARD_SIZE;i++){
        for(int j=0;j<BOARD_SIZE;j++){
            if((i>=2 && i<=4) || (j>=2 && j<=4)){
                board[i][j].valid = true;
                board[i][j].state = true;
            }
        }
    }

    board[3][3].state = false; // centro vazio
}

int to_screen_x(int i) { return i*SPACING + offsetX; }
int to_screen_y(int j) { return j*SPACING + offsetY; }

bool movimento_valido(int x1, int y1, int x2, int y2){
    if(!board[x1][y1].state || !board[x2][y2].valid || board[x2][y2].state) return false;
    if(x1==x2 && abs(y1-y2)==2){
        int midY = (y1+y2)/2;
        return board[x1][midY].state;
    }
    if(y1==y2 && abs(x1-x2)==2){
        int midX = (x1+x2)/2;
        return board[midX][y1].state;
    }
    return false;
}

bool podeMover(int x, int y){
    if(!board[x][y].state) return false;
    int dir[4][2] = {{2,0},{-2,0},{0,2},{0,-2}};
    for(int i=0;i<4;i++){
        int nx = x+dir[i][0];
        int ny = y+dir[i][1];
        if(nx>=0 && nx<BOARD_SIZE && ny>=0 && ny<BOARD_SIZE){
            if(movimento_valido(x,y,nx,ny)) return true;
        }
    }
    return false;
}

int contar_pecas(){
    int total=0;
    for(int i=0;i<BOARD_SIZE;i++)
        for(int j=0;j<BOARD_SIZE;j++)
            if(board[i][j].valid && board[i][j].state) total++;
    return total;
}

//------------------------ MAIN ------------------------

int main(){
    InitWindow(SCREEN_SIZE, SCREEN_SIZE, "Resta Um");
    SetTargetFPS(60);

    // Calcular offsets para centralizar tabuleiro
    offsetX = (SCREEN_SIZE - (BOARD_SIZE-1)*SPACING)/2;
    offsetY = (SCREEN_SIZE - (BOARD_SIZE-1)*SPACING)/2;

    gerar_tabuleiro();

    while(!WindowShouldClose()){
        Vector2 mouse = GetMousePosition();

        // Botão Reset
        Rectangle botaoReset = { SCREEN_SIZE - 150, 30, 120, 40 };
        bool sobreBotao = CheckCollisionPointRec(mouse, botaoReset);

        // Clicar no botão Reset
        if(IsMouseButtonPressed(MOUSE_LEFT_BUTTON) && sobreBotao){
            gerar_tabuleiro();
        }

        // Seleção e movimento das bolinhas
        if(IsMouseButtonPressed(MOUSE_LEFT_BUTTON) && !sobreBotao){
            for(int i=0;i<BOARD_SIZE;i++){
                for(int j=0;j<BOARD_SIZE;j++){
                    if(!board[i][j].valid) continue;
                    int px = to_screen_x(i);
                    int py = to_screen_y(j);
                    float dist = sqrtf((px - mouse.x)*(px - mouse.x) + (py - mouse.y)*(py - mouse.y));

                    if(dist < R){
                        if(!aguardandoDestino && board[i][j].state){
                            origem = (Selection){px, py, i, j, true};
                            aguardandoDestino = true;
                        }
                        else if(aguardandoDestino){
                            if(movimento_valido(origem.boardX, origem.boardY, i, j)){
                                int meioX = (origem.boardX + i)/2;
                                int meioY = (origem.boardY + j)/2;
                                board[origem.boardX][origem.boardY].state = false;
                                board[meioX][meioY].state = false;
                                board[i][j].state = true;
                                movimentos++;
                            }
                            aguardandoDestino = false;
                            origem.state = false;
                        }
                    }
                }
            }
        }

        // Verificar vitória (1 peça restante no centro)
        if(!jogoFinalizado && contar_pecas() == 1 && board[3][3].state){
            jogoFinalizado = true;
        }

        double tempoDecorrido = GetTime() - tempoInicio;

        // DRAW
        BeginDrawing();
        ClearBackground((Color){210, 190, 150, 255});

        DrawText("Resta Um", 320, 20, 30, DARKBROWN);
        DrawText(TextFormat("Movimentos: %d", movimentos), 20, 60, 20, BROWN);
        DrawText(TextFormat("Restam: %d", contar_pecas()), 20, 90, 20, BROWN);
        DrawText(TextFormat("Tempo: %.0f s", tempoDecorrido), 20, 120, 20, BROWN);

        // Botão Reset
        DrawRectangleRec(botaoReset, sobreBotao ? SKYBLUE : BLUE);
        DrawText("Reset", botaoReset.x + 25, botaoReset.y + 10, 20, RAYWHITE);

        // Mensagem de vitória
        if(jogoFinalizado)
            DrawText(TextFormat("Parabéns! Você venceu em %.0f s", tempoDecorrido), 120, 160, 20, DARKGREEN);

        // Desenhar tabuleiro
        for(int i=0;i<BOARD_SIZE;i++){
            for(int j=0;j<BOARD_SIZE;j++){
                if(!board[i][j].valid) continue;

                Vector2 pos = { to_screen_x(i), to_screen_y(j) };
                Color fill = board[i][j].state ? DARKBLUE : GRAY;
                if(i==3 && j==3 && !board[i][j].state) fill = GRAY; // centro vazio
                Color border = BLANK;
                float borderWidth = 6.0f;

                if(CheckCollisionPointCircle(mouse, pos, R)){
                    if(board[i][j].state){
                        if(podeMover(i,j)) border = GREEN;
                        else border = RED;
                    }
                }

                if(origem.state && origem.boardX == i && origem.boardY == j)
                    border = SKYBLUE;

                DrawCircleV(pos, R, fill);
                if(border.a > 0)
                    DrawRing(pos, R - borderWidth/2, R + borderWidth/2, 0, 360, 64, border);
            }
        }

        EndDrawing();
    }

    CloseWindow();
    return 0;
}

