## 프로그래밍 심화 발표자료 제작

#####
```c
#define _CRT_SECURE_NO_WARNINGS
#include "raylib.h"
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <time.h>
#include <math.h>
#include <wchar.h>
#include <locale.h>

#define SCREEN_WIDTH 1200
#define SCREEN_HEIGHT 700
#define MAX_INPUT_LENGTH 512
#define NUM_SENTENCES 15
#define MAX_PARTICLES 100
#define CHALLENGE_TIME_LIMIT 300.0  // 5분

// 프로그램 상태
typedef enum {
    STATE_MENU,
    STATE_LANGUAGE_SELECT,
    STATE_TYPING,
    STATE_RESULT,
    STATE_STATS,
    STATE_SUCCESS_PAUSE,
    STATE_CHALLENGE,
    STATE_CHALLENGE_RESULT
} GameState;

// 언어 선택
typedef enum {
    LANG_ENGLISH,
    LANG_KOREAN
} Language;

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
    wchar_t targetText[256];
    wchar_t userInput[256];
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
    int score;
    int comboCount;
} TypingSession;

// 챌린지 모드 데이터
typedef struct {
    double startTime;
    double timeLimit;
    int totalScore;
    int completedSentences;
    int totalCharacters;
    int previousBestScore;
    bool isActive;
} ChallengeMode;

// 통계 데이터
typedef struct {
    int totalSessions;
    double totalWPM;
    double bestWPM;
    double averageAccuracy;
    int totalScore;
    int bestScore;
    int bestChallengeScore;
} Statistics;

// 전역 변수
GameState currentState = STATE_MENU;
Language currentLanguage = LANG_ENGLISH;
TypingSession session = { 0 };
ChallengeMode challenge = { 0 };
Statistics stats = { 0 };
Particle particles[MAX_PARTICLES] = { 0 };
float animationTime = 0.0f;
int selectedMenuItem = 0;
float successPauseTimer = 0.0f;
Font koreanFont;  // 한글 폰트

// 영어 연습 문장들
const char* englishTexts[NUM_SENTENCES] = {
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

// 한글 연습 문장들
const wchar_t* koreanTexts[NUM_SENTENCES] = {
    L"한글은 세종대왕이 창제한 우리나라의 고유 문자입니다",
    L"정보화 시대에는 빠른 타자 속도가 중요한 능력입니다",
    L"연습이 완벽을 만듭니다 꾸준히 노력하세요",
    L"프로그래밍은 논리적 사고를 기르는 좋은 방법입니다",
    L"실패는 성공의 어머니라는 말을 기억하세요",
    L"오늘 하루도 열심히 살아가는 당신을 응원합니다",
    L"작은 노력이 모여 큰 성과를 이룹니다",
    L"키보드 타자 연습으로 업무 효율을 높이세요",
    L"한글 타자는 자음과 모음의 조합으로 이루어집니다",
    L"디지털 시대의 필수 능력 빠른 타이핑 속도",
    L"천 리 길도 한 걸음부터 시작됩니다",
    L"노력은 배신하지 않습니다 계속 도전하세요",
    L"타자 속도와 정확도를 동시에 높여보세요",
    L"매일 조금씩 연습하면 실력이 향상됩니다",
    L"집중력과 인내심이 타자 실력 향상의 열쇠입니다"
};

// UTF-8 문자열을 와이드 문자열로 변환
void UTF8ToWide(const char* utf8, wchar_t* wide, int maxLen) {
    mbstowcs(wide, utf8, maxLen);
}

// 와이드 문자열을 UTF-8로 변환
void WideToUTF8(const wchar_t* wide, char* utf8, int maxLen) {
    wcstombs(utf8, wide, maxLen);
}

// 한글 문자인지 확인
bool IsKoreanChar(wchar_t ch) {
    return (ch >= 0xAC00 && ch <= 0xD7A3) || // 한글 완성형
           (ch >= 0x1100 && ch <= 0x11FF) || // 한글 자모
           (ch >= 0x3130 && ch <= 0x318F);   // 한글 호환 자모
}

// 점수 계산
int CalculateScore(int textLength, int errors, double timeSeconds) {
    int baseScore = textLength * 10;  // 기본 점수: 문자당 10점
    int lengthBonus = (textLength > 30) ? textLength * 5 : 0;  // 긴 문장 보너스
    int errorPenalty = errors * 20;  // 오류당 20점 감점
    int timeBonus = (timeSeconds < textLength) ? 100 : 0;  // 빠른 완성 보너스
    
    int finalScore = baseScore + lengthBonus - errorPenalty + timeBonus;
    return (finalScore < 0) ? 0 : finalScore;
}

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
            particles[i].velocity.y += 9.8f * deltaTime;
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
    if (currentLanguage == LANG_KOREAN) {
        // 한글은 한 글자를 2.5타로 계산
        return (correctChars * 2.5 / 5.0) / (timeInSeconds / 60.0);
    }
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
    
    if (currentLanguage == LANG_KOREAN) {
        wcscpy(session.targetText, koreanTexts[session.currentSentenceIndex]);
    } else {
        UTF8ToWide(englishTexts[session.currentSentenceIndex], session.targetText, 256);
    }
    
    session.totalChars = wcslen(session.targetText);
    session.isActive = true;
    session.startTime = GetTime();
    currentState = STATE_TYPING;
}

// 챌린지 모드 시작
void StartChallengeMode() {
    memset(&challenge, 0, sizeof(ChallengeMode));
    challenge.startTime = GetTime();
    challenge.timeLimit = CHALLENGE_TIME_LIMIT;
    challenge.isActive = true;
    challenge.previousBestScore = stats.bestChallengeScore;
    
    // 첫 문장 시작
    memset(&session, 0, sizeof(TypingSession));
    session.currentSentenceIndex = rand() % NUM_SENTENCES;
    
    if (currentLanguage == LANG_KOREAN) {
        wcscpy(session.targetText, koreanTexts[session.currentSentenceIndex]);
    } else {
        UTF8ToWide(englishTexts[session.currentSentenceIndex], session.targetText, 256);
    }
    
    session.totalChars = wcslen(session.targetText);
    session.isActive = true;
    session.startTime = GetTime();
    currentState = STATE_CHALLENGE;
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
    
    // 점수 계산
    session.score = CalculateScore(session.totalChars, session.errors, elapsedTime);
    stats.totalScore += session.score;
    
    if (session.score > stats.bestScore) {
        stats.bestScore = session.score;
    }

    // 통계 업데이트
    stats.totalSessions++;
    stats.totalWPM += session.wpm;
    if (session.wpm > stats.bestWPM) {
        stats.bestWPM = session.wpm;
    }
    stats.averageAccuracy = ((stats.averageAccuracy * (stats.totalSessions - 1)) +
        session.accuracy) / stats.totalSessions;

    session.isActive = false;
    
    // 성공 시 잠시 멈춤
    if (session.errors == 0 && session.inputLength == session.totalChars) {
        currentState = STATE_SUCCESS_PAUSE;
        successPauseTimer = 2.0f;  // 2초 대기
    } else {
        currentState = STATE_RESULT;
    }
}

// 챌린지 모드에서 다음 문장으로
void NextChallengeText() {
    challenge.completedSentences++;
    challenge.totalCharacters += session.correctChars;
    challenge.totalScore += session.correctChars * 5;  // 문자당 5점
    
    // 새 문장 시작
    memset(&session, 0, sizeof(TypingSession));
    session.currentSentenceIndex = rand() % NUM_SENTENCES;
    
    if (currentLanguage == LANG_KOREAN) {
        wcscpy(session.targetText, koreanTexts[session.currentSentenceIndex]);
    } else {
        UTF8ToWide(englishTexts[session.currentSentenceIndex], session.targetText, 256);
    }
    
    session.totalChars = wcslen(session.targetText);
    session.isActive = true;
}

// 챌린지 모드 종료
void EndChallengeMode() {
    challenge.isActive = false;
    
    // 최고 점수 갱신 확인
    if (challenge.totalScore > stats.bestChallengeScore) {
        stats.bestChallengeScore = challenge.totalScore;
    }
    
    currentState = STATE_CHALLENGE_RESULT;
}

// 언어 선택 화면 그리기
void DrawLanguageSelectScreen() {
    DrawRectangleGradientV(0, 0, SCREEN_WIDTH, SCREEN_HEIGHT,
        (Color){20, 20, 30, 255}, (Color){40, 40, 60, 255});
    
    DrawText("SELECT LANGUAGE",
        SCREEN_WIDTH/2 - MeasureText("SELECT LANGUAGE", 50)/2,
        150, 50, WHITE);
    
    const char* languages[] = {"English", "한국어 (Korean)"};
    
    for (int i = 0; i < 2; i++) {
        Color itemColor = (i == selectedMenuItem) ? YELLOW : WHITE;
        int fontSize = (i == selectedMenuItem) ? 40 : 35;
        
        if (i == selectedMenuItem) {
            DrawRectangle(SCREEN_WIDTH/2 - 200, 300 + i*100 - 10,
                400, 60, (Color){255, 255, 255, 20});
        }
        
        DrawText(languages[i],
            SCREEN_WIDTH/2 - MeasureText(languages[i], fontSize)/2,
            300 + i*100, fontSize, itemColor);
    }
    
    DrawText("Use UP/DOWN arrows to navigate, ENTER to select",
        SCREEN_WIDTH/2 - MeasureText("Use UP/DOWN arrows to navigate, ENTER to select", 20)/2,
        SCREEN_HEIGHT - 100, 20, LIGHTGRAY);
}

// 메뉴 그리기
void DrawMenu() {
    DrawRectangleGradientV(0, 0, SCREEN_WIDTH, SCREEN_HEIGHT,
        (Color){20, 20, 30, 255}, (Color){40, 40, 60, 255});

    float titleY = 100 + sin(animationTime * 2) * 10;
    DrawText("TYPING MASTER",
        SCREEN_WIDTH/2 - MeasureText("TYPING MASTER", 60)/2,
        titleY, 60, WHITE);

    DrawText("Professional Typing Practice",
        SCREEN_WIDTH/2 - MeasureText("Professional Typing Practice", 20)/2,
        titleY + 70, 20, GRAY);

    const char* menuItems[] = {
        "Practice Mode",
        "Challenge Mode (5 min)",
        "Select Language",
        "View Statistics",
        "Exit"
    };

    int menuY = 250;
    for (int i = 0; i < 5; i++) {
        Color itemColor = (i == selectedMenuItem) ? YELLOW : WHITE;
        int fontSize = (i == selectedMenuItem) ? 35 : 30;

        if (i == selectedMenuItem) {
            DrawRectangle(SCREEN_WIDTH/2 - 250, menuY + i*70 - 10,
                500, 50, (Color){255, 255, 255, 20});
        }

        DrawText(menuItems[i],
            SCREEN_WIDTH/2 - MeasureText(menuItems[i], fontSize)/2,
            menuY + i*70, fontSize, itemColor);
    }

    // 현재 언어 표시
    char langText[50];
    sprintf(langText, "Current Language: %s", 
        currentLanguage == LANG_KOREAN ? "Korean" : "English");
    DrawText(langText,
        SCREEN_WIDTH/2 - MeasureText(langText, 20)/2,
        SCREEN_HEIGHT - 150, 20, SKYBLUE);

    DrawText("Use UP/DOWN arrows to navigate, ENTER to select",
        SCREEN_WIDTH/2 - MeasureText("Use UP/DOWN arrows to navigate, ENTER to select", 20)/2,
        SCREEN_HEIGHT - 100, 20, LIGHTGRAY);
}

// 성공 일시정지 화면 그리기
void DrawSuccessPauseScreen() {
    DrawRectangleGradientV(0, 0, SCREEN_WIDTH, SCREEN_HEIGHT,
        (Color){20, 40, 20, 255}, (Color){40, 80, 40, 255});
    
    // Success! 애니메이션
    float scale = 1.0f + sin(animationTime * 5) * 0.2f;
    int fontSize = (int)(80 * scale);
    DrawText("Success!",
        SCREEN_WIDTH/2 - MeasureText("Success!", fontSize)/2,
        SCREEN_HEIGHT/2 - 100, fontSize, GOLD);
    
    // 현재 점수
    char scoreText[100];
    sprintf(scoreText, "Current Score: %d", stats.totalScore);
    DrawText(scoreText,
        SCREEN_WIDTH/2 - MeasureText(scoreText, 40)/2,
        SCREEN_HEIGHT/2, 40, WHITE);
    
    // 보너스 점수 표시
    sprintf(scoreText, "+%d points!", session.score);
    DrawText(scoreText,
        SCREEN_WIDTH/2 - MeasureText(scoreText, 30)/2,
        SCREEN_HEIGHT/2 + 60, 30, GREEN);
    
    // 파티클 효과
    for (int i = 0; i < 5; i++) {
        CreateParticle(SCREEN_WIDTH/2 + rand()%200 - 100,
            SCREEN_HEIGHT/2, GOLD);
    }
}

// 챌린지 모드 화면 그리기
void DrawChallengeScreen() {
    DrawRectangleGradientV(0, 0, SCREEN_WIDTH, SCREEN_HEIGHT,
        (Color){40, 20, 20, 255}, (Color){60, 40, 40, 255});
    
    // 남은 시간 표시
    double remainingTime = challenge.timeLimit - (GetTime() - challenge.startTime);
    if (remainingTime <= 0) {
        EndChallengeMode();
        return;
    }
    
    // 상단 정보 바
    DrawRectangle(0, 0, SCREEN_WIDTH, 100, (Color){20, 20, 30, 200});
    
    // 타이머 (크고 눈에 띄게)
    char timerText[50];
    int minutes = (int)remainingTime / 60;
    int seconds = (int)remainingTime % 60;
    sprintf(timerText, "%02d:%02d", minutes, seconds);
    Color timerColor = (remainingTime < 30) ? RED : (remainingTime < 60) ? ORANGE : WHITE;
    DrawText(timerText, 50, 30, 40, timerColor);
    
    // 점수
    sprintf(timerText, "Score: %d", challenge.totalScore);
    DrawText(timerText, SCREEN_WIDTH/2 - 100, 30, 35, GOLD);
    
    // 완성한 문장 수
    sprintf(timerText, "Completed: %d", challenge.completedSentences);
    DrawText(timerText, SCREEN_WIDTH - 250, 30, 25, SKYBLUE);
    
    // 목표 텍스트
    DrawText("Type this text:", 50, 130, 25, LIGHTGRAY);
    DrawRectangle(50, 170, SCREEN_WIDTH - 100, 100, (Color){20, 20, 30, 255});
    
    // 목표 텍스트 표시 (와이드 문자 처리)
    char displayText[512];
    WideToUTF8(session.targetText, displayText, 512);
    
    if (currentLanguage == LANG_KOREAN) {
        // 한글은 폰트 사용
        DrawTextEx(koreanFont, displayText, (Vector2){60, 190}, 30, 2, LIGHTGRAY);
    } else {
        DrawText(displayText, 60, 190, 30, LIGHTGRAY);
    }
    
    // 사용자 입력 영역
    DrawText("Your input:", 50, 300, 25, LIGHTGRAY);
    DrawRectangle(50, 340, SCREEN_WIDTH - 100, 100, (Color){30, 30, 40, 255});
    
    // 사용자 입력 표시
    WideToUTF8(session.userInput, displayText, 512);
    if (currentLanguage == LANG_KOREAN) {
        DrawTextEx(koreanFont, displayText, (Vector2){60, 360}, 30, 2, WHITE);
    } else {
        DrawText(displayText, 60, 360, 30, WHITE);
    }
    
    // 커서
    if ((int)(GetTime() * 2) % 2 == 0) {
        int cursorX = 60 + MeasureText(displayText, 30);
        DrawRectangle(cursorX, 360, 3, 35, WHITE);
    }
    
    DrawParticles();
}

// 챌린지 결과 화면 그리기
void DrawChallengeResultScreen() {
    DrawRectangleGradientV(0, 0, SCREEN_WIDTH, SCREEN_HEIGHT,
        (Color){30, 20, 40, 255}, (Color){50, 40, 70, 255});
    
    // Time's Up! 애니메이션
    float scale = 1.0f + sin(animationTime * 3) * 0.1f;
    int fontSize = (int)(70 * scale);
    DrawText("Time's Up!!",
        SCREEN_WIDTH/2 - MeasureText("Time's Up!!", fontSize)/2,
        100, fontSize, RED);
    
    // 결과 박스
    DrawRectangle(SCREEN_WIDTH/2 - 350, 220, 700, 300, (Color){30, 30, 40, 200});
    DrawRectangleLines(SCREEN_WIDTH/2 - 350, 220, 700, 300, GOLD);
    
    // 점수 표시
    char resultText[100];
    sprintf(resultText, "Final Score: %d", challenge.totalScore);
    DrawText(resultText,
        SCREEN_WIDTH/2 - MeasureText(resultText, 40)/2,
        260, 40, WHITE);
    
    sprintf(resultText, "Sentences Completed: %d", challenge.completedSentences);
    DrawText(resultText,
        SCREEN_WIDTH/2 - MeasureText(resultText, 30)/2,
        320, 30, SKYBLUE);
    
    sprintf(resultText, "Total Characters: %d", challenge.totalCharacters);
    DrawText(resultText,
        SCREEN_WIDTH/2 - MeasureText(resultText, 30)/2,
        360, 30, LIGHTGRAY);
    
    // 이전 최고 기록과 비교
    if (challenge.previousBestScore > 0) {
        if (challenge.totalScore >= challenge.previousBestScore * 2) {
            DrawText("INSANE!!!",
                SCREEN_WIDTH/2 - MeasureText("INSANE!!!", 50)/2,
                430, 50, (Color){255, 0, 255, 255});
        } else if (challenge.totalScore > challenge.previousBestScore) {
            DrawText("Awesome!",
                SCREEN_WIDTH/2 - MeasureText("Awesome!", 45)/2,
                430, 45, GOLD);
        }
    }
    
    DrawText("Press SPACE to try again | ESC for menu",
        SCREEN_WIDTH/2 - MeasureText("Press SPACE to try again | ESC for menu", 20)/2,
        SCREEN_HEIGHT - 80, 20, LIGHTGRAY);
}

// 타이핑 화면 그리기
void DrawTypingScreen() {
    DrawRectangleGradientV(0, 0, SCREEN_WIDTH, SCREEN_HEIGHT,
        (Color){30, 30, 40, 255}, (Color){50, 50, 70, 255});

    // 진행 상황 바
    float progress = (float)session.inputLength / session.totalChars;
    DrawRectangle(50, 50, SCREEN_WIDTH - 100, 30, (Color){40, 40, 50, 255});
    DrawRectangle(50, 50, (SCREEN_WIDTH - 100) * progress, 30, GREEN);
    DrawText(TextFormat("Progress: %d/%d", session.inputLength, session.totalChars),
        SCREEN_WIDTH/2 - 60, 55, 20, WHITE);

    // 타이머와 점수
    double elapsedTime = GetTime() - session.startTime;
    DrawText(TextFormat("Time: %.1f s", elapsedTime), 50, 100, 25, YELLOW);
    DrawText(TextFormat("Score: %d", stats.totalScore), SCREEN_WIDTH/2 - 50, 100, 25, GOLD);

    // 실시간 WPM
    double currentWPM = CalculateWPM(session.correctChars, elapsedTime);
    DrawText(TextFormat("WPM: %.0f", currentWPM), SCREEN_WIDTH - 200, 100, 25, SKYBLUE);

    // 목표 텍스트
    DrawText("Type this text:", 50, 180, 25, LIGHTGRAY);
    DrawRectangle(50, 220, SCREEN_WIDTH - 100, 100, (Color){20, 20, 30, 255});

    // 목표 텍스트 표시 (문자별 색상)
    char displayText[512];
    WideToUTF8(session.targetText, displayText, 512);
    
    int xOffset = 60;
    int yOffset = 240;
    
    for (int i = 0; i < wcslen(session.targetText); i++) {
        wchar_t wc[2] = { session.targetText[i], L'\0' };
        char c[10];
        WideToUTF8(wc, c, 10);
        
        Color charColor = LIGHTGRAY;
        if (i < session.inputLength) {
            if (session.userInput[i] == session.targetText[i]) {
                charColor = GREEN;
            } else {
                charColor = RED;
            }
        }
        
        if (currentLanguage == LANG_KOREAN && IsKoreanChar(session.targetText[i])) {
            DrawTextEx(koreanFont, c, (Vector2){xOffset, yOffset}, 30, 1, charColor);
            xOffset += MeasureTextEx(koreanFont, c, 30, 1).x;
        } else {
            DrawText(c, xOffset, yOffset, 30, charColor);
            xOffset += MeasureText(c, 30);
        }
        
        if (xOffset > SCREEN_WIDTH - 100) {
            xOffset = 60;
            yOffset += 35;
        }
    }

    // 사용자 입력 영역
    DrawText("Your input:", 50, 340, 25, LIGHTGRAY);
    DrawRectangle(50, 380, SCREEN_WIDTH - 100, 100, (Color){30, 30, 40, 255});

    // 커서 애니메이션
    WideToUTF8(session.userInput, displayText, 512);
    
    if (currentLanguage == LANG_KOREAN) {
        DrawTextEx(koreanFont, displayText, (Vector2){60, 400}, 30, 1, WHITE);
        if ((int)(GetTime() * 2) % 2 == 0) {
            int cursorX = 60 + MeasureTextEx(koreanFont, displayText, 30, 1).x;
            DrawRectangle(cursorX, 400, 3, 35, WHITE);
        }
    } else {
        DrawText(displayText, 60, 400, 30, WHITE);
        if ((int)(GetTime() * 2) % 2 == 0) {
            int cursorX = 60 + MeasureText(displayText, 30);
            DrawRectangle(cursorX, 400, 3, 35, WHITE);
        }
    }

    DrawText("Press ENTER when done | ESC to return to menu",
        SCREEN_WIDTH/2 - MeasureText("Press ENTER when done | ESC to return to menu", 20)/2,
        SCREEN_HEIGHT - 50, 20, GRAY);

    DrawParticles();
}

// 결과 화면 그리기
void DrawResultScreen() {
    DrawRectangleGradientV(0, 0, SCREEN_WIDTH, SCREEN_HEIGHT,
        (Color){20, 30, 40, 255}, (Color){40, 50, 70, 255});

    DrawText("RESULTS",
        SCREEN_WIDTH/2 - MeasureText("RESULTS", 50)/2,
        80, 50, WHITE);

    DrawRectangle(SCREEN_WIDTH/2 - 300, 180, 600, 450, (Color){30, 30, 40, 200});
    DrawRectangleLines(SCREEN_WIDTH/2 - 300, 180, 600, 450, GOLD);

    int statsY = 220;
    int statsSpacing = 55;

    double elapsedTime = session.endTime - session.startTime;

    DrawText(TextFormat("Time: %.2f seconds", elapsedTime),
        SCREEN_WIDTH/2 - 200, statsY, 30, WHITE);

    DrawText(TextFormat("Speed: %.1f WPM", session.wpm),
        SCREEN_WIDTH/2 - 200, statsY + statsSpacing, 30, SKYBLUE);

    DrawText(TextFormat("Accuracy: %.1f%%", session.accuracy),
        SCREEN_WIDTH/2 - 200, statsY + statsSpacing * 2, 30,
        session.accuracy >= 90 ? GREEN : (session.accuracy >= 70 ? YELLOW : RED));

    DrawText(TextFormat("Correct: %d/%d", session.correctChars, session.totalChars),
        SCREEN_WIDTH/2 - 200, statsY + statsSpacing * 3, 30, LIGHTGRAY);

    DrawText(TextFormat("Errors: %d", session.errors),
        SCREEN_WIDTH/2 - 200, statsY + statsSpacing * 4, 30, ORANGE);
    
    // 점수 표시
    DrawText(TextFormat("Score: %d points", session.score),
        SCREEN_WIDTH/2 - 200, statsY + statsSpacing * 5, 35, GOLD);
    
    DrawText(TextFormat("Total Score: %d", stats.totalScore),
        SCREEN_WIDTH/2 - 200, statsY + statsSpacing * 6, 30, WHITE);

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
        SCREEN_WIDTH/2 - MeasureText(evaluation, 25)/2,
        560, 25, evalColor);

    DrawText("Press SPACE to try again | ESC for menu",
        SCREEN_WIDTH/2 - MeasureText("Press SPACE to try again | ESC for menu", 20)/2,
        SCREEN_HEIGHT - 80, 20, LIGHTGRAY);

    if (session.accuracy >= 90) {
        float starScale = 1.0f + sin(animationTime * 3) * 0.2f;
        DrawPoly((Vector2){100, 300}, 5, 30 * starScale, 0, GOLD);
        DrawPoly((Vector2){SCREEN_WIDTH - 100, 300}, 5, 30 * starScale, 36, GOLD);
    }
}

// 통계 화면 그리기
void DrawStatsScreen() {
    DrawRectangleGradientV(0, 0, SCREEN_WIDTH, SCREEN_HEIGHT,
        (Color){25, 25, 35, 255}, (Color){45, 45, 65, 255});

    DrawText("STATISTICS",
        SCREEN_WIDTH/2 - MeasureText("STATISTICS", 50)/2,
        80, 50, WHITE);

    DrawRectangle(SCREEN_WIDTH/2 - 350, 180, 700, 450, (Color){30, 30, 40, 200});
    DrawRectangleLines(SCREEN_WIDTH/2 - 350, 180, 700, 450, SKYBLUE);

    int statsY = 220;
    int statsSpacing = 60;

    DrawText(TextFormat("Total Sessions: %d", stats.totalSessions),
        SCREEN_WIDTH/2 - 250, statsY, 30, WHITE);

    if (stats.totalSessions > 0) {
        double avgWPM = stats.totalWPM / stats.totalSessions;
        DrawText(TextFormat("Average WPM: %.1f", avgWPM),
            SCREEN_WIDTH/2 - 250, statsY + statsSpacing, 30, SKYBLUE);

        DrawText(TextFormat("Best WPM: %.1f", stats.bestWPM),
            SCREEN_WIDTH/2 - 250, statsY + statsSpacing * 2, 30, GOLD);

        DrawText(TextFormat("Average Accuracy: %.1f%%", stats.averageAccuracy),
            SCREEN_WIDTH/2 - 250, statsY + statsSpacing * 3, 30, GREEN);
        
        DrawText(TextFormat("Best Score: %d", stats.bestScore),
            SCREEN_WIDTH/2 - 250, statsY + statsSpacing * 4, 30, YELLOW);
        
        DrawText(TextFormat("Best Challenge Score: %d", stats.bestChallengeScore),
            SCREEN_WIDTH/2 - 250, statsY + statsSpacing * 5, 30, ORANGE);
    }
    else {
        DrawText("No sessions completed yet!",
            SCREEN_WIDTH/2 - MeasureText("No sessions completed yet!", 25)/2,
            statsY + statsSpacing * 2, 25, GRAY);
    }

    // 그래프
    if (stats.totalSessions > 0) {
        int graphY = 530;
        int barWidth = 50;
        int barSpacing = 70;
        int graphX = SCREEN_WIDTH/2 - 140;

        float avgWPM = stats.totalWPM / stats.totalSessions;
        int avgBarHeight = (int)(avgWPM * 1.5);
        DrawRectangle(graphX, graphY - avgBarHeight, barWidth, avgBarHeight, SKYBLUE);
        DrawText("Avg", graphX + 10, graphY + 5, 15, WHITE);

        int bestBarHeight = (int)(stats.bestWPM * 1.5);
        DrawRectangle(graphX + barSpacing, graphY - bestBarHeight, barWidth, bestBarHeight, GOLD);
        DrawText("Best", graphX + barSpacing + 5, graphY + 5, 15, WHITE);

        int accBarHeight = (int)(stats.averageAccuracy * 1.5);
        DrawRectangle(graphX + barSpacing * 2, graphY - accBarHeight, barWidth, accBarHeight, GREEN);
        DrawText("Acc%", graphX + barSpacing * 2, graphY + 5, 15, WHITE);
        
        int scoreBarHeight = (int)(stats.bestScore / 10);
        if (scoreBarHeight > 150) scoreBarHeight = 150;
        DrawRectangle(graphX + barSpacing * 3, graphY - scoreBarHeight, barWidth, scoreBarHeight, ORANGE);
        DrawText("Score", graphX + barSpacing * 3 - 5, graphY + 5, 15, WHITE);
    }

    DrawText("Press ESC to return to menu",
        SCREEN_WIDTH/2 - MeasureText("Press ESC to return to menu", 20)/2,
        SCREEN_HEIGHT - 50, 20, LIGHTGRAY);
}

// 한글 입력 처리
void ProcessKoreanInput(int key) {
    // 간단한 한글 입력 처리 (실제로는 IME 사용 권장)
    // 여기서는 기본적인 ASCII 입력만 처리
    if (key >= 32 && key <= 126 && session.inputLength < MAX_INPUT_LENGTH - 1) {
        session.userInput[session.inputLength] = (wchar_t)key;
        session.inputLength++;
        session.userInput[session.inputLength] = L'\0';
    }
}

// 입력 처리
void HandleInput() {
    switch (currentState) {
    case STATE_MENU:
        if (IsKeyPressed(KEY_UP)) {
            selectedMenuItem = (selectedMenuItem - 1 + 5) % 5;
        }
        if (IsKeyPressed(KEY_DOWN)) {
            selectedMenuItem = (selectedMenuItem + 1) % 5;
        }
        if (IsKeyPressed(KEY_ENTER)) {
            switch (selectedMenuItem) {
            case 0:
                StartNewSession();
                break;
            case 1:
                StartChallengeMode();
                break;
            case 2:
                currentState = STATE_LANGUAGE_SELECT;
                selectedMenuItem = currentLanguage;
                break;
            case 3:
                currentState = STATE_STATS;
                break;
            case 4:
                CloseWindow();
                break;
            }
        }
        break;
        
    case STATE_LANGUAGE_SELECT:
        if (IsKeyPressed(KEY_UP)) {
            selectedMenuItem = (selectedMenuItem - 1 + 2) % 2;
        }
        if (IsKeyPressed(KEY_DOWN)) {
            selectedMenuItem = (selectedMenuItem + 1) % 2;
        }
        if (IsKeyPressed(KEY_ENTER)) {
            currentLanguage = (Language)selectedMenuItem;
            currentState = STATE_MENU;
            selectedMenuItem = 0;
        }
        if (IsKeyPressed(KEY_ESCAPE)) {
            currentState = STATE_MENU;
            selectedMenuItem = 0;
        }
        break;

    case STATE_TYPING:
    case STATE_CHALLENGE:
        if (IsKeyPressed(KEY_ESCAPE)) {
            currentState = STATE_MENU;
            session.isActive = false;
            if (challenge.isActive) {
                EndChallengeMode();
            }
        }

        // 텍스트 입력 처리
        if (currentLanguage == LANG_KOREAN) {
            // 한글 입력 (간단한 처리)
            int key = GetCharPressed();
            while (key > 0) {
                ProcessKoreanInput(key);
                key = GetCharPressed();
            }
        } else {
            // 영어 입력
            int key = GetCharPressed();
            while (key > 0) {
                if (key >= 32 && key <= 126 && session.inputLength < MAX_INPUT_LENGTH - 1) {
                    session.userInput[session.inputLength] = (wchar_t)key;
                    session.inputLength++;
                    session.userInput[session.inputLength] = L'\0';

                    if (session.inputLength <= session.totalChars &&
                        session.userInput[session.inputLength - 1] ==
                        session.targetText[session.inputLength - 1]) {
                        CreateParticle(60 + session.inputLength * 15, 420, GREEN);
                    }
                }
                key = GetCharPressed();
            }
        }

        // 백스페이스 처리
        if (IsKeyPressed(KEY_BACKSPACE) && session.inputLength > 0) {
            session.inputLength--;
            session.userInput[session.inputLength] = L'\0';
        }

        // 엔터키로 제출
        if (IsKeyPressed(KEY_ENTER) || session.inputLength >= session.totalChars) {
            if (currentState == STATE_CHALLENGE) {
                // 챌린지 모드에서는 바로 다음 문장으로
                if (wcscmp(session.userInput, session.targetText) == 0) {
                    // 정확히 입력한 경우
                    challenge.totalScore += session.totalChars * 10;
                    CreateParticle(SCREEN_WIDTH/2, SCREEN_HEIGHT/2, GOLD);
                }
                NextChallengeText();
            } else {
                EndSession();
            }
        }
        break;
        
    case STATE_SUCCESS_PAUSE:
        // 자동으로 다음 화면으로 전환
        break;

    case STATE_RESULT:
        if (IsKeyPressed(KEY_SPACE)) {
            StartNewSession();
        }
        if (IsKeyPressed(KEY_ESCAPE)) {
            currentState = STATE_MENU;
        }
        break;
        
    case STATE_CHALLENGE_RESULT:
        if (IsKeyPressed(KEY_SPACE)) {
            StartChallengeMode();
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
    // 로케일 설정 (한글 처리용)
    setlocale(LC_ALL, "");
    
    // 초기화
    InitWindow(SCREEN_WIDTH, SCREEN_HEIGHT, "Typing Master - Korean & English");
    SetTargetFPS(60);
    
    // 한글 폰트 로드 (시스템에 있는 한글 폰트 경로 지정 필요)
    // Windows: "C:/Windows/Fonts/malgun.ttf"
    // Linux: "/usr/share/fonts/truetype/nanum/NanumGothic.ttf"
    // Mac: "/System/Library/Fonts/AppleSDGothicNeo.ttc"
    koreanFont = LoadFontEx("C:/Windows/Fonts/malgun.ttf", 32, NULL, 0);
    if (koreanFont.texture.id == 0) {
        // 폰트 로드 실패시 기본 폰트 사용
        koreanFont = GetFontDefault();
    }

    srand(time(NULL));

    // 메인 루프
    while (!WindowShouldClose()) {
        // 입력 처리
        HandleInput();

        // 애니메이션 업데이트
        animationTime += GetFrameTime();
        UpdateParticles(GetFrameTime());
        
        // Success 화면 타이머 업데이트
        if (currentState == STATE_SUCCESS_PAUSE) {
            successPauseTimer -= GetFrameTime();
            if (successPauseTimer <= 0) {
                currentState = STATE_RESULT;
            }
        }
        
        // 챌린지 모드 시간 체크
        if (currentState == STATE_CHALLENGE) {
            double remainingTime = challenge.timeLimit - (GetTime() - challenge.startTime);
            if (remainingTime <= 0) {
                EndChallengeMode();
            }
        }

        // 그리기
        BeginDrawing();
        ClearBackground(BLACK);

        switch (currentState) {
        case STATE_MENU:
            DrawMenu();
            break;
        case STATE_LANGUAGE_SELECT:
            DrawLanguageSelectScreen();
            break;
        case STATE_TYPING:
            DrawTypingScreen();
            break;
        case STATE_SUCCESS_PAUSE:
            DrawSuccessPauseScreen();
            break;
        case STATE_CHALLENGE:
            DrawChallengeScreen();
            break;
        case STATE_CHALLENGE_RESULT:
            DrawChallengeResultScreen();
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
    UnloadFont(koreanFont);
    CloseWindow();

    return 0;
}
```
