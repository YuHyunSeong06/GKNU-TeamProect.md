# 와이어 주전자
### 1. 스페이스바 키를 눌러서 색상이 변경될 수 있도록 만듬
### 2. 방향키를 통해서 주전자의 각도를 조정이 가능함
```c
#include <windows.h>
#include <GL/freeglut.h>

float rotationX = 0.0f;
float rotationY = 0.0f;

// 방향키 상태 저장 (눌려있는 동안 true)
bool keyStates[4] = { false, false, false, false };
enum { LEFT, RIGHT, UP, DOWN };

// 색상 목록
int currentColorIndex = 0;
float colors[][3] = {
    {0.8f, 0.2f, 0.2f},
    {0.2f, 0.8f, 0.2f},
    {0.2f, 0.2f, 0.8f},
    {0.8f, 0.8f, 0.2f},
    {0.8f, 0.2f, 0.8f}
};
const int colorCount = sizeof(colors) / sizeof(colors[0]);

void init() {
    glClearColor(0.05f, 0.05f, 0.1f, 1.0f);
    glEnable(GL_DEPTH_TEST);

    // 조명 설정
    glEnable(GL_LIGHTING);
    glEnable(GL_LIGHT0);

    GLfloat light_pos[] = { 2.0f, 2.0f, 2.0f, 1.0f };
    glLightfv(GL_LIGHT0, GL_POSITION, light_pos);

    GLfloat ambient[] = { 0.2f, 0.2f, 0.2f, 1.0f };
    GLfloat diffuse[] = { 0.8f, 0.8f, 0.8f, 1.0f };
    GLfloat specular[] = { 1.0f, 1.0f, 1.0f, 1.0f };
    glLightfv(GL_LIGHT0, GL_AMBIENT, ambient);
    glLightfv(GL_LIGHT0, GL_DIFFUSE, diffuse);
    glLightfv(GL_LIGHT0, GL_SPECULAR, specular);

    glEnable(GL_COLOR_MATERIAL);
    glShadeModel(GL_SMOOTH);
}

void drawTeapot() {
    glPushMatrix();

    glRotatef(rotationY, 0.0f, 1.0f, 0.0f); // Y축 회전
    glRotatef(rotationX, 1.0f, 0.0f, 0.0f); // X축 회전

    float* color = colors[currentColorIndex];
    glColor3f(color[0], color[1], color[2]);

    glutSolidTeapot(0.5); // 고정형 Teapot (해상도 고정이므로 재질로 부드러움 보완)

    glPopMatrix();
}

void display() {
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);

    glMatrixMode(GL_MODELVIEW);
    glLoadIdentity();
    gluLookAt(2.5, 2.0, 2.5, 0.0, 0.0, 0.0, 0.0, 1.0, 0.0);

    drawTeapot();

    glutSwapBuffers();
}

void reshape(int w, int h) {
    if (h == 0) h = 1;
    glViewport(0, 0, w, h);

    glMatrixMode(GL_PROJECTION);
    glLoadIdentity();
    gluPerspective(60.0, (double)w / (double)h, 1.0, 20.0);

    glMatrixMode(GL_MODELVIEW);
}

// 일반 키보드
void handleKeyboard(unsigned char key, int x, int y) {
    if (key == 32) { // Space
        currentColorIndex = (currentColorIndex + 1) % colorCount;
        glutPostRedisplay();
    }
}

// 특수키 누름 (방향키)
void handleSpecialKeyDown(int key, int x, int y) {
    switch (key) {
    case GLUT_KEY_LEFT:  keyStates[LEFT] = true; break;
    case GLUT_KEY_RIGHT: keyStates[RIGHT] = true; break;
    case GLUT_KEY_UP:    keyStates[UP] = true; break;
    case GLUT_KEY_DOWN:  keyStates[DOWN] = true; break;
    }
}

// 특수키 뗌
void handleSpecialKeyUp(int key, int x, int y) {
    switch (key) {
    case GLUT_KEY_LEFT:  keyStates[LEFT] = false; break;
    case GLUT_KEY_RIGHT: keyStates[RIGHT] = false; break;
    case GLUT_KEY_UP:    keyStates[UP] = false; break;
    case GLUT_KEY_DOWN:  keyStates[DOWN] = false; break;
    }
}

// 계속 갱신되는 idle 함수
void idle() {
    float speed = 0.3f;

    if (keyStates[LEFT])  rotationY -= speed;
    if (keyStates[RIGHT]) rotationY += speed;
    if (keyStates[UP])    rotationX -= speed;
    if (keyStates[DOWN])  rotationX += speed;

    glutPostRedisplay();
}

int main(int argc, char** argv) {
    glutInit(&argc, argv);
    glutInitDisplayMode(GLUT_DOUBLE | GLUT_RGB | GLUT_DEPTH);
    glutInitWindowSize(800, 600);
    glutCreateWindow("Smooth Rotating Teapot");

    init();
    glutDisplayFunc(display);
    glutReshapeFunc(reshape);
    glutKeyboardFunc(handleKeyboard);
    glutSpecialFunc(handleSpecialKeyDown);
    glutSpecialUpFunc(handleSpecialKeyUp); // 방향키 뗐을 때 처리
    glutIdleFunc(idle); // 계속 회전 처리

    glutMainLoop();
    return 0;
}

```
