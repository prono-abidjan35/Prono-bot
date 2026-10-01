import os, threading, requests, time
from flask import Flask
from PIL import Image, ImageDraw, ImageFont
from io import BytesIO
import telebot
from telebot import types

BOT_TOKEN = os.getenv("BOT_TOKEN")
API_KEY = os.getenv("API_KEY")
CHAT_ID = os.getenv("CHAT_ID")
bot = telebot.TeleBot(BOT_TOKEN)

# --- KEEP ALIVE POUR RENDER FREE ---
app = Flask(__name__)
@app.route('/')
def home(): return "PRONO ABIDJAN PRO V3 LIVE"
def run_flask(): app.run(host='0.0.0.0', port=int(os.environ.get("PORT", 10000)))
threading.Thread(target=run_flask, daemon=True).start()

# --- GENERATEUR IMAGE 3D ---
def create_prono_image(home, away, market, conf, heure):
    W, H = 1080, 1920
    img = Image.new('RGB', (W, H), (10, 15, 25))
    draw = ImageDraw.Draw(img)
    
    # Fond degradé or/bleu
    for y in range(H):
        draw.line([(0,y),(W,y)], fill=(int(10+y*0.02), int(15+y*0.03), int(35+y*0.05)))
    
    # Cadre
    draw.rounded_rectangle([(20,20),(W-20,H-20)], radius=40, outline=(255,215,0), width=6)
    
    # Titre
    try:
        font_big = ImageFont.truetype("arial.ttf", 90)
        font_med = ImageFont.truetype("arial.ttf", 55)
        font_small = ImageFont.truetype("arial.ttf", 45)
    except:
        font_big = ImageFont.load_default()
        font_med = font_big
        font_small = font_big

    draw.text((W//2, 120), "PRONO ABIDJAN PRO", font=font_big, anchor="mm", fill=(255,215,0))
    draw.text((W//2, 220), f"{home} VS {away}", font=font_med, anchor="mm", fill="white")
    draw.text((W//2, 900), f"MARKET:\n{market}", font=font_big, anchor="mm", fill=(255,215,0), align="center")
    draw.text((W//2, 1200), f"CONFIANCE: {conf}/10", font=font_big, anchor="mm", fill="white")
    draw.text((W//2, 1350), f"{heure} GMT ABIDJAN", font=font_med, anchor="mm", fill=(0,200,255))
    
    bio = BytesIO()
    img.save(bio, 'PNG')
    bio.seek(0)
    return bio

# --- ANALYSE VRAIE IA ---
def analyser_match(match):
    # Exemple de vraie logique - tu peux affiner
    # On recupere stats API
    home_goals_avg = match.get('home_avg', 1.8)
    away_goals_avg = match.get('away_avg', 1.2)
    btts_rate = match.get('btts_rate', 0.6)
    
    total = home_goals_avg + away_goals_avg
    conf = 5
    
    if total > 3.0 and btts_rate > 0.65:
        return "Over 2.5 + BTTS OUI", 8.5
    elif total > 2.5:
        return "Over 2.5", 7.8
    elif btts_rate > 0.7:
        return "BTTS OUI", 7.5
    elif home_goals_avg > 2.0:
        return "Victoire Domicile + Over 1.5", 8.0
    else:
        return "Double Chance 1X + Under 3.5", 6.5

def send_top_10():
    # Ici ta logique API-Football
    # Pour l'exemple je simule 3 matchs
    matchs = [
        {"home":"Real Madrid","away":"Barca","home_avg":2.2,"away_avg":1.9,"btts_rate":0.75,"heure":"20:00"},
        {"home":"Man City","away":"Arsenal","home_avg":2.5,"away_avg":1.5,"btts_rate":0.6,"heure":"18:30"},
    ]
    for m in matchs:
        market, conf = analyser_match(m)
        if conf < 6.5: continue # N'ENVOIE QUE SI ANALYSE BONNE
        
        img = create_prono_image(m['home'], m['away'], market, conf, m['heure'])
        caption = f"🎯 **{m['home']} vs {m['away']}**\n📊 {market}\n🔥 Confiance {conf}/10\n💼 Mise conseillée: 5% bankroll\n⏰ {m['heure']} GMT"
        bot.send_photo(CHAT_ID, img, caption=caption, parse_mode="Markdown")
        time.sleep(2)

@bot.message_handler(commands=['start'])
def start(msg):
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True)
    markup.add("🔥 TOP 10 DU JOUR", "🎯 COMBI SAFE", "📊 TAUX IA", "💼 BANKROLL")
    bot.send_message(msg.chat.id, "Bienvenue sur PRONO ABIDJAN PRO V3 3D !", reply_markup=markup)

# Lancement auto
def loop():
    while True:
        try: send_top_10()
        except Exception as e: print(e)
        time.sleep(3600) # 1h

threading.Thread(target=loop, daemon=True).start()
bot.infinity_polling()
