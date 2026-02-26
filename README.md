# === MEGAKINO TWITCH BOT (ABSOLUTE COMPLETE VERSION – ALL FEATURES + AUTO BOT JOIN) ===

import socket
import re
import time
import random
import threading
import json
import os
from datetime import datetime


# ================= CONFIG =================
TOKEN = "oauth:c7oe9nedlf9pdqr76chkzchzerujh3"
NICK = "megakino_tool"
CHANNEL = "#mega_kino"

SERVER = "irc.chat.twitch.tv"
PORT = 6667

CHAT_CLEAR_INTERVAL = 10 * 60
AUTO_RACE_TIMEOUT = 3 * 60  # 3 Minuten

PENDEL_INTERVAL = 15 * 60
PENDEL_DAUER = 30

XP_PER_MESSAGE = 5
XP_PER_LEVEL = 100

TRACK_LENGTH = 600
RACE_JOIN_TIME = 30
RACE_LAPS = 3
RACE_POINTS = 10

AI_MIN = 2
AI_MAX = 5

AI_BOTS = [
    "MaxRacer", "ShadowX", "LucaSpeed", "NightWolf",
    "NeoDrive", "BlazeKing", "FastFinn", "DarkRider",
    "ViperX", "StormOne", "Phantom", "IronWolf"
]

AI_SKILLS = {
    "Noob": {"min": 4, "max": 10, "turbo": 0.00},
    "Normal": {"min": 5, "max": 15, "turbo": 0.00},
    "Pro": {"min": 8, "max": 18, "turbo": 0.02},
    "Legend": {"min": 10, "max": 20, "turbo": 0.05}
}

LOG_FILE = "chat_log.txt"
STATS_FILE = "stats.json"
CMD_FILE = "dashboard_cmd.txt"


# ================= GLOBAL =================

stats = {}
race_active = False
race_join_open = False
race_players = {}
bets = {}

pendel_active = False
pendel_users = set()

last_g_time = time.time()


# ================= TERMINAL LOG =================

def terminal_log(tag, message):
    now = datetime.now().strftime("%H:%M:%S")
    print(f"[{now}] [{tag}] {message}")


# ================= CHAT LOG =================

def log_message(user, message):
    now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    terminal_log("CHAT", f"{user}: {message}")
    with open(LOG_FILE, "a", encoding="utf-8") as f:
        f.write(f"[{now}] {user}: {message}\n")


# ================= STATS =================

def load_stats():
    global stats
    try:
        with open(STATS_FILE, "r", encoding="utf-8") as f:
            stats = json.load(f)
    except:
        stats = {}

def save_stats():
    with open(STATS_FILE, "w", encoding="utf-8") as f:
        json.dump(stats, f, indent=2)

def update_user(user):
    if user not in stats:
        stats[user] = {
            "messages": 0,
            "xp": 0,
            "level": 1,
            "races_won": 0,
            "pendel_wins": 0,
            "points": 100,
            "last_seen": ""
        }
    stats[user]["messages"] += 1
    stats[user]["last_seen"] = datetime.now().strftime("%Y-%m-%d %H:%M")


# ================= XP =================

def add_xp(user):
    update_user(user)
    stats[user]["xp"] += XP_PER_MESSAGE
    if stats[user]["xp"] >= XP_PER_LEVEL:
        stats[user]["xp"] = 0
        stats[user]["level"] += 1
        send_chat(f"🎉 @{user} ist jetzt Level {stats[user]['level']}!")
    save_stats()


# ================= AUTO CLEAR =================

def auto_clear_chat():
    while True:
        time.sleep(CHAT_CLEAR_INTERVAL)
        send_chat("🧹 Chat wird geleert...")
        time.sleep(3)
        send_chat("/clear")


# ================= DASHBOARD =================

def dashboard_command_loop():
    while True:
        try:
            if os.path.exists(CMD_FILE):
                with open(CMD_FILE, "r", encoding="utf-8") as f:
                    cmd = f.read().strip()
                if cmd:
                    send_chat(cmd)
                    open(CMD_FILE, "w").close()
            time.sleep(2)
        except:
            pass


# ================= PENDEL =================

def start_pendel():
    global pendel_active
    if pendel_active:
        return
    pendel_active = True
    pendel_users.clear()
    send_chat("🧲 PENDEL START! Tippe !pendel")
    threading.Thread(target=pendel_timer, daemon=True).start()

def pendel_timer():
    global pendel_active
    time.sleep(PENDEL_DAUER)
    pendel_active = False
    if pendel_users:
        winner = random.choice(list(pendel_users))
        stats[winner]["pendel_wins"] += 1
        send_chat(f"🧲 Gewinner: @{winner}")
    pendel_users.clear()
    save_stats()

def auto_pendel():
    while True:
        time.sleep(PENDEL_INTERVAL)
        start_pendel()


# ================= RACE =================

def start_race():
    global race_active, race_join_open, race_players, bets
    if race_active:
        return

    race_active = True
    race_join_open = True
    race_players = {}
    bets = {}

    # KI erzeugen
    ai_count = random.randint(AI_MIN, AI_MAX)
    bots = random.sample(AI_BOTS, ai_count)

    for bot in bots:
        skill = random.choice(list(AI_SKILLS.keys()))
        race_players[bot] = {"position": 0, "laps": 0, "is_ai": True, "skill": skill}

    send_chat("🤖 KI-Gegner:")
    send_chat(", ".join([f"{b} ({race_players[b]['skill']})" for b in bots]))
    send_chat("🎰 Wetten möglich mit: !bet name betrag")
    send_chat(f"🏁 Rennen startet in {RACE_JOIN_TIME}s! !g zum Mitmachen")

    threading.Thread(target=race_join_timer, daemon=True).start()


def race_join_timer():
    global race_join_open
    time.sleep(RACE_JOIN_TIME)
    race_join_open = False
    send_chat("🚦 LOS!")
    threading.Thread(target=run_race, daemon=True).start()


def run_race():
    global race_active
    while race_active:
        for player in race_players:
            if race_players[player]["is_ai"]:
                skill = race_players[player]["skill"]
                data = AI_SKILLS[skill]
                speed = random.randint(data["min"], data["max"])
                if random.random() < data["turbo"]:
                    speed += 15
            else:
                speed = random.randint(5, 15)

            race_players[player]["position"] += speed

            if race_players[player]["position"] >= TRACK_LENGTH:
                race_players[player]["position"] = 0
                race_players[player]["laps"] += 1
                if race_players[player]["laps"] >= RACE_LAPS:
                    finish_race(player)
                    return
        time.sleep(0.15)


def finish_race(winner):
    global race_active
    race_active = False
    send_chat(f"🏆 {winner} gewinnt das Rennen!")

    for bettor in bets:
        target = bets[bettor]["target"]
        amount = bets[bettor]["amount"]
        if target == winner:
            stats[bettor]["points"] += amount * 2
            send_chat(f"💰 @{bettor} gewinnt {amount*2}P!")
        else:
            stats[bettor]["points"] -= amount
            send_chat(f"❌ @{bettor} verliert {amount}P")

    if winner in stats:
        stats[winner]["points"] += RACE_POINTS
        stats[winner]["races_won"] += 1

    save_stats()


# ================= AUTO RACE =================

def auto_race_monitor():
    global last_g_time
    while True:
        time.sleep(5)
        if race_active:
            continue

        if time.time() - last_g_time >= AUTO_RACE_TIMEOUT:
            send_chat("!g")  # Bot schreibt selbst !g
            last_g_time = time.time()


# ================= IRC =================

sock = socket.socket()
sock.connect((SERVER, PORT))
sock.send(f"PASS {TOKEN}\r\n".encode())
sock.send(f"NICK {NICK}\r\n".encode())
sock.send(f"JOIN {CHANNEL}\r\n".encode())

def send_chat(msg):
    sock.send(f"PRIVMSG {CHANNEL} :{msg}\r\n".encode())

def irc_loop():
    global last_g_time

    while True:
        resp = sock.recv(2048).decode(errors="ignore")

        if resp.startswith("PING"):
            sock.send("PONG :tmi.twitch.tv\r\n".encode())
            continue

        match = re.search(r":(\w+)!.* PRIVMSG .* :(.*)", resp)
        if not match:
            continue

        user = match.group(1).lower()
        msg = match.group(2).strip()

        log_message(user, msg)
        add_xp(user)

        if msg.lower() == "!g":
            last_g_time = time.time()
            if not race_active:
                start_race()
            elif race_join_open:
                race_players[user] = {"position": 0, "laps": 0, "is_ai": False}
                send_chat(f"🚗 @{user} fährt mit!")

        elif msg.lower().startswith("!bet"):
            parts = msg.split()
            if race_active and race_join_open and len(parts) == 3:
                target = parts[1]
                try:
                    amount = int(parts[2])
                except:
                    return
                if user in stats and stats[user]["points"] >= amount:
                    bets[user] = {"target": target, "amount": amount}
                    send_chat(f"🎰 @{user} wettet {amount}P auf {target}")

        elif msg.lower().startswith("!stats"):
            if user in stats:
                s = stats[user]
                send_chat(f"📊 {user} | Lvl:{s['level']} 💰{s['points']}P")

        elif msg.lower() == "!top":
            sorted_users = sorted(stats.items(), key=lambda x: x[1]["points"], reverse=True)[:5]
            ranking = " | ".join([f"{i+1}. {u[0]} ({u[1]['points']}P)" for i, u in enumerate(sorted_users)])
            send_chat("🏆 Top 5: " + ranking)


# ================= START =================

load_stats()

threading.Thread(target=irc_loop, daemon=True).start()
threading.Thread(target=dashboard_command_loop, daemon=True).start()
threading.Thread(target=auto_clear_chat, daemon=True).start()
threading.Thread(target=auto_pendel, daemon=True).start()
threading.Thread(target=auto_race_monitor, daemon=True).start()

while True:
    time.sleep(5)
