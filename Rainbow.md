# 무지개 사각형
### 1. 무지개 사각형의 색상이 축을 중심으로 시계방향으로 돌아감
### 2. 정육면체 형태로 만들고 다각도에서 볼 수 있음
```c
#pragma comment(lib, "opengl32.lib")
#pragma comment(lib, "glu32.lib")

#include <windows.h>
#include <gl/GL.h>
#include <gl/GLU.h>
#include <math.h>

void HSVtoRGB(float h, float s, float v, float* r, float* g, float* b);

// 윈도우 프로시저
LRESULT CALLBACK WndProc(HWND hWnd, UINT msg, WPARAM wParam, LPARAM lParam) {
    static BOOL keys[256] = { 0 };
    switch (msg) {
    case WM_KEYDOWN: keys[wParam] = TRUE; break;
    case WM_KEYUP: keys[wParam] = FALSE; break;
    case WM_DESTROY: PostQuitMessage(0); return 0;
    }
    return DefWindowProc(hWnd, msg, wParam, lParam);
}

// HSV → RGB 변환 함수
void HSVtoRGB(float h, float s, float v, float* r, float* g, float* b) {
    int i = (int)(h * 6.0f);
    float f = h * 6.0f - i;
    float p = v * (1.0f - s);
    float q = v * (1.0f - f * s);
    float t = v * (1.0f - (1.0f - f) * s);
    switch (i % 6) {
    case 0: *r = v; *g = t; *b = p; break;
    case 1: *r = q; *g = v; *b = p; break;
    case 2: *r = p; *g = v; *b = t; break;
    case 3: *r = p; *g = q; *b = v; break;
    case 4: *r = t; *g = p; *b = v; break;
    case 5: *r = v; *g = p; *b = q; break;
    }
}

// 큐브 면의 꼭짓점 정의
float faces[6][4][3] = {
    { {-1,-1, 1}, { 1,-1, 1}, { 1, 1, 1}, {-1, 1, 1} }, // Front
    { { 1,-1,-1}, {-1,-1,-1}, {-1, 1,-1}, { 1, 1,-1} }, // Back
    { {-1,-1,-1}, {-1,-1, 1}, {-1, 1, 1}, {-1, 1,-1} }, // Left
    { { 1,-1, 1}, { 1,-1,-1}, { 1, 1,-1}, { 1, 1, 1} }, // Right
    { {-1, 1, 1}, { 1, 1, 1}, { 1, 1,-1}, {-1, 1,-1} }, // Top
    { {-1,-1,-1}, { 1,-1,-1}, { 1,-1, 1}, {-1,-1, 1} }, // Bottom
};

int main() {
    HINSTANCE hInst = GetModuleHandle(NULL);
    WNDCLASS wc = { CS_OWNDC, WndProc, 0, 0, hInst, NULL, NULL, NULL, NULL, L"GLCube" };
    RegisterClass(&wc);

    HWND hwnd = CreateWindow(L"GLCube", L"Rotating Rainbow Cube with Per-Vertex Color", WS_OVERLAPPEDWINDOW,
        100, 100, 800, 600, NULL, NULL, hInst, NULL);
    ShowWindow(hwnd, SW_SHOW);

    HDC dc = GetDC(hwnd);
    PIXELFORMATDESCRIPTOR pfd = {
        sizeof(pfd), 1, PFD_DRAW_TO_WINDOW | PFD_SUPPORT_OPENGL | PFD_DOUBLEBUFFER,
        PFD_TYPE_RGBA, 32,
        0, 0, 0, 0, 0, 0,
        0, 0, 0, 0, 0, 0, 0,
        24, 8, 0, PFD_MAIN_PLANE, 0, 0, 0, 0
    };
    SetPixelFormat(dc, ChoosePixelFormat(dc, &pfd), &pfd);
    HGLRC rc = wglCreateContext(dc); wglMakeCurrent(dc, rc);

    glEnable(GL_DEPTH_TEST);
    MSG msg;
    ULONGLONG startTime = GetTickCount64();

    BOOL keys[256] = { 0 };
    float angleX = 20.0f, angleY = 30.0f;

    while (1) {
        while (PeekMessage(&msg, NULL, 0, 0, PM_REMOVE)) {
            if (msg.message == WM_QUIT) return 0;
            if (msg.message == WM_KEYDOWN) keys[msg.wParam] = TRUE;
            if (msg.message == WM_KEYUP) keys[msg.wParam] = FALSE;
            TranslateMessage(&msg); DispatchMessage(&msg);
        }

        if (keys[VK_LEFT])  angleY -= 0.02f;
        if (keys[VK_RIGHT]) angleY += 0.02f;
        if (keys[VK_UP])    angleX -= 0.02f;
        if (keys[VK_DOWN])  angleX += 0.02f;

        ULONGLONG time = GetTickCount64() - startTime;
        float baseHue = fmodf((time % 10000) / 10000.0f, 1.0f); // 10초 주기

        glClearColor(0.1f, 0.1f, 0.1f, 1.0f);
        glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);

        glMatrixMode(GL_PROJECTION);
        glLoadIdentity();
        gluPerspective(60.0, 800.0 / 600.0, 1.0, 100.0);

        glMatrixMode(GL_MODELVIEW);
        glLoadIdentity();
        glTranslatef(0.0f, 0.0f, -5.0f);
        glRotatef(angleX, 1.0f, 0.0f, 0.0f);
        glRotatef(angleY, 0.0f, 1.0f, 0.0f);

        // 면 렌더링
        for (int f = 0; f < 6; ++f) {
            glBegin(GL_QUADS);
            for (int v = 0; v < 4; ++v) {
                float hue = fmodf(baseHue + v * 0.25f, 1.0f);  // 시계방향 변화
                float r, g, b;
                HSVtoRGB(hue, 1.0f, 1.0f, &r, &g, &b);
                glColor3f(r, g, b);
                glVertex3fv(faces[f][v]);
            }
            glEnd();
        }

        // 윤곽선
        glColor3f(0.0f, 0.0f, 0.0f);
        glLineWidth(2.0f);
        for (int f = 0; f < 6; ++f) {
            glBegin(GL_LINE_LOOP);
            for (int i = 0; i < 4; ++i)
                glVertex3fv(faces[f][i]);
            glEnd();
        }

        SwapBuffers(dc);
    }

    wglMakeCurrent(NULL, NULL);
    wglDeleteContext(rc);
    ReleaseDC(hwnd, dc);
    return 0;
}
```
