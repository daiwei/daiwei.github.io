---
layout: post
title: 【游戏】Linux命令行游戏 -- 俄罗斯方块
---

我在Ubuntu 10.04下测试过，可以正常运行。不过界面让人蛋疼。

代码用到了NCURSES库。编译的时候链一下ncurses库就可以了，如：

```
cc -Wall -O2 -o c01 file.c -lncurses
```

界面：

![tetris_cli_001](/assets/img/tetris_cli_001.gif)

```
/***************************************
 *
 * TETRIS
 *
 * author: dave
 * date  : 2010/07/28
 *
 ***************************************/
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <sys/select.h>
#include <sys/time.h>
#include <sys/types.h>
#include <unistd.h>
#include <ncurses.h>

#define TETRADS_LEN   4
#define GAMEWIN_YLEN  20
#define GAMEWIN_XLEN  10
#define GAMEWIN_Y     1
#define GAMEWIN_X     2
#define INFOWIN_YLEN  10
#define INFOWIN_XLEN  10
#define INFOWIN_Y     GAMEWIN_Y
#define INFOWIN_X     (GAMEWIN_X + GAMEWIN_XLEN*2 + 5)
#define PIC_BLOCK     '#'
#define PIC_NULL      ' '

#define _X(x)         ((x)*2)

#define BASEWIN \
    WINDOW *win; \
    void   (*init)(); \
    void   (*show)();

#define EXCHANGE_XY(_pos) \
    do { \
        (_pos).x = (_pos).y + (_pos).x; \
        (_pos).y = (_pos).x - (_pos).y; \
        (_pos).x -= (_pos).y; \
    } while (0)

#define EXCHANGE_2Y(_pos) \
        (_pos).y = 2 - (_pos).y

#define COPY_TETRADS(_dest, _src) \
    memcpy(&(_dest), &(_src), sizeof(Tetrads))

#define TETRISNEW(_p, _t) \
    do { \
        (_p) = (_t*)malloc(sizeof(_t)); \
        (_p)->init = init##_t; \
        (_p)->show = show##_t; \
    } while (0)

#define TETRISDEL(_p) \
    do { \
        delwin(_p->win); \
        free(_p); \
    } while (0)

/* 俄罗斯方块的7种方块(Tetromino) */
typedef enum {
    TETRADS_S = 0,
    TETRADS_Z,
    TETRADS_L,
    TETRADS_J,
    TETRADS_O,
    TETRADS_T,
    TETRADS_I,

    TETRADS_MAX
} ETetrads;

typedef enum {
    DIR_UP = 0,
    DIR_DOWN,
    DIR_LEFT,
    DIR_RIGHT,

    DIR_MAX
} EDirction;

typedef enum {
    TETRIS_STATE_NEW,
    TETRIS_STATE_MOVE,
    TETRIS_STATE_STOP,

    TETRIS_STATE_MAX,
} ETetrisState;

typedef struct {
    int y;
    int x;
} Point;

typedef struct {
    ETetrads type;
    Point    blocks[TETRADS_LEN];
} Tetrads;

/*** BEGIN : 界面显示 ***/ /* 将界面显示与数据处理分离 */
typedef struct _BaseWin {
    /**
     * WINDOW *win;
     * void   (*init)();
     * void   (*show)();
     */
    BASEWIN
} BaseWin;

typedef struct _GameWin {
    /**
     * WINDOW *win;
     * void   (*init)();
     * void   (*show)();
     */
    BASEWIN

    char    background[GAMEWIN_YLEN][GAMEWIN_XLEN];
    char    matrix[GAMEWIN_YLEN][GAMEWIN_XLEN];
    int     level;
    Point   pos;
    Tetrads curTetrads;
} GameWin;

typedef struct _InfoWin {
    /**
     * WINDOW *win;
     * void   (*init)();
     * void   (*show)();
     */
    BASEWIN

    int     score;
    int     level;
    Tetrads nextTetrads;
} InfoWin;

void tetrisPrint(WINDOW *win, Point pos, Point block, char chr);

void initGameWin(GameWin* _self)
{
    _self->win = newwin(GAMEWIN_YLEN + 2, _X(GAMEWIN_XLEN) + 1, GAMEWIN_Y, GAMEWIN_X);
    box(_self->win, 0, 0);
    mvwprintw(_self->win, 0, (_X(GAMEWIN_XLEN) + 2)/2 - 3, "TETRIS");
    wrefresh(_self->win);

    memset(_self->background, PIC_NULL, GAMEWIN_YLEN * GAMEWIN_XLEN);
    memset(_self->matrix, PIC_NULL, GAMEWIN_YLEN * GAMEWIN_XLEN);

    _self->pos = (Point){1, GAMEWIN_XLEN/2};
    _self->level = 0;
}

void initInfoWin(InfoWin* _self)
{
    _self->win = newwin(INFOWIN_YLEN, _X(GAMEWIN_XLEN), INFOWIN_Y, INFOWIN_X);
    box(_self->win, 0, 0);
    mvwprintw(_self->win, 0, _X(GAMEWIN_XLEN)/2 - 2, "INFO");
    mvwprintw(_self->win, 7, 2, "SCORE: 0");
    mvwprintw(_self->win, 8, 2, "LEVEL: 0");
    wrefresh(_self->win);

    _self->score = 0;
    _self->level = 0;
}

void showGameWin(GameWin* _self)
{
    int y = 0;
    int x = 0;

    for (y = 0; y < GAMEWIN_YLEN; ++y)
        for (x = 0; x < GAMEWIN_XLEN; ++x)
            tetrisPrint(_self->win, (Point){0, 0}, (Point){y, x}, (_self->background)[y][x]);

    for (y = 0; y < GAMEWIN_YLEN; ++y)
        for (x = 0; x < GAMEWIN_XLEN; ++x)
            tetrisPrint(_self->win, (Point){0, 0}, (Point){y, x}, (_self->matrix)[y][x]);

    for (x = 0; x < TETRADS_LEN; ++x)
        tetrisPrint(_self->win, (Point){(_self->pos).y, (_self->pos).x}, (_self->curTetrads).blocks[x], PIC_BLOCK);

    wrefresh(_self->win);
}

void showInfoWin(InfoWin* _self)
{
    int i = 0;

    mvwprintw(_self->win, 2, _X(INFOWIN_XLEN/2 - 2), "        ");
    mvwprintw(_self->win, 3, _X(INFOWIN_XLEN/2 - 2), "        ");
    mvwprintw(_self->win, 4, _X(INFOWIN_XLEN/2 - 2), "        ");
    mvwprintw(_self->win, 5, _X(INFOWIN_XLEN/2 - 2), "        ");
    mvwprintw(_self->win, 6, _X(INFOWIN_XLEN/2 - 2), "        ");

    for (i = 0; i < TETRADS_LEN; ++i)
        tetrisPrint(_self->win, (Point){2, INFOWIN_XLEN/2 - 2}, _self->nextTetrads.blocks[i], PIC_BLOCK);

    mvwprintw(_self->win, INFOWIN_YLEN - 3, 2, "SCORE: %d", _self->score);
    mvwprintw(_self->win, INFOWIN_YLEN - 2, 2, "LEVEL: %d", _self->level);

    wrefresh(_self->win);
}
/*** END   : 界面显示 ***/

/*** 函数声明 ***/
void newTetrads(Tetrads *tetrads);
void initTetrads(Tetrads *tetrads);
void spinTetrads(Tetrads *tetrads);
int  runTetris(GameWin *gwin);
int  checkBorder(GameWin *gwin);
int  checkStop(GameWin *gwin);
int  checkOver(GameWin *gwin);
int  checkClean(GameWin *gwin);
void refreshMatrix(GameWin *gwin);
int  genRandom(int max);

int main()
{
    /* init ncurses screen */
    initscr();
    raw();
    noecho();
    keypad(stdscr, TRUE);
    curs_set(0);
    refresh();

    /* 初始化界面 */
    GameWin *gwin;
    InfoWin *iwin;

    TETRISNEW(gwin, GameWin);
    TETRISNEW(iwin, InfoWin);

    gwin->init(gwin);
    iwin->init(iwin);

    /* Tetris的处理使用简单的状态机实现 */
    int f_end = 0;
    int state = TETRIS_STATE_NEW;

    newTetrads(&(iwin->nextTetrads));

    while (!f_end) {
        switch (state) {
        case TETRIS_STATE_NEW:
            COPY_TETRADS(gwin->curTetrads, iwin->nextTetrads);
            gwin->pos = (Point){1, 4};
            newTetrads(&(iwin->nextTetrads));

            iwin->show(iwin);

            state = TETRIS_STATE_MOVE;
            break;

        case TETRIS_STATE_MOVE:
            gwin->show(gwin);

            switch (runTetris(gwin)) {
            case -1:
                goto END;
                break;
            case 0:
                break;
            case 1:
                state = TETRIS_STATE_STOP;
                break;
            default:
                break;
            }

            break;

        case TETRIS_STATE_STOP:
            refreshMatrix(gwin);
            iwin->score = checkClean(gwin);
            state = TETRIS_STATE_NEW;
            break;

        default :
            f_end = 1;
            break;
        }
    }

END:
    mvwprintw(gwin->win, GAMEWIN_YLEN/2 - 2, 5, "GAME OVER!!!");
    mvwprintw(gwin->win, GAMEWIN_YLEN/2,     4, "Press any key");
    mvwprintw(gwin->win, GAMEWIN_YLEN/2 + 1, 6, "to quit...");
    wrefresh(gwin->win);

    getch();

    TETRISDEL(iwin);
    TETRISDEL(gwin);
    endwin();

    return 0;
}

void tetrisPrint(WINDOW *win, Point pos, Point block, char chr)
{
    mvwaddch(win, pos.y + block.y + 1, (pos.x + block.x) * 2 + 1, chr);
}

void newTetrads(Tetrads *tetrads)
{
    tetrads->type = genRandom(TETRADS_MAX);

    initTetrads(tetrads);


    int spin = genRandom(DIR_MAX);
    int i = 0;
    for (; i <= spin; ++i) {
        spinTetrads(tetrads);
    }
}

void initTetrads(Tetrads *tetrads)
{
    switch (tetrads->type) {
    case TETRADS_S:
        tetrads->blocks[0] = (Point){2, 0};
        tetrads->blocks[1] = (Point){2, 1};
        tetrads->blocks[2] = (Point){1, 1};
        tetrads->blocks[3] = (Point){1, 2};
        break;

    case TETRADS_Z:
        tetrads->blocks[0] = (Point){1, 0};
        tetrads->blocks[1] = (Point){1, 1};
        tetrads->blocks[2] = (Point){2, 1};
        tetrads->blocks[3] = (Point){2, 2};
        break;

    case TETRADS_L:
        tetrads->blocks[0] = (Point){2, 0};
        tetrads->blocks[1] = (Point){2, 1};
        tetrads->blocks[2] = (Point){1, 1};
        tetrads->blocks[3] = (Point){0, 1};
        break;

    case TETRADS_J:
        tetrads->blocks[0] = (Point){0, 0};
        tetrads->blocks[1] = (Point){1, 0};
        tetrads->blocks[2] = (Point){1, 1};
        tetrads->blocks[3] = (Point){1, 2};
        break;

    case TETRADS_O:
        tetrads->blocks[0] = (Point){0, 0};
        tetrads->blocks[1] = (Point){0, 1};
        tetrads->blocks[2] = (Point){1, 0};
        tetrads->blocks[3] = (Point){1, 1};
        break;

    case TETRADS_T:
        tetrads->blocks[0] = (Point){0, 1};
        tetrads->blocks[1] = (Point){1, 0};
        tetrads->blocks[2] = (Point){1, 1};
        tetrads->blocks[3] = (Point){1, 2};
        break;

    case TETRADS_I:
        tetrads->blocks[0] = (Point){0, 1};
        tetrads->blocks[1] = (Point){1, 1};
        tetrads->blocks[2] = (Point){2, 1};
        tetrads->blocks[3] = (Point){3, 1};
        break;

    default:
        break;
    }
}

/**
 * 旋转Tetrads
 */
void spinTetrads(Tetrads *tetrads)
{
    int i = 0;

    switch (tetrads->type) {
    case TETRADS_O:
        break;

    case TETRADS_I:
        /* x,y互换 */
        for (i = 0; i < TETRADS_LEN; ++i)
            EXCHANGE_XY(tetrads->blocks[i]);
        break;

    default:
        for (i = 0; i < TETRADS_LEN; ++i)
            EXCHANGE_XY(tetrads->blocks[i]);

        for (i = 0; i < TETRADS_LEN; ++i)
            EXCHANGE_2Y(tetrads->blocks[i]);

        break;
    }
}

/**
 * Return:
 *   -1    game over
 *    0    continue
 *    1    stop
 */
int runTetris(GameWin *gwin)
{
    int ret = 0;

    fd_set fset;

    FD_ZERO(&fset);
    FD_SET(0, &fset);

    struct timeval timeout;
    timeout.tv_sec  = 0;
    timeout.tv_usec = 500000 - 10000 * gwin->level;

    int fd = -1;
    if ((fd = select(1, &fset, NULL, NULL, &timeout)) <= 0) {
        ++((gwin->pos).y);
        while (checkStop(gwin) != 0) {
            --((gwin->pos).y);
            ret = 1;
        }

        if (ret == 1) {
            if (checkOver(gwin))
                return -1;

            return 1;
        }

        return 0;
    }

    Tetrads tmptetrads;
    char ch;
    int  n = 0;
    switch (ch = getch()) {
    case 'w':
        COPY_TETRADS(tmptetrads, gwin->curTetrads);
        spinTetrads(&gwin->curTetrads);

        while ((n = checkBorder(gwin)) != 0)
            (gwin->pos).x += n;

        while (checkStop(gwin) != 0) {
            --((gwin->pos).y);
            ret = 1;
        }

        return ret;

    case 's':
        ++((gwin->pos).y);
        while (checkStop(gwin) != 0) {
            --((gwin->pos).y);
            ret = 1;
        }

        if (ret == 1) {
            if (checkOver(gwin))
                return -1;

            return 1;
        }

        return 0;

    case 'a':
        --((gwin->pos).x);
        while ((n = checkBorder(gwin)) != 0)
            (gwin->pos).x += n;

        while (checkStop(gwin) != 0) {
            ++((gwin->pos).x);
            ret = 1;
        }

        break;

    case 'd':
        ++((gwin->pos).x);
        while ((n = checkBorder(gwin)) != 0)
            (gwin->pos).x += n;

        while (checkStop(gwin) != 0) {
            --((gwin->pos).x);
            ret = 1;
        }

        break;

    default:
        break;
    }

    return 0;
}

/**
 * 检查是否达到边线
 */
int checkBorder(GameWin *gwin)
{
    int i = 0;
    int n = 0;
    for (i = 0; i < TETRADS_LEN; ++i) {
        if ((n = ((gwin->pos).x + (gwin->curTetrads).blocks[i].x)) < 0)
            return -n;

        if ((n = ((gwin->pos).x + (gwin->curTetrads).blocks[i].x)) > GAMEWIN_XLEN - 1)
            return GAMEWIN_XLEN - 1 - n;
    }

    return 0;
}

/**
 * 检查是否停止
 */
int checkStop(GameWin *gwin)
{
    int i = 0;
    for (i = 0; i < TETRADS_LEN; ++i)
        if (gwin->matrix[(gwin->pos).y + (gwin->curTetrads).blocks[i].y][(gwin->pos).x + (gwin->curTetrads).blocks[i].x] == PIC_BLOCK
            || ((gwin->pos).y + (gwin->curTetrads).blocks[i].y) >= GAMEWIN_YLEN)
            return 1;

    return 0;
}

/**
 * 检查是否游戏结束
 */
int checkOver(GameWin *gwin)
{
    int i = 0;
    for (i = 0; i < TETRADS_LEN; ++i)
        if ((gwin->pos).y <= 0)
            return 1;

    return 0;
}

/**
 * 检查是否需要清楚一行
 */
int checkClean(GameWin *gwin)
{
    char bline[GAMEWIN_XLEN];
    memset(bline, PIC_BLOCK, GAMEWIN_XLEN);

    int i     = 0;
    int num   = 0;
    int score = 0;
    for (i = 0; i < TETRADS_LEN; ++i) {
        num = (gwin->pos).y + (gwin->curTetrads).blocks[i].y;
        if (strncmp(gwin->matrix[num], bline, GAMEWIN_XLEN) == 0) {
            score += 10;
            for (; num > 0; --num)
                memcpy(gwin->matrix[num], gwin->matrix[num-1], GAMEWIN_XLEN);
        }
    }

    return score;
}

void refreshMatrix(GameWin *gwin)
{
    int i = 0;
    for (i = 0; i < TETRADS_LEN; ++i)
        gwin->matrix[(gwin->pos).y + (gwin->curTetrads).blocks[i].y][(gwin->pos).x + (gwin->curTetrads).blocks[i].x] = PIC_BLOCK;
}

/**
 * 返回0~max-1的一个随机数
 */
int genRandom(int max)
{
    srandom((int)time(NULL));
    return (random() % max);
}
```
