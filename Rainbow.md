# 무지개 사각형
```c
#pragma comment(lib, "opengl32.lib")

#include <windows.h>
#include <gl/GL.h>

LRESULT CALLBACK WndProc(HWND h, UINT m, WPARAM w, LPARAM l) {
    if (m == WM_DESTROY) { PostQuitMessage(0); return 0; }
    return DefWindowProc(h, m, w, l);
}

int WINAPI WinMain(HINSTANCE hInst, HINSTANCE hPrevInstance, LPSTR lpCmdLine, int nCmdShow) {
    WNDCLASS wc = { CS_OWNDC, WndProc, 0, 0, hInst, 0, 0, 0, 0, L"GL" };
    RegisterClass(&wc);
    HWND hwnd = CreateWindow(L"GL", L"Rainbow Rectangle with GL_QUADS",
        WS_OVERLAPPEDWINDOW, 100, 100, 800, 600, 0, 0, hInst, 0);
    ShowWindow(hwnd, nCmdShow);

    HDC dc = GetDC(hwnd);
    PIXELFORMATDESCRIPTOR pfd = { sizeof(pfd), 1,
        PFD_DRAW_TO_WINDOW | PFD_SUPPORT_OPENGL | PFD_DOUBLEBUFFER, PFD_TYPE_RGBA };
    SetPixelFormat(dc, ChoosePixelFormat(dc, &pfd), &pfd);
    HGLRC rc = wglCreateContext(dc); wglMakeCurrent(dc, rc);

    MSG msg;
    while (PeekMessage(&msg, 0, 0, 0, PM_REMOVE) || 1) {
        if (msg.message == WM_QUIT) break;
        TranslateMessage(&msg); DispatchMessage(&msg);

        glClearColor(0.0f, 0.9f, 0.99f, 1); // 배경색
        glClear(GL_COLOR_BUFFER_BIT);

        glBegin(GL_QUADS); // 사각형 시작

        glColor3f(1, 0, 0); glVertex2f(-0.5f, -0.5f); // 좌하단 - 빨강
        glColor3f(0, 1, 0); glVertex2f(0.5f, -0.5f); // 우하단 - 초록
        glColor3f(0, 0, 1); glVertex2f(0.5f, 0.5f); // 우상단 - 파랑
        glColor3f(1, 1, 0); glVertex2f(-0.5f, 0.5f); // 좌상단 - 노랑

        glEnd();

        SwapBuffers(dc);
    }

    wglMakeCurrent(0, 0); wglDeleteContext(rc); ReleaseDC(hwnd, dc);
    return 0;
}
```
