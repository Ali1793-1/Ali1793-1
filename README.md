#include "raylib.h"
#include <cmath>
#include <cstdlib>
#include <ctime>
#include <iostream>

// Constants
#define WIDTH 17
#define HEIGHT 20
#define CELL_SIZE 40
#define MIN_MATCH 3
#define MAX_ANGLE 180
#define MIN_ANGLE 0
#define SHOOT_SPEED 20
#define NEW_ROW_INTERVAL 800
#define BUBBLE_COLORS 4

// Bubble colors
Color bubbleColors[] = { RED, GREEN, BLUE, YELLOW };

// Node Class (for queue)
struct Node {
    int x, y;
    Node() : x(0), y(0) {} // Default constructor
    Node(int posX, int posY) : x(posX), y(posY) {}
};

// Custom Queue Class (Simplified Circular Array Implementation)
template <typename T>
class Queue {
private:
    T elements[1000]; // Static array for storing elements
    int front;
    int rear;
    int count;

public:
    Queue() : front(0), rear(0), count(0) {}

    // Add an item to the queue
    void Enqueue(const T& item) {
        if (count < 1000) {
            elements[rear] = item;
            rear = (rear + 1) % 1000; // Move rear circularly
            count++;
        }
    }

    // Remove the front item from the queue
    void Dequeue() {
        if (!IsEmpty()) {
            front = (front + 1) % 1000; // Move front circularly
            count--;
        }
    }

    // Get the front item of the queue
    T& Front() {
        return elements[front];
    }

    // Check if the queue is empty
    bool IsEmpty() const {
        return count == 0;
    }

    // Clear the queue
    void Clear() {
        front = 0;
        rear = 0;
        count = 0;
    }
};

// Bubble Class
class Bubble {
public:
    float x, y;      // Position
    int color;       // Color index
    bool isMoving;   // Whether the bubble is in motion
    Queue<Node> trajectory; // Custom queue to store the movement path

    Bubble(float startX, float startY, int colorIndex)
        : x(startX), y(startY), color(colorIndex), isMoving(false) {}

    void SetTrajectory(float angle) {
        trajectory.Clear();
        float currentX = x;
        float currentY = y;
        float dx = cos(angle * DEG2RAD) * SHOOT_SPEED;
        float dy = -sin(angle * DEG2RAD) * SHOOT_SPEED;

        for (int i = 0; i < 200; i++) {
            currentX += dx;
            currentY += dy;

            if (currentX <= CELL_SIZE / 2 || currentX >= WIDTH * CELL_SIZE - CELL_SIZE / 2) {
                dx = -dx;
            }

            if (currentY <= CELL_SIZE / 2) break;

            trajectory.Enqueue(Node((int)currentX, (int)currentY));
        }

        isMoving = true;
    }

    void Update() {
        if (isMoving && !trajectory.IsEmpty()) {
            Node nextStep = trajectory.Front();
            trajectory.Dequeue();
            x = nextStep.x;
            y = nextStep.y;
        }
        else {
            isMoving = false;
        }
    }

    void Draw() {
        DrawCircle((int)x, (int)y, CELL_SIZE / 2 - 5, bubbleColors[color - 1]);
    }
};

// Trajectory Class
class Trajectory {
public:
    float angle; // Shooting angle

    Trajectory() : angle(-90.0f) {}

    void AdjustAngle(int direction) {
        angle += direction;
        if (angle < MIN_ANGLE) angle = MIN_ANGLE;
        if (angle > MAX_ANGLE) angle = MAX_ANGLE;
    }

    void Draw(float startX, float startY) {
        float x = startX;
        float y = startY;
        float dx = cos(angle * DEG2RAD) * SHOOT_SPEED;
        float dy = -sin(angle * DEG2RAD) * SHOOT_SPEED;

        for (int i = 0; i < 200; i++) {
            x += dx;
            y += dy;

            if (x <= CELL_SIZE / 2 || x >= WIDTH * CELL_SIZE - CELL_SIZE / 2) {
                dx = -dx;
            }

            if (y <= CELL_SIZE / 2) break;

            DrawCircle((int)x, (int)y, 2, WHITE);
        }
    }
};

// Game Class
class Game {
private:
    int grid[HEIGHT][WIDTH];  // Game grid
    Bubble* currentBubble;    // Currently shooting bubble
    Trajectory trajectory;    // Trajectory instance
    int score;                // Game score
    int frameCount;           // Frame counter for spawning new rows
    bool isGameOver;          // Game over flag

    void SpawnBubble() {
        currentBubble = new Bubble(WIDTH * CELL_SIZE / 2, (HEIGHT - 2) * CELL_SIZE + CELL_SIZE / 2, rand() % BUBBLE_COLORS + 1);
    }

    void SpawnNewRow() {
        for (int i = HEIGHT - 1; i > 0; i--) {
            for (int j = 0; j < WIDTH; j++) {
                grid[i][j] = grid[i - 1][j];
            }
        }
        for (int j = 0; j < WIDTH; j++) {
            grid[0][j] = rand() % BUBBLE_COLORS + 1;
        }

        if (CheckGameOver()) {
            isGameOver = true;
        }
    }

    bool CheckGameOver() {
        for (int j = 0; j < WIDTH; j++) {
            if (grid[HEIGHT - 1][j] > 0) {
                return true;
            }
        }
        return false;
    }

    void PlaceBubble() {
        int gridX = (int)((currentBubble->x + CELL_SIZE / 2) / CELL_SIZE);
        int gridY = (int)((currentBubble->y + CELL_SIZE / 2) / CELL_SIZE);

        // Snap the bubble into the grid
        if (gridY < HEIGHT && gridX >= 0 && gridX < WIDTH && grid[gridY][gridX] == 0) {
            grid[gridY][gridX] = currentBubble->color;
            CheckAndPopClusters(gridX, gridY, currentBubble->color);
            delete currentBubble;
            currentBubble = nullptr;
            SpawnBubble();
        }
    }

    void CheckAndPopClusters(int x, int y, int color) {
        bool visited[HEIGHT][WIDTH] = { false };
        Node cluster[WIDTH * HEIGHT];
        int clusterSize = 0;
        Queue<Node> toVisit;

        toVisit.Enqueue(Node(x, y));
        visited[y][x] = true;

        while (!toVisit.IsEmpty()) {
            Node current = toVisit.Front();
            toVisit.Dequeue();
            cluster[clusterSize++] = current;

            int directions[4][2] = { {0, 1}, {1, 0}, {0, -1}, {-1, 0} };
            for (auto& dir : directions) {
                int newX = current.x + dir[0];
                int newY = current.y + dir[1];

                if (newX >= 0 && newX < WIDTH && newY >= 0 && newY < HEIGHT &&
                    !visited[newY][newX] && grid[newY][newX] == color) {
                    toVisit.Enqueue(Node(newX, newY));
                    visited[newY][newX] = true;
                }
            }
        }

        if (clusterSize >= MIN_MATCH) {
            for (int i = 0; i < clusterSize; i++) {
                grid[cluster[i].y][cluster[i].x] = 0;
                score += 10;
            }
        }
    }

public:
    Game() : score(0), frameCount(0), isGameOver(false) {
        srand(time(NULL));
        currentBubble = nullptr;

        // Initialize grid with random bubbles
        for (int i = 0; i < HEIGHT; i++) {
            for (int j = 0; j < WIDTH; j++) {
                grid[i][j] = (i < 4 && rand() % 2 == 0) ? rand() % BUBBLE_COLORS + 1 : 0;
            }
        }

        SpawnBubble();
    }

    ~Game() {
        delete currentBubble;
    }

    void Update() {
        if (isGameOver) return;

        frameCount++;
        if (frameCount % NEW_ROW_INTERVAL == 0) {
            SpawnNewRow();
        }

        if (currentBubble->isMoving) {
            currentBubble->Update();

            // Check for collision with top or other bubbles
            if (currentBubble->y <= CELL_SIZE / 2 ||
                grid[(int)(currentBubble->y / CELL_SIZE)][(int)(currentBubble->x / CELL_SIZE)] != 0) {
                currentBubble->isMoving = false;
                PlaceBubble();
            }
        }
        else {
            if (IsKeyPressed(KEY_LEFT)) {
                trajectory.AdjustAngle(-5);
            }
            if (IsKeyPressed(KEY_RIGHT)) {
                trajectory.AdjustAngle(5);
            }
            if (IsKeyPressed(KEY_SPACE)) {
                currentBubble->SetTrajectory(trajectory.angle);
            }
        }
    }

    void Draw() {
        if (isGameOver) {
            DrawText("GAME OVER", WIDTH * CELL_SIZE / 2 - 100, HEIGHT * CELL_SIZE / 2, 40, RED);
            DrawText("Press R to Restart", WIDTH * CELL_SIZE / 2 - 120, HEIGHT * CELL_SIZE / 2 + 50, 20, WHITE);
            if (IsKeyPressed(KEY_R)) {
                delete currentBubble;
                currentBubble = nullptr;
                *this = Game(); // Restart the game with cleanup
            }
            return;
        }

        // Draw grid
        for (int i = 0; i < HEIGHT; i++) {
            for (int j = 0; j < WIDTH; j++) {
                if (grid[i][j] > 0) {
                    int offset = (i % 2) * (CELL_SIZE / 2);
                    DrawCircle(j * CELL_SIZE + offset + CELL_SIZE / 2, i * CELL_SIZE + CELL_SIZE / 2, CELL_SIZE / 2 - 5, bubbleColors[grid[i][j] - 1]);
                }
            }
        }

        // Draw current bubble
        if (currentBubble) {
            currentBubble->Draw();
        }

        // Draw score
        DrawText(TextFormat("Score: %d", score), 10, 10, 20, WHITE);

        // Draw trajectory
        if (currentBubble && !currentBubble->isMoving) {
            trajectory.Draw(currentBubble->x, currentBubble->y);
        }
    }
};

// Main Function
int main() {
    InitWindow(WIDTH * CELL_SIZE, HEIGHT * CELL_SIZE, "Bubble Shooter with Queue Logic");
    SetTargetFPS(60);

    Game* gameInstance = new Game();

    while (!WindowShouldClose()) {
        gameInstance->Update();

        BeginDrawing();
        ClearBackground(BLACK);
        gameInstance->Draw();
        EndDrawing();
    }

    delete gameInstance;
    CloseWindow();

    return 0;
}
