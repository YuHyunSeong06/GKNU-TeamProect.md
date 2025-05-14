# Catch The Coin
### 2025.05.13 GKNU C++ 프로그래밍 과제
#### 중앙 원을 클릭해서 움직여 랜덤으로 생성되는 코인을 수집하는 게임
#### 각 코인별 효과/지속시간/점수/확률 추가
##### CoinType	Color 상수	RGB 값	Score	LifeTime(s)	Ability (Effect)	Probability (%)
##### Red	RED	(230,41,55)	50	0.35	Speed x1.5 for 2s	5
##### Pink	PINK	(255,109,194)	40	0.37	Speed x1.3 for 2s	7
##### Orange	ORANGE	(255,161,0)	30	0.40	-	9
##### Yellow	YELLOW	(253,249,0)	25	0.42	Speed x1.1 for 2s	10
##### LightGreen	LIME	(0,158,47)	20	0.44	-	12
##### Green	GREEN	(0,228,48)	15	0.45	Speed x0.95 for 1.5s	14
##### Blue	BLUE	(0,121,241)	10	0.46	Speed x0.9 for 1.5s	17
####Purple	PURPLE	(200,122,255)	5	0.47	Speed x0.8 for 1.5s	22
White	WHITE	(255,255,255)	100	0.30	Size x2 for 2s	2
SkyBlue	SKYBLUE	(102,191,255)	300	0.39	Can't move for 1.5s (stun)	2
```cpp
#include "raylib.h"
#include <vector>
#include <cmath>
#include <string>

// 1. 코인 타입 정의 (SkyBlue 포함)
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

// 두 원의 충돌 체크 함수
bool CheckCircleCollision(Vector2 a, float ar, Vector2 b, float br) {
    float dist = sqrtf((a.x - b.x) * (a.x - b.x) + (a.y - b.y) * (a.y - b.y));
    return dist <= (ar + br);
}

enum class GameState {
    TITLE,
    GAME,
    GAMEOVER
};

// 3. 코인 등장 확률 (CoinType 순서와 일치)
float coinProbabilities[] = {
    5,  // Red
    7,  // Pink
    9,  // Orange
    10, // Yellow
    12, // LightGreen
    14, // Green
    17, // Blue
    22, // Purple
    2,  // White
    2   // SkyBlue
};
const int coinTypeCount = sizeof(coinProbabilities) / sizeof(float);
float cumulativeProbabilities[coinTypeCount];

// 누적 확률 계산 함수
void InitProbabilities() {
    cumulativeProbabilities[0] = coinProbabilities[0];
    for (int i = 1; i < coinTypeCount; ++i)
        cumulativeProbabilities[i] = cumulativeProbabilities[i - 1] + coinProbabilities[i];
}

// 확률에 따라 코인 타입 반환 함수
CoinType GetRandomCoinType() {
    float r = GetRandomValue(0, 9999) / 100.0f; // 0~100
    for (int i = 0; i < coinTypeCount; ++i) {
        if (r < cumulativeProbabilities[i])
            return static_cast<CoinType>(i);
    }
    return CoinType::White; // fallback
}

// 코인 능력/확률 표시용 구조체
struct CoinInfo {
    const char* name;
    Color color;
    int score;
    float probability;
    const char* ability;
};

// 코인 정보 배열
CoinInfo coinInfos[] = {
    { "Red",      RED,      50, 5,  "Speed x1.5 for 2s" },
    { "Pink",     PINK,     40, 7,  "Speed x1.3 for 2s" },
    { "Orange",   ORANGE,   30, 9,  "-" },
    { "Yellow",   YELLOW,   25, 10, "Speed x1.1 for 2s" },
    { "LightGreen",LIME,    20, 12, "-" },
    { "Green",    GREEN,    15, 14, "Speed x0.95 for 1.5s" },
    { "Blue",     BLUE,     10, 17, "Speed x0.9 for 1.5s" },
    { "Purple",   PURPLE,   5,  22, "Speed x0.8 for 1.5s" },
    { "White",    WHITE,    100, 2, "Size x2 for 2s" },
    { "SkyBlue",  SKYBLUE,  300, 2, "Can't move for 1.5s" }
};

int main() {
    const int screenWidth = 1100;
    const int screenHeight = 700;
    InitWindow(screenWidth, screenHeight, "Catch the Coin");
    SetTargetFPS(60);

    InitProbabilities();

    GameState state = GameState::TITLE;

    Vector2 circlePos = { (float)screenWidth / 2, (float)screenHeight / 2 };
    float defaultCircleRadius = 25.0f;
    float circleRadius = defaultCircleRadius;
    bool dragging = false;
    Vector2 dragOffset = { 0,0 };

    // (1) 이동속도 2배
    float defaultMoveSpeed = 600.0f;
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

    // (2) 원 색상 변수
    Color circleColor = SKYBLUE;

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
                circlePos = { (float)screenWidth / 2, (float)screenHeight / 2 };
                circleRadius = defaultCircleRadius;
                moveSpeed = defaultMoveSpeed;
                whiteBuffActive = false;
                speedBuffActive = false;
                speedBuffMultiplier = 1.0f;
                stunActive = false;
                circleColor = SKYBLUE;
                gameStartTime = GetTime();
                coinSpawnTimer = 0.0f;
            }
        }
        else if (state == GameState::GAME) {
            float elapsed = GetTime() - gameStartTime;
            float remaining = TIME_LIMIT - elapsed;

            // White 버프 해제
            if (whiteBuffActive && GetTime() - whiteBuffStartTime >= SPEED_BUFF_DURATION) {
                circleRadius = defaultCircleRadius;
                whiteBuffActive = false;
            }

            // 속도 버프/디버프 해제
            if (speedBuffActive) {
                float duration = (speedBuffMultiplier < 1.0f) ? SPEED_DEBUFF_DURATION : SPEED_BUFF_DURATION;
                if (GetTime() - speedBuffStartTime >= duration) {
                    moveSpeed = defaultMoveSpeed;
                    speedBuffActive = false;
                    speedBuffMultiplier = 1.0f;
                }
            }

            // 스턴 해제
            if (stunActive && GetTime() - stunStartTime >= STUN_DURATION) {
                stunActive = false;
            }

            if (remaining <= 0.0f) {
                state = GameState::GAMEOVER;
            }
            else {
                Vector2 mouse = GetMousePosition();
                if (!stunActive) {
                    if (IsMouseButtonPressed(MOUSE_LEFT_BUTTON)) {
                        float dist = sqrtf((mouse.x - circlePos.x) * (mouse.x - circlePos.x) + (mouse.y - circlePos.y) * (mouse.y - circlePos.y));
                        if (dist <= circleRadius) {
                            dragging = true;
                            dragOffset = { circlePos.x - mouse.x, circlePos.y - mouse.y };
                        }
                    }
                    if (IsMouseButtonReleased(MOUSE_LEFT_BUTTON)) dragging = false;
                    if (dragging) {
                        Vector2 target = { mouse.x + dragOffset.x, mouse.y + dragOffset.y };
                        Vector2 delta = { target.x - circlePos.x, target.y - circlePos.y };
                        float dist = sqrtf(delta.x * delta.x + delta.y * delta.y);
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
                    Vector2 randomPos = { (float)(GetRandomValue(margin, screenWidth - margin)), (float)(GetRandomValue(margin, screenHeight - margin)) };
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
                        // (2) 코인을 먹으면 원 색상 변경
                        circleColor = coin.color;
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
                circlePos = { (float)screenWidth / 2, (float)screenHeight / 2 };
                circleRadius = defaultCircleRadius;
                moveSpeed = defaultMoveSpeed;
                whiteBuffActive = false;
                speedBuffActive = false;
                speedBuffMultiplier = 1.0f;
                stunActive = false;
                circleColor = SKYBLUE;
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
            DrawText(title, (screenWidth - titleWidth) / 2, 40, fontSize, WHITE);

            const char* insertCoin = "Press 1 key to Insert Coin";
            int insertWidth = MeasureText(insertCoin, 30);
            DrawText(insertCoin, (screenWidth - insertWidth) / 2, 120, 30, LIGHTGRAY);

            const char* quitMsg = "Press Esc key to Quit";
            int quitWidth = MeasureText(quitMsg, 24);
            DrawText(quitMsg, (screenWidth - quitWidth) / 2, 160, 24, GRAY);

            // (3) 코인 능력/확률 표시
            int y = 210;
            DrawText("Coin Ability & Probability", 40, y, 28, WHITE); y += 36;
            DrawText("Name      Score   Prob(%)    Ability", 40, y, 22, WHITE); y += 30;
            for (int i = 0; i < (int)CoinType::COUNT; i++) {
                DrawRectangle(40, y + 3, 20, 20, coinInfos[i].color);
                DrawText(TextFormat("%-9s %3d     %5.1f      %s",
                    coinInfos[i].name, coinInfos[i].score, coinInfos[i].probability, coinInfos[i].ability),
                    70, y, 20, coinInfos[i].color);
                y += 28;
            }
        }
        else if (state == GameState::GAME) {
            DrawCircleV(circlePos, circleRadius, circleColor);

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

            // (3) 게임 중에도 코인 능력/확률 표시
            int y = 170;
            DrawText("Coin Ability & Probability", 40, y, 24, WHITE); y += 30;
            DrawText("Name      Score   Prob(%)    Ability", 40, y, 18, WHITE); y += 24;
            for (int i = 0; i < (int)CoinType::COUNT; i++) {
                DrawRectangle(40, y + 2, 16, 16, coinInfos[i].color);
                DrawText(TextFormat("%-9s %3d     %5.1f      %s",
                    coinInfos[i].name, coinInfos[i].score, coinInfos[i].probability, coinInfos[i].ability),
                    62, y, 16, coinInfos[i].color);
                y += 20;
            }
        }
        else if (state == GameState::GAMEOVER) {
            const char* result = TextFormat("Your Score: %d", score);
            int fontSize = 60;
            int resultWidth = MeasureText(result, fontSize);
            DrawText(result, (screenWidth - resultWidth) / 2, screenHeight / 2 - fontSize, fontSize, WHITE);

            const char* insertCoin = "Press 1 key to Insert Coin";
            int insertWidth = MeasureText(insertCoin, 30);
            DrawText(insertCoin, (screenWidth - insertWidth) / 2, screenHeight / 2 + 40, 30, LIGHTGRAY);

            const char* quitMsg = "Press Esc key to Quit";
            int quitWidth = MeasureText(quitMsg, 24);
            DrawText(quitMsg, (screenWidth - quitWidth) / 2, screenHeight / 2 + 90, 24, GRAY);

            // (3) 게임오버에도 코인 능력/확률 표시
            int y = 170;
            DrawText("Coin Ability & Probability", 40, y, 24, WHITE); y += 30;
            DrawText("Name      Score   Prob(%)    Ability", 40, y, 18, WHITE); y += 24;
            for (int i = 0; i < (int)CoinType::COUNT; i++) {
                DrawRectangle(40, y + 2, 16, 16, coinInfos[i].color);
                DrawText(TextFormat("%-9s %3d     %5.1f      %s",
                    coinInfos[i].name, coinInfos[i].score, coinInfos[i].probability, coinInfos[i].ability),
                    62, y, 16, coinInfos[i].color);
                y += 20;
            }
        }

        EndDrawing();

        if (IsKeyPressed(KEY_ESCAPE)) break;
    }

    CloseWindow();
    return 0;
}
```
