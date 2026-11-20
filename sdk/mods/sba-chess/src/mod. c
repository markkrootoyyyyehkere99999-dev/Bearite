#include <stdlib.h>
#include <string.h>
#include <stdio.h>
#include "bearite_api.h"
#include "bearite_il2cpp.h"

typedef struct {
    float x, y, width, height;
} UnityRect;

static const BeariteApi* g_api = NULL;
static const char* g_mod_id = NULL;

/* 0=пусто, 1=Пешка, 2=Ладья, 3=Конь, 4=Слон, 5=Ферзь, 6=Король
   Положительные (>0) — Белые, Отрицательные (<0) — Чёрные */
static int board[8][8];
static int turn = 1;
static int sel_r = -1;
static int sel_c = -1;
static int board_open = 0;
static int initialized = 0;
static char status_text[64] = "Ход: БЕЛЫЕ";

static void* mi_gui_button = NULL;
static void* mi_gui_box = NULL;
static void* mi_screen_w = NULL;
static void* mi_screen_h = NULL;

static void reset_board(void) {
    int back_rank[8] = {2, 3, 4, 5, 6, 4, 3, 2};
    for (int r = 0; r < 8; r++) {
        for (int c = 0; c < 8; c++) {
            board[r][c] = 0;
        }
    }
    for (int c = 0; c < 8; c++) {
        board[0][c] = -back_rank[c];
        board[1][c] = -1;
        board[6][c] = 1;
        board[7][c] = back_rank[c];
    }
    turn = 1;
    sel_r = -1;
    sel_c = -1;
    snprintf(status_text, sizeof(status_text), "Ход: БЕЛЫЕ");
}

static const char* get_piece_label(int p) {
    switch (p) {
        case  1: return "П";
        case  2: return "Л";
        case  3: return "К";
        case  4: return "С";
        case  5: return "Ф";
        case  6: return "Кр";
        case -1: return "п";
        case -2: return "л";
        case -3: return "к";
        case -4: return "с";
        case -5: return "ф";
        case -6: return "кр";
        default: return "";
    }
}

static int is_path_clear(int r1, int c1, int r2, int c2) {
    int dr = (r2 > r1) ? 1 : (r2 < r1 ? -1 : 0);
    int dc = (c2 > c1) ? 1 : (c2 < c1 ? -1 : 0);
    int r = r1 + dr;
    int c = c1 + dc;
    while (r != r2 || c != c2) {
        if (board[r][c] != 0) return 0;
        r += dr;
        c += dc;
    }
    return 1;
}

static int can_move(int r1, int c1, int r2, int c2) {
    if (r1 == r2 && c1 == c2) return 0;
    int p = board[r1][c1];
    int target = board[r2][c2];
    if (p == 0) return 0;
    if ((p > 0 && target > 0) || (p < 0 && target < 0)) return 0;

    int dr = r2 - r1;
    int dc = c2 - c1;
    int adr = abs(dr);
    int adc = abs(dc);
    int type = abs(p);

    if (type == 1) {
        int dir = (p > 0) ? -1 : 1;
        int start_row = (p > 0) ? 6 : 1;
        if (dc == 0 && target == 0) {
            if (dr == dir) return 1;
            if (r1 == start_row && dr == 2 * dir && board[r1 + dir][c1] == 0) return 1;
        }
        if (adc == 1 && dr == dir && target != 0) return 1;
        return 0;
    }
    if (type == 2) return (dr == 0 || dc == 0) && is_path_clear(r1, c1, r2, c2);
    if (type == 3) return (adr == 2 && adc == 1) || (adr == 1 && adc == 2);
    if (type == 4) return (adr == adc) && is_path_clear(r1, c1, r2, c2);
    if (type == 5) return (dr == 0 || dc == 0 || adr == adc) && is_path_clear(r1, c1, r2, c2);
    if (type == 6) return adr <= 1 && adc <= 1;
    return 0;
}

static void on_cell_click(int r, int c) {
    int clicked = board[r][c];
    if (sel_r == -1) {
        if ((turn == 1 && clicked > 0) || (turn == -1 && clicked < 0)) {
            sel_r = r;
            sel_c = c;
        }
        return;
    }
    if ((turn == 1 && clicked > 0) || (turn == -1 && clicked < 0)) {
        if (sel_r == r && sel_c == c) {
            sel_r = -1;
            sel_c = -1;
        } else {
            sel_r = r;
            sel_c = c;
        }
        return;
    }
    if (can_move(sel_r, sel_c, r, c)) {
        int captured = board[r][c];
        board[r][c] = board[sel_r][sel_c];
        board[sel_r][sel_c] = 0;

        if (board[r][c] == 1 && r == 0) board[r][c] = 5;
        if (board[r][c] == -1 && r == 7) board[r][c] = -5;

        sel_r = -1;
        sel_c = -1;

        if (abs(captured) == 6) {
            snprintf(status_text, sizeof(status_text), "ПОБЕДА: %s!", turn == 1 ? "БЕЛЫЕ" : "ЧЁРНЫЕ");
            turn = 0;
        } else {
            turn = -turn;
            snprintf(status_text, sizeof(status_text), "Ход: %s", turn == 1 ? "БЕЛЫЕ" : "ЧЁРНЫЕ");
        }
    }
}

static int call_gui_button(UnityRect rect, const char* label) {
    if (!mi_gui_button || !bearite_il2.string_new) return 0;
    void* str = bearite_il2.string_new(label);
    void* args[2] = { &rect, str };
    void* res = bearite_invoke(mi_gui_button, NULL, args);
    if (!res) return 0;
    return *(unsigned char*)((char*)res + sizeof(void*) * 2);
}

static void call_gui_box(UnityRect rect, const char* label) {
    if (!mi_gui_box || !bearite_il2.string_new) return;
    void* str = bearite_il2.string_new(label);
    void* args[2] = { &rect, str };
    bearite_invoke(mi_gui_box, NULL, args);
}

static int get_screen_width(void) {
    if (!mi_screen_w) return 1280;
    void* res = bearite_invoke(mi_screen_w, NULL, NULL);
    return res ? *(int*)((char*)res + sizeof(void*) * 2) : 1280;
}

static int get_screen_height(void) {
    if (!mi_screen_h) return 720;
    void* res = bearite_invoke(mi_screen_h, NULL, NULL);
    return res ? *(int*)((char*)res + sizeof(void*) * 2) : 720;
}

static void draw_chess_ui(void) {
    if (!g_api || !g_mod_id) return;
    if (!g_api->get_bool(g_mod_id, "enabled", 1)) return;

    int sw = get_screen_width();
    int sh = get_screen_height();

    UnityRect toggle_rect = { (float)(sw - 220), 20.0f, 200.0f, 55.0f };
    if (call_gui_button(toggle_rect, board_open ? "Закрыть шахматы" : "Шахматы")) {
        board_open = !board_open;
    }

    if (!board_open) return;

    long long scale = g_api->get_int(g_mod_id, "board_scale", 100);
    if (scale < 50) scale = 50;
    if (scale > 150) scale = 150;

    float base_size = (sh < sw ? sh : sw) * 0.72f * (scale / 100.0f);
    float cell = base_size / 8.0f;
    float start_x = (sw - base_size) * 0.5f;
    float start_y = (sh - base_size) * 0.5f + 20.0f;

    UnityRect bg_rect = { start_x - 15.0f, start_y - 65.0f, base_size + 30.0f, base_size + 125.0f };
    call_gui_box(bg_rect, status_text);

    char cell_text[16];
    for (int r = 0; r < 8; r++) {
        for (int c = 0; c < 8; c++) {
            UnityRect r_cell = { start_x + c * cell, start_y + r * cell, cell - 2.0f, cell - 2.0f };
            const char* p_str = get_piece_label(board[r][c]);
            int is_dark = ((r + c) % 2 == 1);

            if (sel_r == r && sel_c == c) {
                snprintf(cell_text, sizeof(cell_text), "[%s]", p_str[0] ? p_str : "+");
            } else if (p_str[0]) {
                snprintf(cell_text, sizeof(cell_text), "%s", p_str);
            } else {
                snprintf(cell_text, sizeof(cell_text), "%s", is_dark ? "." : " ");
            }

            if (call_gui_button(r_cell, cell_text)) {
                on_cell_click(r, c);
            }
        }
    }

    UnityRect reset_rect = { start_x, start_y + base_size + 10.0f, base_size, 45.0f };
    if (call_gui_button(reset_rect, "Новая партия")) {
        reset_board();
    }
}

static void (*orig_gui_changed)(void);
static void my_gui_changed(void) {
    if (orig_gui_changed) orig_gui_changed();
    draw_chess_ui();
}

__attribute__((visibility("default")))
void bearite_mod_init(const BeariteApi* api, const char* mod_id) {
    if (api->version != BEARITE_API_VERSION) return;
    g_api = api;
    g_mod_id = mod_id;

    if (!initialized) {
        reset_board();
        initialized = 1;
    }

    if (bearite_il2cpp_init(api)) {
        mi_gui_button = bearite_find_method("UnityEngine", "GUI", "Button", 2);
        mi_gui_box    = bearite_find_method("UnityEngine", "GUI", "Box", 2);
        mi_screen_w   = bearite_find_method("UnityEngine", "Screen", "get_width", 0);
        mi_screen_h   = bearite_find_method("UnityEngine", "Screen", "get_height", 0);

        int hooked = bearite_hook_method(api, mod_id, "UnityEngine", "GUIUtility", "CheckOnGUI", 0,
                                         (void*)my_gui_changed, (void**)&orig_gui_changed);
        if (hooked == 0) {
            api->log(mod_id, "2D Шахматы успешно загружены!");
        } else {
            api->log(mod_id, "Не удалось привязаться к GUIUtility::CheckOnGUI");
        }
    }
}
