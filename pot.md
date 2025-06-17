# 와이어 주전자
```c
#include <windows.h>
#include <GL/freeglut.h>
#pragma comment(lib, "freeglut.lib")

float rotationAngle = 0.0f;

void init() {
    glClearColor(0.0f, 0.0f, 0.0f, 1.0f);
    glEnable(GL_DEPTH_TEST);

    // 조명 설정
    glEnable(GL_LIGHTING);
    glEnable(GL_LIGHT0);

    GLfloat light_pos[] = { 1.0f, 1.0f, 1.0f, 0.0f };
    glLightfv(GL_LIGHT0, GL_POSITION, light_pos);

    glEnable(GL_COLOR_MATERIAL);
}

void drawTeapot() {
    glPushMatrix(); // 현재 행렬 저장
    glRotatef(rotationAngle, 0.0f, 1.0f, 0.0f); // Y축 중심 회전
    glColor3f(0.8f, 0.2f, 0.2f); // 주전자 색상
    glutSolidTeapot(0.5);
    glPopMatrix(); // 이전 행렬 복원
}

void display() {
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);

    glMatrixMode(GL_MODELVIEW);
    glLoadIdentity();
    gluLookAt(2.0, 2.0, 2.0,   // eye
        0.0, 0.0, 0.0,   // center
        0.0, 1.0, 0.0);  // up

    drawTeapot();

    glutSwapBuffers();
}

void reshape(int w, int h) {
    if (h == 0) h = 1;
    glViewport(0, 0, w, h);

    glMatrixMode(GL_PROJECTION);
    glLoadIdentity();
    gluPerspective(45.0, (double)w / (double)h, 1.0, 10.0);

    glMatrixMode(GL_MODELVIEW);
}

void update(int value) {
    rotationAngle += 1.0f;
    if (rotationAngle >= 360.0f)
        rotationAngle -= 360.0f;

    glutPostRedisplay();
    glutTimerFunc(16, update, 0); // 약 60 FPS
}

int main(int argc, char** argv) {
    glutInit(&argc, argv);
    glutInitDisplayMode(GLUT_DOUBLE | GLUT_RGB | GLUT_DEPTH);
    glutInitWindowSize(800, 600);
    glutCreateWindow("Rotating Teapot");

    init();
    glutDisplayFunc(display);
    glutReshapeFunc(reshape);
    glutTimerFunc(16, update, 0);

    glutMainLoop();
    return 0;
}
```
