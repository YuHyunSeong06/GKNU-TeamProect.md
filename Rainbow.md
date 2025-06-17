# 무지개 사각형
```c
#pragma comment(lib, "opengl32.lib")
#pragma comment(lib, "glu32.lib")

#include <windows.h>
#include <gl/GL.h>
#include <gl/GLU.h>
#include <cmath>

LRESULT CALLBACK WndProc(HWND h, UINT m, WPARAM w, LPARAM l) {
    if (m == WM_DESTROY) { PostQuitMessage(0); return 0; }
    return DefWindowProc(h, m, w, l);
}

int WINAPI WinMain(HINSTANCE hInst, HINSTANCE hPrevInstance, LPSTR lpCmdLine, int nCmdShow) {
    WNDCLASS wc = { CS_OWNDC, WndProc, 0, 0, hInst, 0, 0, 0, 0, L"GL" };
    RegisterClass(&wc);
    HWND hwnd = CreateWindow(L"GL", L"Rainbow Circle Inside Rectangle",
        WS_OVERLAPPEDWINDOW, 100, 100, 800, 600, 0, 0, hInst, 0);
    ShowWindow(hwnd, nCmdShow);

    HDC dc = GetDC(hwnd);
    PIXELFORMATDESCRIPTOR pfd = { sizeof(pfd), 1,
        PFD_DRAW_TO_WINDOW | PFD_SUPPORT_OPENGL | PFD_DOUBLEBUFFER, PFD_TYPE_RGBA };
    SetPixelFormat(dc, ChoosePixelFormat(dc, &pfd), &pfd);
    HGLRC rc = wglCreateContext(dc); wglMakeCurrent(dc, rc);

    // 투영 설정 (2D 직교 투영)
    glMatrixMode(GL_PROJECTION);
    glLoadIdentity();
    gluOrtho2D(-1, 1, -1, 1);
    glMatrixMode(GL_MODELVIEW);

    MSG msg;
    float angle = 0.0f;

    // 무지개 색상
    float colors[7][3] = {
        {1.0f, 0.0f, 0.0f},   // 빨강
        {1.0f, 0.5f, 0.0f},   // 주황
        {1.0f, 1.0f, 0.0f},   // 노랑
        {0.0f, 1.0f, 0.0f},   // 초록
        {0.0f, 0.0f, 1.0f},   // 파랑
        {0.3f, 0.0f, 0.5f},   // 남색
        {0.5f, 0.0f, 1.0f}    // 보라
    };

    while (PeekMessage(&msg, 0, 0, 0, PM_REMOVE) || 1) {
        if (msg.message == WM_QUIT) break;
        TranslateMessage(&msg); DispatchMessage(&msg);

        glClearColor(0.0f, 0.0f, 0.0f, 1);  // 배경 검정
        glClear(GL_COLOR_BUFFER_BIT);
        glLoadIdentity();

        angle += 0.02f;

        // 사각형 배경 (흰색)
        glColor3f(1, 1, 1);
        glBegin(GL_QUADS);
        glVertex2f(-0.6f, -0.6f);
        glVertex2f(0.6f, -0.6f);
        glVertex2f(0.6f, 0.6f);
        glVertex2f(-0.6f, 0.6f);
        glEnd();

        // 동심원으로 무지개 원형 색상 넣기
        float cx = 0.0f, cy = 0.0f;
        int numCircles = 70;     // 원 겹 수
        int segments = 100;      // 각 원당 분할 수
        float maxRadius = 0.6f;  // 사각형 안에 딱 맞게

        for (int i = numCircles - 1; i >= 0; --i) {
            float r = maxRadius * i / numCircles;

            // 색상 인덱스 반복적으로 계산 (회전 효과 포함)
            int colorIndex = (i + int(angle * 10)) % 7;
            float* color = colors[colorIndex];
            glColor3f(color[0], color[1], color[2]);

            glBegin(GL_TRIANGLE_FAN);
            glVertex2f(cx, cy); // 중심
            for (int j = 0; j <= segments; ++j) {
                float theta = 2.0f * 3.14159f * j / segments;
                float x = cx + r * cos(theta);
                float y = cy + r * sin(theta);
                glVertex2f(x, y);
            }
            glEnd();
        }

        SwapBuffers(dc);
    }

    wglMakeCurrent(0, 0); wglDeleteContext(rc); ReleaseDC(hwnd, dc);
    return 0;
}
```
