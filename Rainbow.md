# 무지개 사각형
```c
#pragma comment(lib, "opengl32.lib")

#include <windows.h>
#include <gl/GL.h>
#include <cmath>  // sin, cos 등 사용
#include <ctime>

LRESULT CALLBACK WndProc(HWND h, UINT m, WPARAM w, LPARAM l) {
    if (m == WM_DESTROY) { PostQuitMessage(0); return 0; }
    return DefWindowProc(h, m, w, l);
}

int WINAPI WinMain(HINSTANCE hInst, HINSTANCE hPrevInstance, LPSTR lpCmdLine, int nCmdShow) {
    WNDCLASS wc = { CS_OWNDC, WndProc, 0, 0, hInst, 0, 0, 0, 0, L"GL" };
    RegisterClass(&wc);
    HWND hwnd = CreateWindow(L"GL", L"Rainbow Rectangle with 3D Effect",
        WS_OVERLAPPEDWINDOW, 100, 100, 800, 600, 0, 0, hInst, 0);
    ShowWindow(hwnd, nCmdShow);

    HDC dc = GetDC(hwnd);
    PIXELFORMATDESCRIPTOR pfd = { sizeof(pfd), 1,
        PFD_DRAW_TO_WINDOW | PFD_SUPPORT_OPENGL | PFD_DOUBLEBUFFER, PFD_TYPE_RGBA };
    SetPixelFormat(dc, ChoosePixelFormat(dc, &pfd), &pfd);
    HGLRC rc = wglCreateContext(dc); wglMakeCurrent(dc, rc);

    // 초기 설정: 투영 방식 변경
    glMatrixMode(GL_PROJECTION);
    glLoadIdentity();
    gluPerspective(45.0, 800.0 / 600.0, 1.0, 10.0);  // 3D 시야 확보
    glMatrixMode(GL_MODELVIEW);

    MSG msg;
    float angle = 0.0f;

    while (PeekMessage(&msg, 0, 0, 0, PM_REMOVE) || 1) {
        if (msg.message == WM_QUIT) break;
        TranslateMessage(&msg); DispatchMessage(&msg);

        glClearColor(0.0f, 0.9f, 0.99f, 1); // 배경색
        glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
        glEnable(GL_DEPTH_TEST); // 깊이 버퍼 사용

        glLoadIdentity();
        glTranslatef(0.0f, 0.0f, -3.0f); // 뒤로 카메라 이동 (Z축)

        angle += 0.01f; // 시간에 따라 변화하는 각도 (색 애니메이션용)

        // 무지개 색 사각형 그리기 (각 줄마다 색 변하게)
        float colors[7][3] = {
            {1.0f, 0.0f, 0.0f}, {1.0f, 0.5f, 0.0f}, {1.0f, 1.0f, 0.0f},
            {0.0f, 1.0f, 0.0f}, {0.0f, 0.0f, 1.0f}, {0.3f, 0.0f, 0.5f}, {0.5f, 0.0f, 1.0f}
        };

        float depth = 0.1f; // 사각형의 두께 (3D 효과용)
        for (int i = 0; i < 7; ++i) {
            float y1 = -0.5f + i * (1.0f / 7);
            float y2 = -0.5f + (i + 1) * (1.0f / 7);

            // 색상 시간에 따라 약간 변형 (애니메이션)
            float r = (colors[i][0] + sin(angle + i)) * 0.5f;
            float g = (colors[i][1] + sin(angle + i + 2)) * 0.5f;
            float b = (colors[i][2] + sin(angle + i + 4)) * 0.5f;
            glColor3f(r, g, b);

            glBegin(GL_QUADS);
            // 앞면
            glVertex3f(-0.5f, y1, depth);
            glVertex3f( 0.5f, y1, depth);
            glVertex3f( 0.5f, y2, depth);
            glVertex3f(-0.5f, y2, depth);

            // 뒷면
            glVertex3f(-0.5f, y1, -depth);
            glVertex3f( 0.5f, y1, -depth);
            glVertex3f( 0.5f, y2, -depth);
            glVertex3f(-0.5f, y2, -depth);

            // 상단
            glVertex3f(-0.5f, y2, depth);
            glVertex3f( 0.5f, y2, depth);
            glVertex3f( 0.5f, y2, -depth);
            glVertex3f(-0.5f, y2, -depth);

            // 하단
            glVertex3f(-0.5f, y1, depth);
            glVertex3f( 0.5f, y1, depth);
            glVertex3f( 0.5f, y1, -depth);
            glVertex3f(-0.5f, y1, -depth);

            glEnd();

            // 윤곽선 추가 (검은 선)
            glColor3f(0.0f, 0.0f, 0.0f);
            glBegin(GL_LINE_LOOP);
            glVertex3f(-0.5f, y1, depth);
            glVertex3f(0.5f, y1, depth);
            glVertex3f(0.5f, y2, depth);
            glVertex3f(-0.5f, y2, depth);
            glEnd();
        }

        SwapBuffers(dc);
    }

    wglMakeCurrent(0, 0); wglDeleteContext(rc); ReleaseDC(hwnd, dc);
    return 0;
}

```
