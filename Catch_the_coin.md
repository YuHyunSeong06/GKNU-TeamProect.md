# Catch The Coin
### 2025.05.13 GKNU C++ 프로그래밍 과제
#### 중앙 원을 클릭해서 움직여 랜덤으로 생성되는 코인을 수집하는 게임
#### 코인 색상별로 점수와 사라지는 시간, 능력등이 다양하다
```cpp
#include "raylib.h"
#include <vector>
#include <cmath>

enum class CoinType {
    Red,
    Pink,
    Orange,
    Yellow,
    LightGreen,
    Green,
    Blue,
    Purple,
    White,
    COUNT
};

class Coin {
public:
    Vector2 pos;
    float radius;
    CoinType type;
    Color color;
    float lifeTime;
    int score;
    bool active;
    float spawnTime;

    Coin(Vector2 p, CoinType t, float now)
        : pos(p), radius(15.0f), type(t), active(true), spawnTime(now)
    {
        switch (type) {
            case CoinType::Red:
                color = RED; lifeTime = 0.35f; score = 50; break;
            case CoinType::Pink:
                color = PINK; lifeTime = 0.37f; score = 40; break;
            case CoinType::Orange:
                color = ORANGE; lifeTime = 0.40f; score = 30; break;
            case CoinType::Yellow:
                color = YELLOW; lifeTime = 0.42f; score = 25; break;
            case CoinType::LightGreen:
                color = LIME; lifeTime = 0.44f; score = 20; break;
            case CoinType::Green:
                color = GREEN; lifeTime = 0.45f; score = 15; break;
            case CoinType::Blue:
                color = BLUE; lifeTime = 0.46f; score = 10; break;
            case CoinType::Purple:
                color = PURPLE; lifeTime = 0.47f; score = 5; break;
            case CoinType::White:
                color = WHITE; lifeTime = 0.30f; score = 100; break;
            default:
                color = GRAY; lifeTime = 0.4f; score = 0;
        }
    }

    void Draw() const {
        if (active)
            DrawCircleV(pos, radius, color);
    }
};

bool CheckCircleCollision(Vector2 a, float ar, Vector2 b, float br) {
    float dist = sqrtf((a.x - b.x)*(a.x - b.x) + (a.y - b.y)*(a.y - b.y));
    return dist <= (ar + br);
}

enum class GameState {
    TITLE,
    GAME,
    GAMEOVER
};

int main() {
    const int screenWidth = 800;
    const int screenHeight = 600;
    InitWindow(screenWidth, screenHeight, "Catch the Coin");
    SetTargetFPS(60);

    GameState state = GameState::TITLE;

    Vector2 circlePos = {(float)screenWidth/2, (float)screenHeight/2};
    float defaultCircleRadius = 25.0f;
    float circleRadius = defaultCircleRadius;
    bool dragging = false;
    Vector2 dragOffset = {0,0};

    float defaultMoveSpeed = 300.0f;
    float moveSpeed = defaultMoveSpeed;

    // 모든 능력 지속시간 2초로 통일
    const float BUFF_DURATION = 2.0f;

    // 버프 상태 변수들
    bool whiteBuffActive = false;
    float whiteBuffStartTime = 0.0f;

    bool speedBuffActive = false;
    float speedBuffStartTime = 0.0f;
    float speedBuffMultiplier = 1.0f;

    std::vector<Coin> coins;
    float coinSpawnTimer = 0.0f;
    int score = 0;

    const float TIME_LIMIT = 30.0f;
    float gameStartTime = 0.0f;

    while (!WindowShouldClose()) {
        if (state == GameState::TITLE) {
            if (IsKeyPressed(KEY_ONE)) {
                state = GameState::GAME;
                score = 0;
                coins.clear();
                circlePos = {(float)screenWidth/2, (float)screenHeight/2};
                circleRadius = defaultCircleRadius;
                moveSpeed = defaultMoveSpeed;
                whiteBuffActive = false;
                speedBuffActive = false;
                speedBuffMultiplier = 1.0f;
                gameStartTime = GetTime();
                coinSpawnTimer = 0.0f;
            }
        }
        else if (state == GameState::GAME) {
            float elapsed = GetTime() - gameStartTime;
            float remaining = TIME_LIMIT - elapsed;

            // 하얀색 버프 지속시간 체크
            if (whiteBuffActive && GetTime() - whiteBuffStartTime >= BUFF_DURATION) {
                circleRadius = defaultCircleRadius;
                whiteBuffActive = false;
            }

            // 속도 버프 지속시간 체크
            if (speedBuffActive && GetTime() - speedBuffStartTime >= BUFF_DURATION) {
                moveSpeed = defaultMoveSpeed;
                speedBuffActive = false;
                speedBuffMultiplier = 1.0f;
            }

            if (remaining <= 0.0f) {
                state = GameState::GAMEOVER;
            } else {
                Vector2 mouse = GetMousePosition();
                if (IsMouseButtonPressed(MOUSE_LEFT_BUTTON)) {
                    float dist = sqrtf((mouse.x - circlePos.x)*(mouse.x - circlePos.x) + (mouse.y - circlePos.y)*(mouse.y - circlePos.y));
                    if (dist <= circleRadius) {
                        dragging = true;
                        dragOffset = {circlePos.x - mouse.x, circlePos.y - mouse.y};
                    }
                }
                if (IsMouseButtonReleased(MOUSE_LEFT_BUTTON)) dragging = false;
                if (dragging) {
                    Vector2 target = {mouse.x + dragOffset.x, mouse.y + dragOffset.y};
                    Vector2 delta = {target.x - circlePos.x, target.y - circlePos.y};
                    float dist = sqrtf(delta.x*delta.x + delta.y*delta.y);
                    float maxMove = moveSpeed * GetFrameTime();
                    if (dist > maxMove) {
                        delta.x = delta.x / dist * maxMove;
                        delta.y = delta.y / dist * maxMove;
                    }
                    circlePos.x += delta.x;
                    circlePos.y += delta.y;
                }

                coinSpawnTimer += GetFrameTime();
                if (coinSpawnTimer > 0.25f) {
                    CoinType randomType = static_cast<CoinType>(GetRandomValue(0, static_cast<int>(CoinType::COUNT) - 1));
                    float margin = 60.0f;
                    Vector2 randomPos = {(float)(GetRandomValue(margin, screenWidth-margin)), (float)(GetRandomValue(margin, screenHeight-margin))};
                    coins.emplace_back(randomPos, randomType, GetTime());
                    coinSpawnTimer = 0.0f;
                }

                for (auto& coin : coins) {
                    if (!coin.active) continue;
                    if (GetTime() - coin.spawnTime > coin.lifeTime) {
                        coin.active = false;
                        continue;
                    }
                    if (CheckCircleCollision(circlePos, circleRadius, coin.pos, coin.radius)) {
                        score += coin.score;
                        switch (coin.type) {
                            case CoinType::White:
                                circleRadius = defaultCircleRadius * 2.0f;
                                whiteBuffActive = true;
                                whiteBuffStartTime = GetTime();
                                break;
                            case CoinType::Red:
                                moveSpeed = defaultMoveSpeed * 1.3f;
                                speedBuffActive = true;
                                speedBuffStartTime = GetTime();
                                speedBuffMultiplier = 1.3f;
                                break;
                            case CoinType::Green:
                                moveSpeed = defaultMoveSpeed * 0.8f;
                                speedBuffActive = true;
                                speedBuffStartTime = GetTime();
                                speedBuffMultiplier = 0.8f;
                                break;
                            case CoinType::Purple:
                                moveSpeed = defaultMoveSpeed * 0.6f;
                                speedBuffActive = true;
                                speedBuffStartTime = GetTime();
                                speedBuffMultiplier = 0.6f;
                                break;
                            default:
                                break;
                        }
                        coin.active = false;
                    }
                }
            }
        }
        else if (state == GameState::GAMEOVER) {
            if (IsKeyPressed(KEY_ONE)) {
                state = GameState::GAME;
                score = 0;
                coins.clear();
                circlePos = {(float)screenWidth/2, (float)screenHeight/2};
                circleRadius = defaultCircleRadius;
                moveSpeed = defaultMoveSpeed;
                whiteBuffActive = false;
                speedBuffActive = false;
                speedBuffMultiplier = 1.0f;
                gameStartTime = GetTime();
                coinSpawnTimer = 0.0f;
            }
        }

        BeginDrawing();
        ClearBackground(BLACK);

        if (state == GameState::TITLE) {
            const char* title = "CATCH THE COIN";
            int fontSize = 60;
            int titleWidth = MeasureText(title, fontSize);
            DrawText(title, (screenWidth - titleWidth) / 2, screenHeight/3 - fontSize, fontSize, WHITE);

            const char* insertCoin = "Press 1 key to Insert Coin";
            int insertWidth = MeasureText(insertCoin, 30);
            DrawText(insertCoin, (screenWidth - insertWidth) / 2, screenHeight/2, 30, LIGHTGRAY);

            const char* quitMsg = "Press Esc key to Quit";
            int quitWidth = MeasureText(quitMsg, 24);
            DrawText(quitMsg, (screenWidth - quitWidth) / 2, screenHeight/2 + 50, 24, GRAY);
        }
        else if (state == GameState::GAME) {
            DrawCircleV(circlePos, circleRadius, SKYBLUE);

            for (const auto& coin : coins)
                coin.Draw();

            DrawText(TextFormat("Score: %d", score), 10, 10, 30, WHITE);

            float elapsed = GetTime() - gameStartTime;
            float remaining = TIME_LIMIT - elapsed;
            if (remaining < 0) remaining = 0;
            DrawText(TextFormat("Time: %.1f", remaining), screenWidth - 160, 10, 30, WHITE);

            if (whiteBuffActive) {
                DrawText("White Buff: Size x2", 10, 50, 24, WHITE);
            }
            if (speedBuffActive) {
                DrawText(TextFormat("Speed Buff: %.2fx", speedBuffMultiplier), 10, 80, 24, WHITE);
            }
        }
        else if (state == GameState::GAMEOVER) {
            const char* result = TextFormat("Your Score: %d", score);
            int fontSize = 60;
            int resultWidth = MeasureText(result, fontSize);
            DrawText(result, (screenWidth - resultWidth)/2, screenHeight/2 - fontSize, fontSize, WHITE);

            const char* insertCoin = "Press 1 key to Insert Coin";
            int insertWidth = MeasureText(insertCoin, 30);
            DrawText(insertCoin, (screenWidth - insertWidth) / 2, screenHeight/2 + 40, 30, LIGHTGRAY);

            const char* quitMsg = "Press Esc key to Quit";
            int quitWidth = MeasureText(quitMsg, 24);
            DrawText(quitMsg, (screenWidth - quitWidth) / 2, screenHeight/2 + 90, 24, GRAY);
        }

        EndDrawing();

        if (IsKeyPressed(KEY_ESCAPE)) break;
    }

    CloseWindow();
    return 0;
}
```
