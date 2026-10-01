import os, requests, telebot, json
from telebot import types
from apscheduler.schedulers.background import BackgroundScheduler
import pytz
from datetime import datetime

# --- CONFIG SECURISEE ---
BOT_TOKEN = os.getenv("BOT_TOKEN")
API_KEY = os.getenv("API_KEY")
CHAT_ID = os.getenv("CHAT_ID")

bot = telebot.TeleBot(BOT_TOKEN)
headers = {"x-apisports-key": API_KEY}
tz = pytz.timezone('Africa/Abidjan')

# --- HISTORIQUE IA ---
def apprendre(type_prono, gagne):
    try:
        with open("historique.json", "r") as f: hist = json.load(f)
    except: hist = {}
    if type_prono not in hist: hist[type_prono] = {"gagne":0, "perdu":0}
    if gagne: hist[type_prono]["gagne"] += 1
    else: hist[type_prono]["perdu"] += 1
    with open("historique.json", "w") as f: json.dump(hist, f)

def get_taux_reussite():
    try:
        with open("historique.json", "r") as f: hist = json.load(f)
        txt = "🧠 IA - TAUX DE REUSSITE:\n"
        for k,v in hist.items():
            total = v["gagne"]+v["perdu"]
            if total>0: txt += f"{k}: {v['gagne']/total*100:.0f}% ({v['gagne']}/{total})\n"
        return txt
    except: return "Pas encore de données IA"

# --- PARTIE 1 : LIVE + STATS ---
def get_live():
    url = "https://v3.football.api-sports.io/fixtures?live=all"
    r = requests.get(url, headers=headers).json()
    if not r['response']: return "Aucun match live actuellement."
    txt = "📊 LIVE EN COURS:\n\n"
    for m in r['response'][:10]:
        txt += f"{m['teams']['home']['name']} {m['goals']['home']}-{m['goals']['away']} {m['teams']['away']['name']} ({m['fixture']['status']['elapsed']}')\n"
    return txt

def get_stats(fixture_id):
    url = f"https://v3.football.api-sports.io/fixtures/statistics?fixture={fixture_id}"
    r = requests.get(url, headers=headers).json()
    if not r['response']: return "Stats non dispo"
    txt = "📊 STATS DÉTAILLÉES:\n"
    for team_stat in r['response']:
        txt += f"\n{team_stat['team']['name']}:\n"
        for s in team_stat['statistics']:
            if s['type'] in ['Ball Possession', 'Total Shots', 'Shots on Goal', 'Corner Kicks', 'Yellow Cards']:
                txt += f"- {s['type']}: {s['value']}\n"
    return txt

# --- PARTIE 2 : PRONOS ---
def get_pronos(period):
    count = 10 if period=="day" else 4
    today = datetime.now(tz).strftime("%Y-%m-%d")
    url = f"https://v3.football.api-sports.io/fixtures?date={today}"
    r = requests.get(url, headers=headers).json()
    txt = f"{'☀️ JOUR' if period=='day' else '🌙 NUIT'} - TOP {count} - {today}\n\n"
    if not r['response']: return txt + "Pas de matchs aujourd'hui"
    for i, m in enumerate(r['response'][:count], 1):
        home = m['teams']['home']['name']
        away = m['teams']['away']['name']
        txt += f"{i}. {home} vs {away}\n -> Confiance 7/10 - Tendance: Over 1.5 + BTTS\n"
    txt += f"\n{get_taux_reussite()}"
    return txt

# --- PARTIE 3 : ALERTE BUT + CASHOUT ---
def check_goals_and_cashout():
    url = "https://v3.football.api-sports.io/fixtures?live=all"
    r = requests.get(url, headers=headers).json()
    for m in r['response']:
        elapsed = m['fixture']['status']['elapsed'] or 0
        # Alerte Cashout si match tendu après 75'
        if elapsed > 75 and abs(m['goals']['home'] - m['goals']['away']) == 1:
            msg = f"💸 ALERTE CASHOUT!\n{m['teams']['home']['name']} {m['goals']['home']}-{m['goals']['away']} {m['teams']['away']['name']} ({elapsed}')\nScore serré, sécurise ton gain maintenant!"
            try: bot.send_message(CHAT_ID, msg)
            except: pass

# --- MENU TELEGRAM ---
@bot.message_handler(commands=['start'])
def start(msg):
    markup = types.InlineKeyboardMarkup(row_width=1)
    markup.add(
        types.InlineKeyboardButton("📊 SUIVI LIVE", callback_data="live"),
        types.InlineKeyboardButton("📈 STATS LIVE (1er match)", callback_data="stats_first"),
        types.InlineKeyboardButton("☀️ 10 PRONOS JOUR", callback_data="day"),
        types.InlineKeyboardButton("🌙 4 PRONOS NUIT", callback_data="night"),
        types.InlineKeyboardButton("🧠 TAUX IA", callback_data="ia"),
    )
    bot.send_message(msg.chat.id, "🔵 PRONO ABIDJAN PRO - Choisis:", reply_markup=markup)

@bot.callback_query_handler(func=lambda c: True)
def handle(c):
    if c.data=="live": bot.send_message(c.message.chat.id, get_live())
    if c.data=="day": bot.send_message(c.message.chat.id, get_pronos("day"))
    if c.data=="night": bot.send_message(c.message.chat.id, get_pronos("night"))
    if c.data=="ia": bot.send_message(c.message.chat.id, get_taux_reussite())
    if c.data=="stats_first":
        url = "https://v3.football.api-sports.io/fixtures?live=all"
        r = requests.get(url, headers=headers).json()
        if r['response']:
            fid = r['response'][0]['fixture']['id']
            bot.send_message(c.message.chat.id, get_stats(fid))
        else: bot.send_message(c.message.chat.id, "Aucun match live pour les stats")

# --- SCHEDULER ---
sched = BackgroundScheduler(timezone=tz)
sched.add_job(lambda: bot.send_message(CHAT_ID, get_pronos("day")), 'cron', hour=8, minute=0)
sched.add_job(lambda: bot.send_message(CHAT_ID, get_pronos("night")), 'cron', hour=20, minute=0)
sched.add_job(check_goals_and_cashout, 'interval', minutes=3)
sched.start()

print("Bot PRONO ABIDJAN PRO lancé...")
bot.infinity_polling()
