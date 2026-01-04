#include <iostream>
#include <vector>
#include <deque>
#include <ctime>
#include <cstdlib>
#include <string>
#include "rlutil.h" // 必須包含此標頭檔

using namespace std;

// --- 設定與常數 ---
const int MAP_WIDTH = 60;
const int MAP_HEIGHT = 20;
const int UI_OFFSET_Y = 5; // 遊戲區在 Y=5 之後開始

// 顏色定義 (對應 rlutil)
enum GameColor {
    COLOR_WHITE = rlutil::WHITE,
    COLOR_RED = rlutil::RED,
    COLOR_GREEN = rlutil::GREEN,
    COLOR_BLUE = rlutil::BLUE,
    COLOR_YELLOW = rlutil::YELLOW,
    COLOR_MAGENTA = rlutil::MAGENTA
};

struct Point {
    int x, y;
    bool operator==(const Point& other) const {
        return x == other.x && y == other.y;
    }
};

struct Item {
    Point pos;
    int color;
    bool active;
};

// --- 遊戲類別 ---
class ChromaticSnake {
private:
    deque<Point> snake;
    Point dir;
    Point exitPos;
    bool exitLocked;
    
    int currentColor;
    int targetColor;
    int level;
    bool gameOver;
    bool victory;
    
    vector<Item> items;
    vector<Point> walls;

public:
    ChromaticSnake() {
        srand(time(0));
        level = 1;
        resetGame();
    }

    void resetGame() {
        snake.clear();
        // 初始蛇位置
        snake.push_back({10, 10});
        snake.push_back({9, 10});
        snake.push_back({8, 10});
        
        dir = {1, 0}; // 向右
        currentColor = COLOR_WHITE;
        exitLocked = true;
        gameOver = false;
        victory = false;
        
        generateLevel();
    }

    void generateLevel() {
        items.clear();
        walls.clear();
        
        // 1. 設定目標顏色 (隨機從紅、綠、藍、黃中選)
        int colors[] = {COLOR_RED, COLOR_GREEN, COLOR_BLUE, COLOR_YELLOW};
        targetColor = colors[rand() % 4];

        // 2. 設置出口
        exitPos = {MAP_WIDTH - 2, MAP_HEIGHT - 2};
        exitLocked = true;

        // 3. 設置牆壁 (簡單外框)
        for (int x = 0; x < MAP_WIDTH; x++) {
            walls.push_back({x, 0});
            walls.push_back({x, MAP_HEIGHT - 1});
        }
        for (int y = 0; y < MAP_HEIGHT; y++) {
            walls.push_back({0, y});
            walls.push_back({MAP_WIDTH - 1, y});
        }

        // 4. 放置正確道具 (確保存在)
        // 簡單邏輯：隨機位置，不與牆壁重疊 (進階邏輯需使用 BFS/A* 確保路徑)
        items.push_back({ {rand() % (MAP_WIDTH - 4) + 2, rand() % (MAP_HEIGHT - 4) + 2}, targetColor, true });

        // 5. 放置錯誤道具 (陷阱)
        for (int i = 0; i < 3; i++) {
            int trapColor = colors[rand() % 4];
            if (trapColor == targetColor) trapColor = COLOR_MAGENTA; // 避免重複或混淆
            items.push_back({ {rand() % (MAP_WIDTH - 4) + 2, rand() % (MAP_HEIGHT - 4) + 2}, trapColor, true });
        }
    }

    void input() {
        if (kbhit()) {
            char k = rlutil::getkey();
            if (k == 'r' || k == 'R') {
                resetGame();
                return;
            }
            if (k == 27) { // ESC
                exit(0);
            }

            // 方向控制 (避免回頭)
            if (k == rlutil::KEY_UP && dir.y == 0) dir = {0, -1};
            else if (k == rlutil::KEY_DOWN && dir.y == 0) dir = {0, 1};
            else if (k == rlutil::KEY_LEFT && dir.x == 0) dir = {-1, 0};
            else if (k == rlutil::KEY_RIGHT && dir.x == 0) dir = {1, 0};
        }
    }

    void update() {
        if (gameOver || victory) return;

        Point newHead = {snake.front().x + dir.x, snake.front().y + dir.y};

        // A. 撞牆檢查
        for (auto& w : walls) {
            if (newHead == w) {
                gameOver = true;
                return;
            }
        }

        // B. 撞身體檢查
        for (auto& s : snake) {
            if (newHead == s) {
                gameOver = true;
                return;
            }
        }

        // C. 道具互動
        bool ate = false;
        for (auto& item : items) {
            if (item.active && newHead == item.pos) {
                if (item.color == targetColor) {
                    // 吃對了：變色、解鎖出口
                    currentColor = item.color;
                    exitLocked = false;
                    ate = true;
                    item.active = false;
                } else {
                    // 吃錯了：死亡
                    gameOver = true;
                    return;
                }
            }
        }

        // D. 出口判定
        if (newHead == exitPos) {
            if (!exitLocked && currentColor == targetColor) {
                victory = true;
                level++; // 簡單演示：勝利後直接重置並增加關卡號
                rlutil::msleep(500); // 暫停一下慶祝
                resetGame(); // 實際專案可切換到下一關生成邏輯
                return;
            } else {
                // 鎖住時視為牆壁，或直接穿過但沒反應？這裡設定為撞牆
                gameOver = true; 
                return;
            }
        }

        snake.push_front(newHead);
        if (!ate) {
            snake.pop_back();
        }
    }

    void draw() {
        rlutil::hidecursor();
        
        // --- 繪製 UI 區 (上方) ---
        rlutil::locate(1, 1);
        std::cout << "============================================================";
        rlutil::locate(1, 2);
        std::cout << " Level: " << level << "   Target: ";
        
        rlutil::setColor(targetColor);
        std::cout << "[ Color Block ]";
        rlutil::setColor(rlutil::GREY);
        
        std::cout << "   Status: ";
        if (exitLocked) std::cout << "Locked (#)"; 
        else { rlutil::setColor(rlutil::GREEN); std::cout << "OPEN (O)"; rlutil::setColor(rlutil::GREY); }

        rlutil::locate(1, 3);
        std::cout << " [R] Restart   [ESC] Quit";
        rlutil::locate(1, 4);
        std::cout << "============================================================";

        // --- 繪製 遊戲區 (下方) ---
        // 為了效能，這裡只重繪變動部分會更好，但為求代碼簡潔，這裡做全地圖清除重繪
        // 實際優化建議使用 double buffer 或只清除蛇尾
        
        // 畫牆壁
        rlutil::setColor(rlutil::GREY);
        for (auto& w : walls) {
            rlutil::locate(w.x + 1, w.y + UI_OFFSET_Y);
            std::cout << "#";
        }

        // 畫道具
        for (auto& item : items) {
            if (item.active) {
                rlutil::locate(item.pos.x + 1, item.pos.y + UI_OFFSET_Y);
                rlutil::setColor(item.color);
                std::cout << "Q"; // 道具圖示
            }
        }

        // 畫出口
        rlutil::locate(exitPos.x + 1, exitPos.y + UI_OFFSET_Y);
        if (exitLocked) {
            rlutil::setColor(rlutil::RED);
            std::cout << "X";
        } else {
            rlutil::setColor(rlutil::GREEN);
            std::cout << "O";
        }

        // 畫蛇
        for (size_t i = 0; i < snake.size(); i++) {
            rlutil::locate(snake[i].x + 1, snake[i].y + UI_OFFSET_Y);
            if (i == 0) { // 頭
                rlutil::setColor(currentColor);
                std::cout << (gameOver ? "x" : "@");
            } else { // 身
                rlutil::setColor(currentColor);
                std::cout << "o";
            }
        }

        // 遊戲結束訊息
        if (gameOver) {
            rlutil::locate(MAP_WIDTH / 2 - 4, MAP_HEIGHT / 2 + UI_OFFSET_Y);
            rlutil::setColor(rlutil::RED);
            std::cout << "GAME OVER";
        }

        rlutil::setColor(rlutil::GREY); // 重置顏色
    }

    void run() {
        while (true) {
            rlutil::cls(); // 清除螢幕
            draw();
            input();
            update();
            rlutil::msleep(100); // 遊戲速度控制
        }
    }
};

int main() {
    rlutil::saveDefaultColor();
    ChromaticSnake game;
    game.run();
    return 0;
}
