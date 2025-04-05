#include "raylib.h"
#include <math.h>

#define screenwidth 800
#define screenheight 600
#define movespeed 5.0f
#define jumpforce 9.0f
#define gravity 0.5f
#define framespeed 6
#define maxplatforms 50
#define pullpower 20.0f
#define minropedistance 5.0f

typedef enum {
    mainmenu,
    chooselevel,
    gameplay,
    leveldone
} gamestate;

typedef struct {
    Vector2 pos;
    float yspeed;
    bool inair;
    bool moving;
    int framecount;
} player;

typedef struct {
    Rectangle area;
    bool blueonly;
    bool redonly;
} platform;

void menu(Rectangle playbtn) {
    ClearBackground(WHITE);
    DrawText("Rope Runners", screenwidth/2 - MeasureText("Rope Runners", 40)/2, 150, 40, DARKGRAY);
    DrawRectangleRec(playbtn, BLUE);
    DrawText("Play", playbtn.x + playbtn.width/2 - MeasureText("Play", 30)/2, 
             playbtn.y + playbtn.height/2 - 15, 30, WHITE);
}

void levelselect(Rectangle level1btn, Rectangle level2btn, Rectangle backbtn) {
    ClearBackground(WHITE);
    DrawText("Select Level", screenwidth/2 - MeasureText("Select Level", 40)/2, 100, 40, DARKGRAY);
    
    DrawRectangleRec(level1btn, BLUE);
    DrawText("Level 1", level1btn.x + level1btn.width/2 - MeasureText("Level 1", 30)/2, 
             level1btn.y + level1btn.height/2 - 15, 30, WHITE);
    
    DrawRectangleRec(level2btn, BLUE);
    DrawText("Level 2", level2btn.x + level2btn.width/2 - MeasureText("Level 2", 30)/2, 
             level2btn.y + level2btn.height/2 - 15, 30, WHITE);
    
    DrawRectangleRec(backbtn, RED);
    DrawText("Back", backbtn.x + backbtn.width/2 - MeasureText("Back", 20)/2, 
             backbtn.y + backbtn.height/2 - 10, 20, WHITE);
}

void levelcomplete(Rectangle replaybtn, Rectangle nextbtn, Rectangle menubtn, Rectangle level1btn, int level) {
    ClearBackground(WHITE);
    DrawText("Level Complete!", screenwidth/2 - MeasureText("Level Complete!", 40)/2, 150, 40, DARKGRAY);
    
    if (level == 1) {
        DrawRectangleRec(replaybtn, GREEN);
        DrawText("Replay", replaybtn.x + replaybtn.width/2 - MeasureText("Replay", 30)/2, 
                 replaybtn.y + replaybtn.height/2 - 15, 30, WHITE);
        
        DrawRectangleRec(nextbtn, PURPLE);
        DrawText("Level 2", nextbtn.x + nextbtn.width/2 - MeasureText("Level 2", 30)/2, 
                 nextbtn.y + nextbtn.height/2 - 15, 30, WHITE);
    } else if (level == 2) {
        DrawRectangleRec(replaybtn, GREEN);
        DrawText("Replay", replaybtn.x + replaybtn.width/2 - MeasureText("Replay", 30)/2, 
                 replaybtn.y + replaybtn.height/2 - 15, 30, WHITE);
    }
    
    DrawRectangleRec(menubtn, RED);
    DrawText("Menu", menubtn.x + menubtn.width/2 - MeasureText("Menu", 20)/2, 
             menubtn.y + menubtn.height/2 - 10, 20, WHITE);
}

int main() {
    SetConfigFlags(FLAG_VSYNC_HINT);
    InitWindow(screenwidth, screenheight, "Rope Runners");
    SetTargetFPS(60);

    gamestate state = mainmenu;
    int level = 1;
    
    Rectangle playbtn = { screenwidth/2 - 100, screenheight/2 - 25, 200, 50 };
    Rectangle quitbtn = { screenwidth - 90, 10, 80, 40 };
    Rectangle menubtn = { screenwidth - 90, 10, 80, 40 };
    Rectangle replaybtn = { screenwidth/2 - 150, screenheight/2 - 25, 120, 50 };
    Rectangle nextbtn = { screenwidth/2 + 30, screenheight/2 - 25, 120, 50 };
    Rectangle level1btn = { screenwidth/2 - 150, screenheight/2 - 25, 120, 50 };
    Rectangle level1selectbtn = { screenwidth/2 - 150, screenheight/2 - 25, 120, 50 };
    Rectangle level2btn = { screenwidth/2 + 30, screenheight/2 - 25, 120, 50 };

    player blueguy = { {screenwidth / 3, screenheight - 60}, 0, false, false, 0 };
    player redguy = { {screenwidth / 2, screenheight - 60}, 0, false, false, 0 };
    bool pullingblue = false;
    bool pullingred = false;
    float pulltime = 0;

    //level 1 platforms
    platform level1platforms[maxplatforms] = {
        { {100, 520, 100, 10}, false, false },
        { {220, 450, 100, 10}, true, false },
        { {340, 380, 100, 10}, true, false },
        { {460, 310, 100, 10}, false, false },
        { {600, 250, 120, 10}, false, false },
        { {460, 180, 100, 10}, false, true },
        { {320, 180, 80, 10}, false, false },
        { {160, 180, 100, 10}, false, true },
        { {-100, 180, 200, 10}, false, false }
    };
    Rectangle level1exit = { 0, 120, 100, 60 };

    //level 2 platforms
    platform level2platforms[maxplatforms] = {
        { {50, 520, 100, 10}, false, false },
        { {220, 450, 50, 10}, false, true },
        { {340, 450, 50, 10}, false, true },
        { {460, 380, 100, 10}, false, false },
        { {600, 310, 50, 10}, true, false },
        { {500, 240, 50, 10}, true, false },
        { {400, 170, 50, 10}, true, false },
        { {200, 200, 100, 10}, false, false },
        { {70, 170, 70, 10}, false, true },
        { {220, 90, 50, 10}, false, true },
        { {0, 80, 100, 10}, false, false },
    };
    Rectangle level2exit = { 0, 20, 100, 60 };

    while (!WindowShouldClose()) {
        BeginDrawing();

        switch (state) {
            case mainmenu:
                menu(playbtn);
                if (CheckCollisionPointRec(GetMousePosition(), playbtn) && IsMouseButtonPressed(MOUSE_LEFT_BUTTON)) {
                    state = chooselevel;
                }
                break;

            case chooselevel:
                levelselect(level1selectbtn, level2btn, menubtn);
                if (CheckCollisionPointRec(GetMousePosition(), level1selectbtn) && IsMouseButtonPressed(MOUSE_LEFT_BUTTON)) {
                    state = gameplay;
                    level = 1;
                    blueguy = (player){ {screenwidth / 3, screenheight - 60}, 0, false, false, 0 };
                    redguy = (player){ {screenwidth / 2, screenheight - 60}, 0, false, false, 0 };
                }
                if (CheckCollisionPointRec(GetMousePosition(), level2btn) && IsMouseButtonPressed(MOUSE_LEFT_BUTTON)) {
                    state = gameplay;
                    level = 2;
                    blueguy = (player){ {screenwidth / 3, screenheight - 60}, 0, false, false, 0 };
                    redguy = (player){ {screenwidth / 2, screenheight - 60}, 0, false, false, 0 };
                }
                if (CheckCollisionPointRec(GetMousePosition(), menubtn) && IsMouseButtonPressed(MOUSE_LEFT_BUTTON)) {
                    state = mainmenu;
                }
                break;

            case gameplay:
                blueguy.moving = false;
                redguy.moving = false;

                //moving controls
                if (IsKeyDown(KEY_D)) {
                    blueguy.pos.x += movespeed;
                    blueguy.moving = true;
                }
                if (IsKeyDown(KEY_A)) {
                    blueguy.pos.x -= movespeed;
                    blueguy.moving = true;
                }
                if (IsKeyPressed(KEY_SPACE) && !blueguy.inair) {
                    blueguy.yspeed = -jumpforce;
                    blueguy.inair = true;
                }

                if (IsKeyDown(KEY_RIGHT)) {
                    redguy.pos.x += movespeed;
                    redguy.moving = true;
                }
                if (IsKeyDown(KEY_LEFT)) {
                    redguy.pos.x -= movespeed;
                    redguy.moving = true;
                }
                if (IsKeyPressed(KEY_UP) && !redguy.inair) {
                    redguy.yspeed = -jumpforce;
                    redguy.inair = true;
                }

                //gravity application
                blueguy.yspeed += gravity;
                redguy.yspeed += gravity;

                //update positioning
                blueguy.pos.y += blueguy.yspeed;
                redguy.pos.y += redguy.yspeed;

                //collision detection
                Rectangle bluehead = { blueguy.pos.x - 5, blueguy.pos.y - 10, 10, 10 };
                Rectangle redhead = { redguy.pos.x - 5, redguy.pos.y - 10, 10, 10 };
                Rectangle bluefeet = { blueguy.pos.x - 5, blueguy.pos.y + 60, 10, 5 };
                Rectangle redfeet = { redguy.pos.x - 5, redguy.pos.y + 60, 10, 5 };

                platform* platforms = (level == 1) ? level1platforms : level2platforms;
                int platformcount = (level == 1) ? 50 : 50;
                Rectangle exit = (level == 1) ? level1exit : level2exit;

                for (int i = 0; i < platformcount; i++) {
                    if (CheckCollisionRecs(bluehead, platforms[i].area) && blueguy.yspeed < 0) {
                        blueguy.pos.y = platforms[i].area.y + platforms[i].area.height + 10;
                        blueguy.yspeed = 0;
                    }
                    if (CheckCollisionRecs(redhead, platforms[i].area) && redguy.yspeed < 0) {
                        redguy.pos.y = platforms[i].area.y + platforms[i].area.height + 10;
                        redguy.yspeed = 0;
                    }

                    if (!platforms[i].redonly && CheckCollisionRecs(bluefeet, platforms[i].area) && blueguy.yspeed > 0) {
                        blueguy.pos.y = platforms[i].area.y - 60;
                        blueguy.yspeed = 0;
                        blueguy.inair = false;
                    }
                    if (!platforms[i].blueonly && CheckCollisionRecs(redfeet, platforms[i].area) && redguy.yspeed > 0) {
                        redguy.pos.y = platforms[i].area.y - 60;
                        redguy.yspeed = 0;
                        redguy.inair = false;
                    }
                }

                //ground collision
                if (blueguy.pos.y >= screenheight - 60) {
                    blueguy.pos.y = screenheight - 60;
                    blueguy.yspeed = 0;
                    blueguy.inair = false;
                }
                if (redguy.pos.y >= screenheight - 60) {
                    redguy.pos.y = screenheight - 60;
                    redguy.yspeed = 0;
                    redguy.inair = false;
                }

                //rope mechanics
                if (IsKeyPressed(KEY_F)) {
                    pullingred = true;
                    pulltime = 0;
                }
                if (pullingred) {
                    pulltime += GetFrameTime();
                    float xdiff = blueguy.pos.x - redguy.pos.x;
                    float ydiff = blueguy.pos.y - redguy.pos.y;
                    float distance = sqrtf(xdiff * xdiff + ydiff * ydiff);

                    if (distance > minropedistance) {
                        float directionx = xdiff / distance;
                        float directiony = ydiff / distance;
                        redguy.pos.x += directionx * pullpower;
                        redguy.pos.y += directiony * pullpower;
                    } else {
                        pullingred = false;
                    }
                    if (pulltime > 2.0f) pullingred = false;
                }

                if (IsKeyPressed(KEY_L)) {
                    pullingblue = true;
                    pulltime = 0;
                }
                if (pullingblue) {
                    pulltime += GetFrameTime();
                    float xdiff = redguy.pos.x - blueguy.pos.x;
                    float ydiff = redguy.pos.y - blueguy.pos.y;
                    float distance = sqrtf(xdiff * xdiff + ydiff * ydiff);

                    if (distance > minropedistance) {
                        float directionx = xdiff / distance;
                        float directiony = ydiff / distance;
                        blueguy.pos.x += directionx * pullpower;
                        blueguy.pos.y += directiony * pullpower;
                    } else {
                        pullingblue = false;
                    }
                    if (pulltime > 2.0f) pullingblue = false;
                }

                //animation
                int blueframe = (blueguy.framecount / framespeed) % 2;
                int bluelegoffset = (blueframe == 0) ? 5 : -5;
                int bluearmoffset = (blueframe == 0) ? -5 : 5;

                int redframe = (redguy.framecount / framespeed) % 2;
                int redlegoffset = (redframe == 0) ? 5 : -5;
                int redarmoffset = (redframe == 0) ? -5 : 5;

                if (blueguy.moving || blueguy.inair) blueguy.framecount++;
                if (redguy.moving || redguy.inair) redguy.framecount++;

                //level completion requirements
                Rectangle bluebody = { blueguy.pos.x - 10, blueguy.pos.y, 20, 60 };
                Rectangle redbody = { redguy.pos.x - 10, redguy.pos.y, 20, 60 };
                if (CheckCollisionRecs(bluebody, exit) && CheckCollisionRecs(redbody, exit)) {
                    state = leveldone;
                }

                //environment
                ClearBackground(WHITE);
                DrawRectangle(0, screenheight - 50, screenwidth, 50, DARKGREEN);

                for (int i = 0; i < platformcount; i++) {
                    if (platforms[i].blueonly) DrawRectangleRec(platforms[i].area, BLUE);
                    else if (platforms[i].redonly) DrawRectangleRec(platforms[i].area, RED);
                    else DrawRectangleRec(platforms[i].area, GRAY);
                }

                DrawRectangleLinesEx(exit, 4, GREEN);

                //rope
                DrawLineV((Vector2){blueguy.pos.x, blueguy.pos.y + 20},
                         (Vector2){redguy.pos.x, redguy.pos.y + 20}, BLACK);

                //blue guy
                DrawCircleV(blueguy.pos, 10, BLUE);
                DrawLineV((Vector2){blueguy.pos.x, blueguy.pos.y + 10},
                         (Vector2){blueguy.pos.x, blueguy.pos.y + 40}, BLUE);
                DrawLineV((Vector2){blueguy.pos.x, blueguy.pos.y + 20},
                         (Vector2){blueguy.pos.x - 15, blueguy.pos.y + 30 + bluearmoffset}, BLUE);
                DrawLineV((Vector2){blueguy.pos.x, blueguy.pos.y + 20},
                         (Vector2){blueguy.pos.x + 15, blueguy.pos.y + 30 - bluearmoffset}, BLUE);
                DrawLineV((Vector2){blueguy.pos.x, blueguy.pos.y + 40},
                         (Vector2){blueguy.pos.x - 10, blueguy.pos.y + 50 + bluelegoffset}, BLUE);
                DrawLineV((Vector2){blueguy.pos.x, blueguy.pos.y + 40},
                         (Vector2){blueguy.pos.x + 10, blueguy.pos.y + 50 - bluelegoffset}, BLUE);

                //red guy
                DrawCircleV(redguy.pos, 10, RED);
                DrawLineV((Vector2){redguy.pos.x, redguy.pos.y + 10},
                         (Vector2){redguy.pos.x, redguy.pos.y + 40}, RED);
                DrawLineV((Vector2){redguy.pos.x, redguy.pos.y + 20},
                         (Vector2){redguy.pos.x - 15, redguy.pos.y + 30 + redarmoffset}, RED);
                DrawLineV((Vector2){redguy.pos.x, redguy.pos.y + 20},
                         (Vector2){redguy.pos.x + 15, redguy.pos.y + 30 - redarmoffset}, RED);
                DrawLineV((Vector2){redguy.pos.x, redguy.pos.y + 40},
                         (Vector2){redguy.pos.x - 10, redguy.pos.y + 50 + redlegoffset}, RED);
                DrawLineV((Vector2){redguy.pos.x, redguy.pos.y + 40},
                         (Vector2){redguy.pos.x + 10, redguy.pos.y + 50 - redlegoffset}, RED);

                //quit button
                DrawRectangleRec(quitbtn, RED);
                DrawText("Quit", quitbtn.x + quitbtn.width/2 - MeasureText("Quit", 20)/2, 
                        quitbtn.y + quitbtn.height/2 - 10, 20, WHITE);

                if (CheckCollisionPointRec(GetMousePosition(), quitbtn) && IsMouseButtonPressed(MOUSE_LEFT_BUTTON)) {
                    state = mainmenu;
                    blueguy = (player){ {screenwidth / 3, screenheight - 60}, 0, false, false, 0 };
                    redguy = (player){ {screenwidth / 2, screenheight - 60}, 0, false, false, 0 };
                    pullingblue = false;
                    pullingred = false;
                    pulltime = 0;
                }
                break;

            case leveldone:
                levelcomplete(replaybtn, nextbtn, menubtn, level1btn, level);
                
                if (CheckCollisionPointRec(GetMousePosition(), replaybtn) && IsMouseButtonPressed(MOUSE_LEFT_BUTTON)) {
                    state = gameplay;
                    blueguy = (player){ {screenwidth / 3, screenheight - 60}, 0, false, false, 0 };
                    redguy = (player){ {screenwidth / 2, screenheight - 60}, 0, false, false, 0 };
                    pullingblue = false;
                    pullingred = false;
                    pulltime = 0;
                }
                if (level == 1 && CheckCollisionPointRec(GetMousePosition(), nextbtn) && IsMouseButtonPressed(MOUSE_LEFT_BUTTON)) {
                    state = gameplay;
                    level = 2;
                    blueguy = (player){ {screenwidth / 3, screenheight - 60}, 0, false, false, 0 };
                    redguy = (player){ {screenwidth / 2, screenheight - 60}, 0, false, false, 0 };
                    pullingblue = false;
                    pullingred = false;
                    pulltime = 0;
                }
                if (level == 2 && CheckCollisionPointRec(GetMousePosition(), level1btn) && IsMouseButtonPressed(MOUSE_LEFT_BUTTON)) {
                    state = gameplay;
                    level = 1;
                    blueguy = (player){ {screenwidth / 3, screenheight - 60}, 0, false, false, 0 };
                    redguy = (player){ {screenwidth / 2, screenheight - 60}, 0, false, false, 0 };
                    pullingblue = false;
                    pullingred = false;
                    pulltime = 0;
                }
                if (CheckCollisionPointRec(GetMousePosition(), menubtn) && IsMouseButtonPressed(MOUSE_LEFT_BUTTON)) {
                    state = mainmenu;
                    blueguy = (player){ {screenwidth / 3, screenheight - 60}, 0, false, false, 0 };
                    redguy = (player){ {screenwidth / 2, screenheight - 60}, 0, false, false, 0 };
                    pullingblue = false;
                    pullingred = false;
                    pulltime = 0;
                }
                break;
        }

        EndDrawing();
    }

    CloseWindow();
    return 0;
}