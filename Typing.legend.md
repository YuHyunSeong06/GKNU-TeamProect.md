## 타자 연습 게임 추후 대규모 프로젝트에 추가 예정
##### 1차 타자 연습게임
```c
#define _CRT_SECURE_NO_WARNINGS
#include "raylib.h"
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <time.h>
#include <math.h>

#define SCREEN_WIDTH 1200
#define SCREEN_HEIGHT 700
#define MAX_INPUT_LENGTH 256
#define NUM_SENTENCES 15
#define MAX_PARTICLES 100

// 프로그램 상태
typedef enum {
    STATE_MENU,
    STATE_TYPING,
    STATE_RESULT,
    STATE_STATS
} GameState;

// 파티클 효과용 구조체
typedef struct {
    Vector2 position;
    Vector2 velocity;
    Color color;
    float lifetime;
    bool active;
} Particle;

// 타이핑 세션 데이터
typedef struct {
    char targetText[256];
    char userInput[256];
    int inputLength;
    double startTime;
    double endTime;
    int correctChars;
    int totalChars;
    int errors;
    double wpm;
    double accuracy;
    bool isActive;
    int currentSentenceIndex;
} TypingSession;

// 통계 데이터
typedef struct {
    int totalSessions;
    double totalWPM;
    double bestWPM;
    double averageAccuracy;
} Statistics;

// 전역 변수
GameState currentState = STATE_MENU;
TypingSession session = { 0 };
Statistics stats = { 0 };
Particle particles[MAX_PARTICLES] = { 0 };
float animationTime = 0.0f;
int selectedMenuItem = 0;

// 연습 문장들
const char* practiceTexts[NUM_SENTENCES] = {
    "The quick brown fox jumps over the lazy dog",
    "Programming is the art of telling another human what one wants the computer to do",
    "Code is like humor. When you have to explain it, it's bad",
    "First, solve the problem. Then, write the code",
    "Experience is the name everyone gives to their mistakes",
    "The best way to predict the future is to invent it",
    "Simplicity is the soul of efficiency",
    "Make it work, make it right, make it fast",
    "Any fool can write code that a computer can understand",
    "Good programmers write code that humans can understand",
    "Practice makes perfect in typing and coding",
    "Every expert was once a beginner who never gave up",
    "The keyboard is mightier than the sword",
    "Type with purpose and precision",
    "Speed comes with accuracy and practice"
};

// 파티클 생성
void CreateParticle(float x, float y, Color color) {
    for (int i = 0; i < MAX_PARTICLES; i++) {
        if (!particles[i].active) {
            particles[i].position = (Vector2){ x, y };
            particles[i].velocity = (Vector2){
                (float)(rand() % 200 - 100) / 50.0f,
                (float)(rand() % 200 - 250) / 50.0f
            };
            particles[i].color = color;
            particles[i].lifetime = 1.0f;
            particles[i].active = true;
            break;
        }
    }
}

// 파티클 업데이트
void UpdateParticles(float deltaTime) {
    for (int i = 0; i < MAX_PARTICLES; i++) {
        if (particles[i].active) {
            particles[i].position.x += particles[i].velocity.x;
            particles[i].position.y += particles[i].velocity.y;
            particles[i].velocity.y += 9.8f * deltaTime; // 중력
            particles[i].lifetime -= deltaTime * 2;

            if (particles[i].lifetime <= 0) {
                particles[i].active = false;
            }
        }
    }
}

// 파티클 그리기
void DrawParticles() {
    for (int i = 0; i < MAX_PARTICLES; i++) {
        if (particles[i].active) {
            Color c = particles[i].color;
            c.a = (unsigned char)(255 * particles[i].lifetime);
            DrawCircle(particles[i].position.x, particles[i].position.y,
                3 * particles[i].lifetime, c);
        }
    }
}

// WPM 계산
double CalculateWPM(int correctChars, double timeInSeconds) {
    if (timeInSeconds <= 0) return 0;
    return (correctChars / 5.0) / (timeInSeconds / 60.0);
}

// 정확도 계산
double CalculateAccuracy(int correct, int total) {
    if (total == 0) return 0;
    return (double)correct / total * 100.0;
}

// 새 세션 시작
void StartNewSession() {
    memset(&session, 0, sizeof(TypingSession));
    session.currentSentenceIndex = rand() % NUM_SENTENCES;
    strcpy(session.targetText, practiceTexts[session.currentSentenceIndex]);
    session.totalChars = strlen(session.targetText);
    session.isActive = true;
    session.startTime = GetTime();
    currentState = STATE_TYPING;
}

// 세션 종료 및 결과 계산
void EndSession() {
    session.endTime = GetTime();
    double elapsedTime = session.endTime - session.startTime;

    // 정확한 문자 수 계산
    session.correctChars = 0;
    session.errors = 0;
    int minLen = (session.inputLength < session.totalChars) ?
        session.inputLength : session.totalChars;

    for (int i = 0; i < minLen; i++) {
        if (session.userInput[i] == session.targetText[i]) {
            session.correctChars++;
        }
        else {
            session.errors++;
        }
    }

    // 길이 차이로 인한 오류 추가
    session.errors += abs(session.inputLength - session.totalChars);

    // WPM과 정확도 계산
    session.wpm = CalculateWPM(session.correctChars, elapsedTime);
    session.accuracy = CalculateAccuracy(session.correctChars, session.totalChars);

    // 통계 업데이트
    stats.totalSessions++;
    stats.totalWPM += session.wpm;
    if (session.wpm > stats.bestWPM) {
        stats.bestWPM = session.wpm;
    }
    stats.averageAccuracy = ((stats.averageAccuracy * (stats.totalSessions - 1)) +
        session.accuracy) / stats.totalSessions;

    session.isActive = false;
    currentState = STATE_RESULT;
}

// 메뉴 그리기
void DrawMenu() {
    // 배경 그라데이션
    DrawRectangleGradientV(0, 0, SCREEN_WIDTH, SCREEN_HEIGHT,
        (Color) {
        20, 20, 30, 255
    }, (Color) { 40, 40, 60, 255 });

    // 애니메이션 타이틀
    float titleY = 100 + sin(animationTime * 2) * 10;
    DrawText("TYPING MASTER",
        SCREEN_WIDTH / 2 - MeasureText("TYPING MASTER", 60) / 2,
        titleY, 60, WHITE);

    DrawText("Professional Typing Practice",
        SCREEN_WIDTH / 2 - MeasureText("Professional Typing Practice", 20) / 2,
        titleY + 70, 20, GRAY);

    // 메뉴 옵션들
    const char* menuItems[] = {
        "Start Practice",
        "View Statistics",
        "Exit"
    };

    int menuY = 300;
    for (int i = 0; i < 3; i++) {
        Color itemColor = (i == selectedMenuItem) ? YELLOW : WHITE;
        int fontSize = (i == selectedMenuItem) ? 35 : 30;

        if (i == selectedMenuItem) {
            DrawRectangle(SCREEN_WIDTH / 2 - 200, menuY + i * 80 - 10,
                400, 50, (Color) { 255, 255, 255, 20 });
        }

        DrawText(menuItems[i],
            SCREEN_WIDTH / 2 - MeasureText(menuItems[i], fontSize) / 2,
            menuY + i * 80, fontSize, itemColor);
    }

    // 지시사항
    DrawText("Use UP/DOWN arrows to navigate, ENTER to select",
        SCREEN_WIDTH / 2 - MeasureText("Use UP/DOWN arrows to navigate, ENTER to select", 20) / 2,
        SCREEN_HEIGHT - 100, 20, LIGHTGRAY);

    // 장식 요소
    DrawCircle(100, 100, 30 + sin(animationTime * 3) * 5, (Color) { 100, 100, 255, 100 });
    DrawCircle(SCREEN_WIDTH - 100, 150, 25 + cos(animationTime * 2) * 5, (Color) { 255, 100, 100, 100 });
}

// 타이핑 화면 그리기
void DrawTypingScreen() {
    // 배경
    DrawRectangleGradientV(0, 0, SCREEN_WIDTH, SCREEN_HEIGHT,
        (Color) {
        30, 30, 40, 255
    }, (Color) { 50, 50, 70, 255 });

    // 진행 상황 바
    float progress = (float)session.inputLength / session.totalChars;
    DrawRectangle(50, 50, SCREEN_WIDTH - 100, 30, (Color) { 40, 40, 50, 255 });
    DrawRectangle(50, 50, (SCREEN_WIDTH - 100) * progress, 30, GREEN);
    DrawText(TextFormat("Progress: %d/%d", session.inputLength, session.totalChars),
        SCREEN_WIDTH / 2 - 60, 55, 20, WHITE);

    // 타이머
    double elapsedTime = GetTime() - session.startTime;
    DrawText(TextFormat("Time: %.1f s", elapsedTime), 50, 100, 25, YELLOW);

    // 실시간 WPM
    double currentWPM = CalculateWPM(session.correctChars, elapsedTime);
    DrawText(TextFormat("WPM: %.0f", currentWPM), SCREEN_WIDTH - 200, 100, 25, SKYBLUE);

    // 목표 텍스트
    DrawText("Type this text:", 50, 180, 25, LIGHTGRAY);
    DrawRectangle(50, 220, SCREEN_WIDTH - 100, 80, (Color) { 20, 20, 30, 255 });

    // 목표 텍스트를 문자별로 그리기
    int xOffset = 60;
    int yOffset = 240;
    Font font = GetFontDefault();

    for (int i = 0; i < strlen(session.targetText); i++) {
        char c[2] = { session.targetText[i], '\0' };
        Color charColor = LIGHTGRAY;

        if (i < session.inputLength) {
            if (session.userInput[i] == session.targetText[i]) {
                charColor = GREEN;
            }
            else {
                charColor = RED;
            }
        }

        DrawText(c, xOffset, yOffset, 30, charColor);
        xOffset += MeasureText(c, 30);

        if (xOffset > SCREEN_WIDTH - 100) {
            xOffset = 60;
            yOffset += 35;
        }
    }

    // 사용자 입력 영역
    DrawText("Your input:", 50, 340, 25, LIGHTGRAY);
    DrawRectangle(50, 380, SCREEN_WIDTH - 100, 80, (Color) { 30, 30, 40, 255 });

    // 커서 애니메이션
    if ((int)(GetTime() * 2) % 2 == 0) {
        int cursorX = 60 + MeasureText(session.userInput, 30);
        DrawRectangle(cursorX, 400, 3, 35, WHITE);
    }

    DrawText(session.userInput, 60, 400, 30, WHITE);

    // 지시사항
    DrawText("Press ENTER when done | ESC to return to menu",
        SCREEN_WIDTH / 2 - MeasureText("Press ENTER when done | ESC to return to menu", 20) / 2,
        SCREEN_HEIGHT - 50, 20, GRAY);

    // 파티클 효과 그리기
    DrawParticles();
}

// 결과 화면 그리기
void DrawResultScreen() {
    // 배경
    DrawRectangleGradientV(0, 0, SCREEN_WIDTH, SCREEN_HEIGHT,
        (Color) {
        20, 30, 40, 255
    }, (Color) { 40, 50, 70, 255 });

    // 결과 타이틀
    DrawText("RESULTS",
        SCREEN_WIDTH / 2 - MeasureText("RESULTS", 50) / 2,
        80, 50, WHITE);

    // 결과 박스
    DrawRectangle(SCREEN_WIDTH / 2 - 300, 180, 600, 400, (Color) { 30, 30, 40, 200 });
    DrawRectangleLines(SCREEN_WIDTH / 2 - 300, 180, 600, 400, GOLD);

    // 통계 표시
    int statsY = 220;
    int statsSpacing = 60;

    double elapsedTime = session.endTime - session.startTime;

    DrawText(TextFormat("Time: %.2f seconds", elapsedTime),
        SCREEN_WIDTH / 2 - 200, statsY, 30, WHITE);

    DrawText(TextFormat("Speed: %.1f WPM", session.wpm),
        SCREEN_WIDTH / 2 - 200, statsY + statsSpacing, 30, SKYBLUE);

    DrawText(TextFormat("Accuracy: %.1f%%", session.accuracy),
        SCREEN_WIDTH / 2 - 200, statsY + statsSpacing * 2, 30,
        session.accuracy >= 90 ? GREEN : (session.accuracy >= 70 ? YELLOW : RED));

    DrawText(TextFormat("Correct: %d/%d", session.correctChars, session.totalChars),
        SCREEN_WIDTH / 2 - 200, statsY + statsSpacing * 3, 30, LIGHTGRAY);

    DrawText(TextFormat("Errors: %d", session.errors),
        SCREEN_WIDTH / 2 - 200, statsY + statsSpacing * 4, 30, ORANGE);

    // 평가 메시지
    const char* evaluation;
    Color evalColor;

    if (session.accuracy >= 95 && session.wpm >= 60) {
        evaluation = "EXCELLENT! You're a typing master!";
        evalColor = GOLD;
    }
    else if (session.accuracy >= 85 && session.wpm >= 40) {
        evaluation = "GREAT JOB! Keep up the good work!";
        evalColor = GREEN;
    }
    else if (session.accuracy >= 75 && session.wpm >= 30) {
        evaluation = "GOOD! Practice makes perfect!";
        evalColor = SKYBLUE;
    }
    else {
        evaluation = "KEEP PRACTICING! You'll get better!";
        evalColor = ORANGE;
    }

    DrawText(evaluation,
        SCREEN_WIDTH / 2 - MeasureText(evaluation, 25) / 2,
        500, 25, evalColor);

    // 버튼
    DrawText("Press SPACE to try again | ESC for menu",
        SCREEN_WIDTH / 2 - MeasureText("Press SPACE to try again | ESC for menu", 20) / 2,
        SCREEN_HEIGHT - 80, 20, LIGHTGRAY);

    // 장식 효과
    float starScale = 1.0f + sin(animationTime * 3) * 0.2f;
    if (session.accuracy >= 90) {
        DrawPoly((Vector2) { 100, 300 }, 5, 30 * starScale, 0, GOLD);
        DrawPoly((Vector2) { SCREEN_WIDTH - 100, 300 }, 5, 30 * starScale, 36, GOLD);
    }
}

// 통계 화면 그리기
void DrawStatsScreen() {
    // 배경
    DrawRectangleGradientV(0, 0, SCREEN_WIDTH, SCREEN_HEIGHT,
        (Color) {
        25, 25, 35, 255
    }, (Color) { 45, 45, 65, 255 });

    DrawText("STATISTICS",
        SCREEN_WIDTH / 2 - MeasureText("STATISTICS", 50) / 2,
        80, 50, WHITE);

    // 통계 박스
    DrawRectangle(SCREEN_WIDTH / 2 - 350, 180, 700, 400, (Color) { 30, 30, 40, 200 });
    DrawRectangleLines(SCREEN_WIDTH / 2 - 350, 180, 700, 400, SKYBLUE);

    int statsY = 220;
    int statsSpacing = 70;

    DrawText(TextFormat("Total Sessions: %d", stats.totalSessions),
        SCREEN_WIDTH / 2 - 250, statsY, 30, WHITE);

    if (stats.totalSessions > 0) {
        double avgWPM = stats.totalWPM / stats.totalSessions;
        DrawText(TextFormat("Average WPM: %.1f", avgWPM),
            SCREEN_WIDTH / 2 - 250, statsY + statsSpacing, 30, SKYBLUE);

        DrawText(TextFormat("Best WPM: %.1f", stats.bestWPM),
            SCREEN_WIDTH / 2 - 250, statsY + statsSpacing * 2, 30, GOLD);

        DrawText(TextFormat("Average Accuracy: %.1f%%", stats.averageAccuracy),
            SCREEN_WIDTH / 2 - 250, statsY + statsSpacing * 3, 30, GREEN);
    }
    else {
        DrawText("No sessions completed yet!",
            SCREEN_WIDTH / 2 - MeasureText("No sessions completed yet!", 25) / 2,
            statsY + statsSpacing * 2, 25, GRAY);
    }

    // 그래프 (간단한 시각화)
    if (stats.totalSessions > 0) {
        int graphY = 450;
        int barWidth = 60;
        int barSpacing = 80;
        int graphX = SCREEN_WIDTH / 2 - 120;

        // 평균 WPM 바
        float avgWPM = stats.totalWPM / stats.totalSessions;
        int avgBarHeight = (int)(avgWPM * 2);
        DrawRectangle(graphX, graphY - avgBarHeight, barWidth, avgBarHeight, SKYBLUE);
        DrawText("Avg", graphX + 15, graphY + 10, 15, WHITE);

        // 최고 WPM 바
        int bestBarHeight = (int)(stats.bestWPM * 2);
        DrawRectangle(graphX + barSpacing, graphY - bestBarHeight, barWidth, bestBarHeight, GOLD);
        DrawText("Best", graphX + barSpacing + 10, graphY + 10, 15, WHITE);

        // 정확도 바
        int accBarHeight = (int)(stats.averageAccuracy * 2);
        DrawRectangle(graphX + barSpacing * 2, graphY - accBarHeight, barWidth, accBarHeight, GREEN);
        DrawText("Acc%", graphX + barSpacing * 2 + 5, graphY + 10, 15, WHITE);
    }

    DrawText("Press ESC to return to menu",
        SCREEN_WIDTH / 2 - MeasureText("Press ESC to return to menu", 20) / 2,
        SCREEN_HEIGHT - 50, 20, LIGHTGRAY);
}

// 입력 처리
void HandleInput() {
    switch (currentState) {
    case STATE_MENU:
        if (IsKeyPressed(KEY_UP)) {
            selectedMenuItem = (selectedMenuItem - 1 + 3) % 3;
        }
        if (IsKeyPressed(KEY_DOWN)) {
            selectedMenuItem = (selectedMenuItem + 1) % 3;
        }
        if (IsKeyPressed(KEY_ENTER)) {
            switch (selectedMenuItem) {
            case 0:
                StartNewSession();
                break;
            case 1:
                currentState = STATE_STATS;
                break;
            case 2:
                CloseWindow();
                break;
            }
        }
        break;

    case STATE_TYPING:
        if (IsKeyPressed(KEY_ESCAPE)) {
            currentState = STATE_MENU;
            session.isActive = false;
        }

        // 텍스트 입력 처리
        int key = GetCharPressed();
        while (key > 0) {
            if (key >= 32 && key <= 126 && session.inputLength < MAX_INPUT_LENGTH - 1) {
                session.userInput[session.inputLength] = (char)key;
                session.inputLength++;
                session.userInput[session.inputLength] = '\0';

                // 올바른 입력 시 파티클 효과
                if (session.inputLength <= session.totalChars &&
                    session.userInput[session.inputLength - 1] ==
                    session.targetText[session.inputLength - 1]) {
                    CreateParticle(60 + MeasureText(session.userInput, 30),
                        420, GREEN);
                }
            }
            key = GetCharPressed();
        }

        // 백스페이스 처리
        if (IsKeyPressed(KEY_BACKSPACE) && session.inputLength > 0) {
            session.inputLength--;
            session.userInput[session.inputLength] = '\0';
        }

        // 엔터키로 제출
        if (IsKeyPressed(KEY_ENTER) || session.inputLength >= session.totalChars) {
            EndSession();
        }
        break;

    case STATE_RESULT:
        if (IsKeyPressed(KEY_SPACE)) {
            StartNewSession();
        }
        if (IsKeyPressed(KEY_ESCAPE)) {
            currentState = STATE_MENU;
        }
        break;

    case STATE_STATS:
        if (IsKeyPressed(KEY_ESCAPE)) {
            currentState = STATE_MENU;
        }
        break;
    }
}

int main(void) {
    // 초기화
    InitWindow(SCREEN_WIDTH, SCREEN_HEIGHT, "Typing Master - Professional Typing Practice");
    SetTargetFPS(60);

    // 랜덤 시드 설정
    srand(time(NULL));

    // 메인 루프
    while (!WindowShouldClose()) {
        // 입력 처리
        HandleInput();

        // 애니메이션 업데이트
        animationTime += GetFrameTime();
        UpdateParticles(GetFrameTime());

        // 그리기
        BeginDrawing();
        ClearBackground(BLACK);

        switch (currentState) {
        case STATE_MENU:
            DrawMenu();
            break;
        case STATE_TYPING:
            DrawTypingScreen();
            break;
        case STATE_RESULT:
            DrawResultScreen();
            break;
        case STATE_STATS:
            DrawStatsScreen();
            break;
        }

        EndDrawing();
    }

    // 정리
    CloseWindow();

    return 0;
}
```
1. 유니코드를 활용한 한글타자 구현
2.단어수에 따른 점수부여 긴 문장 성공시 그만틈 높은점수 하지만 오탈자 있을시 감점
3. (프로토타입 아마도 대규모때 탑재 가능성 높음)옛날 케이크 던지기 처럼 단어 오브젝트가 점점 화면에서 내려옴 단어를 입력하여 제거하기 
4. 성공할때마다 잠시 쉬었다 가는 개념으로 
Success!
( 현재까지 점수 ) 출력
5. 시간제한안에 많은 문장 입력해 높은 점수 맞추는게 목적인 게임
시간 종료시
Times up!!
( 현재까지 점수 )
6. (프로토타입) 초능력 문장 1% 확률로 등장 입력시 일정 시간동안 시간 일시중지
##### 2차 
