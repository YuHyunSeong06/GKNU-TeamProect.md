# Catch The Coin
### 2025.05.13 GKNU C++ 프로그래밍 과제
#### 중앙 원을 클릭해서 움직여 랜덤으로 생성되는 코인을 수집하는 게임
#### 각 코인별 효과/지속시간/점수/확률 추가
Red	속도 1.5배	2초
Pink	속도 1.3배	2초
Yellow	속도 1.1배	2초
Green	속도 0.95배	1.5초
Blue	속도 0.9배	1.5초
Purple	속도 0.8배	1.5초
SkyBlue	1.5초간 이동불가(스턴)	1.5초
White	크기 2배	2초
```cpp
#include "raylib.h"
#include <vector>
#include <cmath>

// 1. 코인 타입 정의 (SkyBlue 추가)
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
    SkyBlue,
    COUNT
};

// 2. 코인 클래스
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
            case CoinType::SkyBlue:
                color = SKYBLUE; lifeTime = 0.39f; score = 300; break;
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

// 3. 코인 등장 확률 (순서 CoinType과 맞추기)
float coinProbabilities[] = {
    17, // Red
    13, // Pink
    13, // Orange
    13, // Yellow
    10, // LightGreen
    10, // Green
    8,  // Blue
    7,  // Purple
    2,  // White
    7   // SkyBlue
};
const int coinTypeCount = sizeof(coinProbabilities)/sizeof(float);
float cumulativeProbabilities[coinTypeCount];

void InitProbabilities() {
    cumulativeProbabilities[0] = coinProbabilities[0];
    for (int i = 1; i < coinTypeCount; ++i)
        cumulativeProbabilities[i] = cumulativeProbabilities[i-1] + coinProbabilities[i];
}

// 4. 확률에 따라 코인 타입 뽑기
CoinType GetRandomCoinType() {
    float r = GetRandomValue(0, 9999) / 100.0f; // 0~100
    for (int i = 0; i < coinTypeCount; ++i) {
        if (r < cumulativeProbabilities[i])
            return static_cast<CoinType>(i);
    }
    return CoinType::White; // fallback
}

int main() {
    const int screenWidth = 800;
    const int screenHeight = 600;
    InitWindow(screenWidth, screenHeight, "Catch the Coin");
    SetTargetFPS(60);

    InitProbabilities();

    GameState state = GameState::TITLE;

    Vector2 circlePos = {(float)screenWidth/2, (float)screenHeight/2};
    float defaultCircleRadius = 25.0f;
    float circleRadius = defaultCircleRadius;
    bool dragging = false;
    Vector2 dragOffset = {0,0};

    float defaultMoveSpeed = 300.0f;
    float moveSpeed = defaultMoveSpeed;

    // 버프/디버프/스턴 지속시간
    const float SPEED_BUFF_DURATION = 2.0f;
    const float SPEED_DEBUFF_DURATION = 1.5f;
    const float STUN_DURATION = 1.5f;

    // 버프 상태 변수
    bool whiteBuffActive = false;
    float whiteBuffStartTime = 0.0f;

    bool speedBuffActive = false;
    float speedBuffStartTime = 0.0f;
    float speedBuffMultiplier = 1.0f;

    bool stunActive = false;
    float stunStartTime = 0.0f;

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
                stunActive = false;
                gameStartTime = GetTime();
                coinSpawnTimer = 0.0f;
            }
        }
        else if (state == GameState::GAME) {
            float elapsed = GetTime() - gameStartTime;
            float remaining = TIME_LIMIT - elapsed;

            // White 버프
            if (whiteBuffActive && GetTime() - whiteBuffStartTime >= SPEED_BUFF_DURATION) {
                circleRadius = defaultCircleRadius;
                whiteBuffActive = false;
            }

            // 속도 버프/디버프
            if (speedBuffActive) {
                float duration = (speedBuffMultiplier < 1.0f) ? SPEED_DEBUFF_DURATION : SPEED_BUFF_DURATION;
                if (GetTime() - speedBuffStartTime >= duration) {
                    moveSpeed = defaultMoveSpeed;
                    speedBuffActive = false;
                    speedBuffMultiplier = 1.0f;
                }
            }

            // 스턴(이동불가)
            if (stunActive && GetTime() - stunStartTime >= STUN_DURATION) {
                stunActive = false;
            }

            if (remaining <= 0.0f) {
                state = GameState::GAMEOVER;
            } else {
                Vector2 mouse = GetMousePosition();
                if (!stunActive) {
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
                }

                coinSpawnTimer += GetFrameTime();
                if (coinSpawnTimer > 0.25f) {
                    CoinType randomType = GetRandomCoinType();
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
                                moveSpeed = defaultMoveSpeed * 1.5f;
                                speedBuffActive = true;
                                speedBuffStartTime = GetTime();
                                speedBuffMultiplier = 1.5f;
                                break;
                            case CoinType::Pink:
                                moveSpeed = defaultMoveSpeed * 1.3f;
                                speedBuffActive = true;
                                speedBuffStartTime = GetTime();
                                speedBuffMultiplier = 1.3f;
                                break;
                            case CoinType::Yellow:
                                moveSpeed = defaultMoveSpeed * 1.1f;
                                speedBuffActive = true;
                                speedBuffStartTime = GetTime();
                                speedBuffMultiplier = 1.1f;
                                break;
                            case CoinType::Green:
                                moveSpeed = defaultMoveSpeed * 0.95f;
                                speedBuffActive = true;
                                speedBuffStartTime = GetTime();
                                speedBuffMultiplier = 0.95f;
                                break;
                            case CoinType::Blue:
                                moveSpeed = defaultMoveSpeed * 0.9f;
                                speedBuffActive = true;
                                speedBuffStartTime = GetTime();
                                speedBuffMultiplier = 0.9f;
                                break;
                            case CoinType::Purple:
                                moveSpeed = defaultMoveSpeed * 0.8f;
                                speedBuffActive = true;
                                speedBuffStartTime = GetTime();
                                speedBuffMultiplier = 0.8f;
                                break;
                            case CoinType::SkyBlue:
                                stunActive = true;
                                stunStartTime = GetTime();
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
                stunActive = false;
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
            if (stunActive) {
                DrawText("Stunned! Can't move!", 10, 110, 24, SKYBLUE);
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
