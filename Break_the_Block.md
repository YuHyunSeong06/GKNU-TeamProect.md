```c
#include "raylib.h"
#include <stdlib.h>
#include <time.h>
#include <stdio.h>
#include <math.h>

#define SCREEN_WIDTH 800
#define SCREEN_HEIGHT 600

typedef struct {
    Color color;
    int score;
} BlockColor;

BlockColor blockColors[100];

void InitGame(int difficulty, Rectangle* paddle, Vector2* ball, Vector2* speed, int* score, int* lives, double* startTime, int* blockCount, float* paddleSpeed) {
    switch (difficulty) {
    case 1:
        paddle->width = 80;
        *speed = (Vector2){ 6, -6 };
        *blockCount = 48;
        break;
    case 2:
        paddle->width = 100;
        *speed = (Vector2){ 5, -5 };
        *blockCount = 36;
        break;
    case 3:
        paddle->width = 120;
        *speed = (Vector2){ 4, -4 };
        *blockCount = 12;
        break;
    default:
        paddle->width = 100;
        *speed = (Vector2){ 5, -5 };
        *blockCount = 36;
        break;
    }

    *paddleSpeed = 8.0f;
    paddle->height = 20;
    paddle->x = SCREEN_WIDTH / 2 - paddle->width / 2;
    paddle->y = SCREEN_HEIGHT - 30;

    *ball = (Vector2){ SCREEN_WIDTH / 2, SCREEN_HEIGHT / 2 };
    *score = 0;
    *lives = 3;
    *startTime = GetTime();

    for (int i = 0; i < *blockCount; i++) {
        blockColors[i].score = 0;
        int colorIndex = rand() % 6;
        switch (colorIndex) {
        case 0: blockColors[i].color = RED; blockColors[i].score = 10; break;
        case 1: blockColors[i].color = ORANGE; blockColors[i].score = 7; break;
        case 2: blockColors[i].color = YELLOW; blockColors[i].score = 5; break;
        case 3: blockColors[i].color = GREEN; blockColors[i].score = 3; break;
        case 4: blockColors[i].color = BLUE; blockColors[i].score = 2; break;
        case 5: blockColors[i].color = PURPLE; blockColors[i].score = 1; break;
        }
    }
}

bool AreAllBlocksCleared(int blockCount) {
    for (int i = 0; i < blockCount; i++) {
        if (blockColors[i].score > 0) return false;
    }
    return true;
}

int main() {
    InitWindow(SCREEN_WIDTH, SCREEN_HEIGHT, "Block Breaker");
    SetTargetFPS(60);
    InitAudioDevice();

    srand((unsigned int)time(NULL));

    Rectangle paddle;
    Vector2 ball;
    Vector2 speed;
    int score = 0;
    int lives = 3;
    int gameState = 0;
    double startTime = 0;
    double totalTime = 0;
    int blockCount = 36;
    float paddleSpeed = 8.0f;
    bool flawless = true;

    Music bgm = LoadMusicStream("bgm.mp3");
    Sound hitSound = LoadSound("hit.wav");
    PlayMusicStream(bgm);

    while (!WindowShouldClose()) {
        if (gameState == 0) {
            BeginDrawing();
            ClearBackground(BLACK);
            DrawText("START PRESS KEY 1", SCREEN_WIDTH / 2 - 130, SCREEN_HEIGHT / 2 - 20, 20, WHITE);
            DrawText("QUIT PRESS KEY 0", SCREEN_WIDTH / 2 - 130, SCREEN_HEIGHT / 2 + 20, 20, WHITE);
            EndDrawing();

            if (IsKeyPressed(KEY_ONE)) {
                int difficulty = 0;
                while (difficulty < 1 || difficulty > 3) {
                    BeginDrawing();
                    ClearBackground(BLACK);
                    DrawText("Select Level:", SCREEN_WIDTH / 2 - 100, SCREEN_HEIGHT / 2 - 40, 20, WHITE);
                    DrawText("HARD: 1, MEDIUM: 2, EASY: 3", SCREEN_WIDTH / 2 - 100, SCREEN_HEIGHT / 2, 20, WHITE);
                    EndDrawing();

                    if (IsKeyPressed(KEY_ONE)) difficulty = 1;
                    if (IsKeyPressed(KEY_TWO)) difficulty = 2;
                    if (IsKeyPressed(KEY_THREE)) difficulty = 3;
                }

                InitGame(difficulty, &paddle, &ball, &speed, &score, &lives, &startTime, &blockCount, &paddleSpeed);
                flawless = true;
                gameState = 1;
            }
            if (IsKeyPressed(KEY_ZERO)) break;
        }

        if (gameState == 1) {
            UpdateMusicStream(bgm);

            if (IsKeyDown(KEY_LEFT)) paddle.x -= paddleSpeed;
            if (IsKeyDown(KEY_RIGHT)) paddle.x += paddleSpeed;

            ball.x += speed.x;
            ball.y += speed.y;

            if (ball.x < 0 || ball.x > SCREEN_WIDTH) speed.x *= -1;
            if (ball.y < 0) speed.y *= -1;

            if (ball.y > SCREEN_HEIGHT) {
                lives--;
                flawless = false;
                ball = (Vector2){ SCREEN_WIDTH / 2, SCREEN_HEIGHT / 2 };
                if (lives <= 0) gameState = 2;
            }

            if (CheckCollisionCircleRec(ball, 10, paddle)) speed.y *= -1;

            for (int i = 0; i < blockCount; i++) {
                if (blockColors[i].score > 0) {
                    Rectangle blockRect = { 20 + (i % 12) * 63, 50 + (i / 12) * 30, 60, 20 };
                    if (CheckCollisionCircleRec(ball, 10, blockRect)) {
                        Color blockColor = blockColors[i].color;

                        if (blockColor.r == RED.r && blockColor.g == RED.g && blockColor.b == RED.b) {
                            lives++;
                        }
                        if (blockColor.r == ORANGE.r && blockColor.g == ORANGE.g && blockColor.b == ORANGE.b) {
                            startTime += 30;
                        }
                        if (blockColor.r == YELLOW.r && blockColor.g == YELLOW.g && blockColor.b == YELLOW.b) {
                            startTime += 15;
                        }
                        if (blockColor.r == GREEN.r && blockColor.g == GREEN.g && blockColor.b == GREEN.b) {
                            startTime += 5;
                        }
                        if (blockColor.r == BLUE.r && blockColor.g == BLUE.g && blockColor.b == BLUE.b) {
                            if (speed.x > 0) speed.x += 0.01f; else speed.x -= 0.01f;
                            if (speed.y > 0) speed.y += 0.01f; else speed.y -= 0.01f;
                        }
                        if (blockColor.r == PURPLE.r && blockColor.g == PURPLE.g && blockColor.b == PURPLE.b) {
                            paddleSpeed += 0.1f;
                        }

                        score += blockColors[i].score;
                        blockColors[i].score = 0;
                        speed.y *= -1;
                        PlaySound(hitSound);
                        break;
                    }
                }
            }

            if (AreAllBlocksCleared(blockCount)) {
                totalTime = GetTime() - startTime;
                gameState = 3;
            }

            BeginDrawing();
            ClearBackground(BLACK);
            DrawRectangleRec(paddle, WHITE);
            DrawCircleV(ball, 10, WHITE);
            for (int i = 0; i < blockCount; i++) {
                if (blockColors[i].score > 0) {
                    DrawRectangle(20 + (i % 12) * 63, 50 + (i / 12) * 30, 60, 20, blockColors[i].color);
                }
            }

            DrawText(TextFormat("SCORE: %d", score), SCREEN_WIDTH - 150, 10, 20, WHITE);
            DrawText(TextFormat("LIFE: %d", lives), 10, 10, 20, WHITE);

            double elapsedTime = GetTime() - startTime;
            int minutes = (int)(elapsedTime / 60);
            int seconds = (int)fmod(elapsedTime, 60);
            DrawText(TextFormat("TIME: %02d:%02d", minutes, seconds), SCREEN_WIDTH - 150, SCREEN_HEIGHT - 30, 20, WHITE);
            EndDrawing();
        }

        else if (gameState == 2) {
            BeginDrawing();
            ClearBackground(BLACK);
            DrawText("GAME OVER!!! RESTART PRESS 'r' QUIT PRESS 'Esc'", SCREEN_WIDTH / 2 - 200, SCREEN_HEIGHT / 2, 20, WHITE);
            EndDrawing();

            if (IsKeyPressed(KEY_R)) gameState = 0;
            if (IsKeyPressed(KEY_ESCAPE)) break;
        }

        else if (gameState == 3) {
            BeginDrawing();
            ClearBackground(BLACK);
            DrawText("CONGRATULATIONS! YOU WIN!", SCREEN_WIDTH / 2 - 180, SCREEN_HEIGHT / 2 - 60, 20, WHITE);
            if (flawless && totalTime <= 60.0) {
                DrawText("FLAWLESS", SCREEN_WIDTH / 2 - 120, SCREEN_HEIGHT / 2 - 10, 40, GOLD);
            }
            DrawText("RESTART PRESS 'r'  |  QUIT PRESS 'Esc'", SCREEN_WIDTH / 2 - 200, SCREEN_HEIGHT / 2 + 40, 20, WHITE);
            EndDrawing();

            if (IsKeyPressed(KEY_R)) gameState = 0;
            if (IsKeyPressed(KEY_ESCAPE)) break;
        }
    }

    UnloadMusicStream(bgm);
    UnloadSound(hitSound);
    CloseAudioDevice();
    CloseWindow();

    return 0;
}
```
