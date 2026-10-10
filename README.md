
import tkinter as tk
from random import randint, choice

# ============================================================
# ЛОВИ КВАДРАТ ULTIMATE — КОМПАКТНА ВЕРСІЯ
# ============================================================

WIDTH = 700
HEIGHT = 300

score = 0
target = 100
time_left = 30
lives = 3
combo = 0
multiplier = 1
coins = 0
xp = 0
player_level = 1
best_score = 0

game_running = False
frozen = False
current_square = None

game_timer = None
square_timer = None
freeze_timer = None

window = tk.Tk()
window.title("ЛОВИ КВАДРАТ ULTIMATE")
window.geometry("800x710")
window.resizable(False, False)
window.configure(bg="#0b1120")


# ============================================================
# СКЛАДНІСТЬ
# ============================================================

def difficulty_settings():
    settings = {
        "Легко": (40, 45, 1000, 65),
        "Нормально": (80, 35, 700, 55),
        "Складно": (150, 30, 500, 48),
        "Хардкор": (300, 25, 350, 42),
        "НЕМОЖЛИВО": (500, 20, 220, 35)
    }
    return settings.get(difficulty.get(), settings["Нормально"])


# ============================================================
# ОНОВЛЕННЯ ІНТЕРФЕЙСУ
# ============================================================

def update_interface():
    score_label.config(text=f"🏆 {score}")
    target_label.config(text=f"🎯 {target}")
    time_label.config(text=f"⏱ {time_left}")
    lives_label.config(text=f"❤️ {lives}")
    combo_label.config(text=f"🔥 {combo}")
    multiplier_label.config(text=f"×{multiplier}")
    coins_label.config(text=f"💰 {coins}")

    level_label.config(
        text=f"РІВЕНЬ {player_level}    XP: {xp}/100"
    )

    progress_label.config(
        text=f"Прогрес: {score} / {target}"
    )

    progress_bar.delete("all")
    progress_bar.create_rectangle(
        0, 0, 550, 20,
        fill="#1e293b", outline=""
    )

    progress = min(score / max(target, 1), 1)

    progress_bar.create_rectangle(
        0, 0, int(550 * progress), 20,
        fill="#22c55e", outline=""
    )


# ============================================================
# ТАЙМЕРИ
# ============================================================

def cancel_timers():
    global game_timer, square_timer, freeze_timer

    for timer in (game_timer, square_timer, freeze_timer):
        if timer is not None:
            try:
                window.after_cancel(timer)
            except tk.TclError:
                pass

    game_timer = None
    square_timer = None
    freeze_timer = None


# ============================================================
# СТВОРЕННЯ КВАДРАТА
# ============================================================

def create_square():
    global square_timer, current_square

    if not game_running:
        return

    if frozen:
        square_timer = window.after(200, create_square)
        return

    canvas.delete("target")
    current_square = None

    _, _, speed, size = difficulty_settings()

    x = randint(10, WIDTH - size - 10)
    y = randint(10, HEIGHT - size - 10)

    kind = randint(1, 100)

    if kind <= 10:
        color = "#facc15"
        square_type = "bonus"
    elif kind <= 17:
        color = "#dc2626"
        square_type = "danger"
    else:
        color = choice([
            "#3b82f6", "#8b5cf6", "#06b6d4",
            "#22c55e", "#ec4899"
        ])
        square_type = "normal"

    current_square = canvas.create_rectangle(
        x, y, x + size, y + size,
        fill=color,
        outline="white",
        width=3,
        tags=("target", square_type)
    )

    canvas.tag_bind("target", "<Button-1>", square_clicked)

    square_timer = window.after(speed, create_square)


# ============================================================
# АНІМАЦІЯ ОЧОК
# ============================================================

def floating_text(x, y, text, color):
    item = canvas.create_text(
        x, y,
        text=text,
        fill=color,
        font=("Arial", 16, "bold")
    )

    def animate(step=0):
        if step >= 10:
            canvas.delete(item)
            return

        canvas.move(item, 0, -3)
        window.after(30, lambda: animate(step + 1))

    animate()


# ============================================================
# КЛІК ПО КВАДРАТУ
# ============================================================

def square_clicked(event):
    global score, combo, multiplier, coins, xp
    global player_level, time_left, lives, current_square

    if not game_running or current_square is None:
        return

    clicked = canvas.find_withtag("current")

    if current_square not in clicked:
        return

    tags = canvas.gettags(current_square)
    coords = canvas.coords(current_square)
    x, y = coords[0], coords[1]

    canvas.delete(current_square)
    current_square = None

    if "danger" in tags:
        lives -= 1
        combo = 0
        multiplier = 1

        floating_text(x, y, "-1 ❤️", "#ef4444")
        message.config(
            text="💥 НЕБЕЗПЕЧНИЙ КВАДРАТ! -1 ЖИТТЯ",
            fg="#ef4444"
        )

        update_interface()

        if lives <= 0:
            lose_game()

        return

    if "bonus" in tags:
        points = 10 * multiplier

        score += points
        coins += 5
        xp += 10
        time_left += 2
        combo += 2

        floating_text(x, y, f"+{points} ⭐", "#facc15")

        message.config(
            text=f"⭐ БОНУС! +{points} очок, +2 сек!",
            fg="#facc15"
        )

    else:
        combo += 1

        if combo >= 30:
            multiplier = 10
        elif combo >= 20:
            multiplier = 7
        elif combo >= 15:
            multiplier = 5
        elif combo >= 10:
            multiplier = 3
        elif combo >= 5:
            multiplier = 2
        else:
            multiplier = 1

        points = multiplier
        score += points
        coins += 1
        xp += 3

        floating_text(x, y, f"+{points}", "#22c55e")

        message.config(
            text=f"🎯 +{points} очок! COMBO ×{multiplier}",
            fg="#22c55e"
        )

    while xp >= 100:
        xp -= 100
        player_level += 1
        message.config(
            text=f"🆙 НОВИЙ РІВЕНЬ {player_level}!",
            fg="#38bdf8"
        )

    update_interface()

    if score >= target:
        win_game()


# ============================================================
# ТАЙМЕР ГРИ
# ============================================================

def countdown():
    global time_left, game_timer

    if not game_running:
        return

    if not frozen:
        time_left -= 1

    update_interface()

    if time_left <= 0:
        time_left = 0
        update_interface()
        lose_game()
        return

    game_timer = window.after(1000, countdown)


# ============================================================
# ПОЧАТОК І ПЕРЕЗАПУСК
# ============================================================

def start_game():
    global score, target, time_left, lives
    global combo, multiplier, coins, xp
    global game_running, frozen, player_level, current_square

    cancel_timers()

    target, time_left, _, _ = difficulty_settings()

    score = 0
    lives = 3
    combo = 0
    multiplier = 1
    coins = 0
    xp = 0
    player_level = 1

    frozen = False
    game_running = True
    current_square = None

    canvas.delete("all")

    message.config(
        text="🔥 ГРУ РОЗПОЧАТО! ЛОВИ КВАДРАТИ!",
        fg="white"
    )

    update_interface()
    create_square()
    countdown()


# ============================================================
# ПЕРЕМОГА / ПОРАЗКА
# ============================================================

def show_end_screen(won):
    global game_running, best_score, current_square

    if not game_running:
        return

    game_running = False
    current_square = None
    cancel_timers()

    if score > best_score:
        best_score = score

    canvas.delete("all")

    if won:
        heading = "🏆 ПЕРЕМОГА!"
        color = "#22c55e"
        message.config(text="🎉 ТИ ПЕРЕМІГ! НАТИСНИ «ЗНОВУ»", fg=color)
    else:
        heading = "💀 ПОРАЗКА"
        color = "#ef4444"
        message.config(text="⏰ ГРУ ЗАВЕРШЕНО! СПРОБУЙ ЩЕ РАЗ", fg=color)

    canvas.create_text(
        WIDTH // 2, 65,
        text=heading,
        font=("Arial", 35, "bold"),
        fill=color
    )

    canvas.create_text(
        WIDTH // 2, 125,
        text=f"Очки: {score} / {target}",
        font=("Arial", 23, "bold"),
        fill="white"
    )

    canvas.create_text(
        WIDTH // 2, 180,
        text=f"💰 Монети: {coins}    🔥 Комбо: {combo}",
        font=("Arial", 17, "bold"),
        fill="#facc15"
    )

    canvas.create_text(
        WIDTH // 2, 225,
        text=f"🏅 Рекорд: {best_score}",
        font=("Arial", 18, "bold"),
        fill="#38bdf8"
    )

    canvas.create_text(
        WIDTH // 2, 270,
        text="Натисни кнопку «🔄 ЗНОВУ» внизу",
        font=("Arial", 15, "bold"),
        fill="#cbd5e1"
    )

    update_interface()


def win_game():
    show_end_screen(True)


def lose_game():
    show_end_screen(False)


# ============================================================
# ЗАМОРОЗКА
# ============================================================

def freeze_game():
    global frozen, freeze_timer

    if not game_running or frozen:
        return

    frozen = True

    message.config(
        text="❄️ ЗАМОРОЗКА НА 5 СЕКУНД!",
        fg="#38bdf8"
    )

    freeze_timer = window.after(5000, unfreeze_game)


def unfreeze_game():
    global frozen, freeze_timer

    freeze_timer = None
    frozen = False

    if game_running:
        message.config(
            text="🔥 ГРА ПРОДОВЖУЄТЬСЯ!",
            fg="white"
        )

        if current_square is None:
            create_square()


# ============================================================
# БОНУСНИЙ ЧАС
# ============================================================

def bonus_time():
    global time_left, score

    if not game_running:
        return

    time_left += 5
    score += 5

    message.config(
        text="⚡ +5 СЕКУНД І +5 ОЧОК!",
        fg="#facc15"
    )

    update_interface()

    if score >= target:
        win_game()


# ============================================================
# КЛАВІАТУРА
# ============================================================

def keyboard_control(event):
    key = event.keysym.lower()

    if key == "space":
        start_game()
    elif key == "r":
        start_game()
    elif key == "f":
        freeze_game()
    elif key == "b":
        bonus_time()


# ============================================================
# ЗАГОЛОВОК
# ============================================================

header = tk.Frame(window, bg="#0b1120")
header.pack(pady=3)

tk.Label(
    header,
    text="🎯 ЛОВИ КВАДРАТ",
    font=("Arial", 25, "bold"),
    bg="#0b1120",
    fg="white"
).pack()

tk.Label(
    header,
    text="ULTIMATE EDITION",
    font=("Arial", 10, "bold"),
    bg="#0b1120",
    fg="#64748b"
).pack()


# ============================================================
# СТАТИСТИКА
# ============================================================

stats = tk.Frame(window, bg="#0b1120")
stats.pack(pady=3)

score_label = tk.Label(stats, text="🏆 0", font=("Arial", 12, "bold"),
                       bg="#0b1120", fg="#22c55e")
score_label.grid(row=0, column=0, padx=8)

target_label = tk.Label(stats, text="🎯 100", font=("Arial", 12, "bold"),
                        bg="#0b1120", fg="#38bdf8")
target_label.grid(row=0, column=1, padx=8)

time_label = tk.Label(stats, text="⏱ 30", font=("Arial", 12, "bold"),
                      bg="#0b1120", fg="#facc15")
time_label.grid(row=0, column=2, padx=8)

lives_label = tk.Label(stats, text="❤️ 3", font=("Arial", 12, "bold"),
                       bg="#0b1120", fg="#fb7185")
lives_label.grid(row=0, column=3, padx=8)

combo_label = tk.Label(stats, text="🔥 0", font=("Arial", 12, "bold"),
                       bg="#0b1120", fg="#fb7185")
combo_label.grid(row=0, column=4, padx=8)

multiplier_label = tk.Label(stats, text="×1", font=("Arial", 13, "bold"),
                            bg="#0b1120", fg="#a855f7")
multiplier_label.grid(row=0, column=5, padx=8)

coins_label = tk.Label(stats, text="💰 0", font=("Arial", 12, "bold"),
                       bg="#0b1120", fg="#facc15")
coins_label.grid(row=0, column=6, padx=8)


# ============================================================
# РІВЕНЬ І СКЛАДНІСТЬ
# ============================================================

level_label = tk.Label(
    window,
    text="РІВЕНЬ 1    XP: 0/100",
    font=("Arial", 11, "bold"),
    bg="#0b1120",
    fg="#38bdf8"
)
level_label.pack(pady=2)

difficulty_frame = tk.Frame(window, bg="#0b1120")
difficulty_frame.pack(pady=2)

tk.Label(
    difficulty_frame,
    text="Складність:",
    font=("Arial", 11, "bold"),
    bg="#0b1120",
    fg="white"
).pack(side="left")

difficulty = tk.StringVar(value="Нормально")

difficulty_menu = tk.OptionMenu(
    difficulty_frame,
    difficulty,
    "Легко", "Нормально", "Складно", "Хардкор", "НЕМОЖЛИВО"
)
difficulty_menu.config(
    font=("Arial", 10, "bold"),
    bg="#334155",
    fg="white",
    width=12
)
difficulty_menu.pack(side="left", padx=5)


# ============================================================
# ІГРОВЕ ПОЛЕ
# ============================================================

canvas = tk.Canvas(
    window,
    width=WIDTH,
    height=HEIGHT,
    bg="#111827",
    highlightthickness=2,
    highlightbackground="#334155"
)
canvas.pack(pady=4)


# ============================================================
# ПРОГРЕС
# ============================================================

progress_label = tk.Label(
    window,
    text="Прогрес: 0 / 100",
    font=("Arial", 10, "bold"),
    bg="#0b1120",
    fg="white"
)
progress_label.pack()

progress_bar = tk.Canvas(
    window,
    width=550,
    height=20,
    bg="#1e293b",
    highlightthickness=0
)
progress_bar.pack(pady=2)


# ============================================================
# ПОВІДОМЛЕННЯ
# ============================================================

message = tk.Label(
    window,
    text="Натисни «ПОЧАТИ»",
    font=("Arial", 11, "bold"),
    bg="#0b1120",
    fg="white"
)
message.pack(pady=2)


# ============================================================
# КНОПКИ
# ============================================================

button_frame = tk.Frame(window, bg="#0b1120")
button_frame.pack(pady=3)

tk.Button(
    button_frame,
    text="▶ ПОЧАТИ",
    font=("Arial", 10, "bold"),
    bg="#16a34a",
    fg="white",
    width=13,
    height=2,
    command=start_game
).grid(row=0, column=0, padx=4)

tk.Button(
    button_frame,
    text="🔄 ЗНОВУ",
    font=("Arial", 10, "bold"),
    bg="#2563eb",
    fg="white",
    width=13,
    height=2,
    command=start_game
).grid(row=0, column=1, padx=4)

tk.Button(
    button_frame,
    text="❄️ ЗАМОРОЗКА",
    font=("Arial", 10, "bold"),
    bg="#0891b2",
    fg="white",
    width=13,
    height=2,
    command=freeze_game
).grid(row=0, column=2, padx=4)

tk.Button(
    button_frame,
    text="⚡ +5 СЕКУНД",
    font=("Arial", 10, "bold"),
    bg="#d97706",
    fg="white",
    width=13,
    height=2,
    command=bonus_time
).grid(row=0, column=3, padx=4)


# ============================================================
# ПІДКАЗКА
# ============================================================

tk.Label(
    window,
    text="SPACE — почати | R — знову | F — заморозка | B — бонус",
    font=("Arial", 9),
    bg="#0b1120",
    fg="#64748b"
).pack(pady=3)


# ============================================================
# ЗАПУСК
# ============================================================

window.bind("<Key>", keyboard_control)

update_interface()
window.mainloop()
