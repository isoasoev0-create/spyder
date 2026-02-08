import telebot
from telebot import types
import sqlite3
import time
import random

# ТОКЕНИ ТУ ВА ТАНЗИМОТ
TOKEN = '8546870219:AAHPWQan9MKFPnZRCJLxLdcgHC1_KCmSp_8'
bot = telebot.TeleBot(TOKEN)

ADMIN_ID = 12345678  # !!! ИН ҶО ID-И ХУДРО АЗ @userinfobot ГИРИФТА МОН !!!
MY_CARD = "9762000176640596"
REFERRAL_BONUS = 0.20

# --- БАЗАИ МАЪЛУМОТ ---
def init_db():
    conn = sqlite3.connect('invest_bot.db')
    cursor = conn.cursor()
    cursor.execute('''CREATE TABLE IF NOT EXISTS users 
                      (user_id INTEGER PRIMARY KEY, balance REAL DEFAULT 0, 
                       invested_amount REAL DEFAULT 0, last_update INTEGER,
                       last_bonus INTEGER DEFAULT 0, referred_by INTEGER)''')
    conn.commit()
    conn.close()

def get_user_data(user_id):
    conn = sqlite3.connect('invest_bot.db')
    cursor = conn.cursor()
    cursor.execute("SELECT balance, invested_amount, last_update FROM users WHERE user_id=?", (user_id,))
    row = cursor.fetchone()
    if not row:
        now = int(time.time())
        cursor.execute("INSERT INTO users (user_id, balance, invested_amount, last_update) VALUES (?, 0, 0, ?)", (user_id, now))
        conn.commit()
        row = (0, 0, now)
    conn.close()
    return row

# --- АСОСӢ / START ---
@bot.message_handler(commands=['start'])
def start(message):
    init_db()
    user_id = message.from_user.id
    
    # Реферальная система
    args = message.text.split()
    if len(args) > 1 and args[1].isdigit():
        ref_id = int(args[1])
        conn = sqlite3.connect('invest_bot.db')
        cursor = conn.cursor()
        cursor.execute("SELECT user_id FROM users WHERE user_id=?", (user_id,))
        if not cursor.fetchone():
            now = int(time.time())
            cursor.execute("INSERT INTO users (user_id, balance, last_update, referred_by) VALUES (?, 0, ?, ?)", (user_id, now, ref_id))
            cursor.execute("UPDATE users SET balance = balance + ? WHERE user_id = ?", (REFERRAL_BONUS, ref_id))
            bot.send_message(ref_id, f"🎊 Дӯсти нав ҳамроҳ шуд! Ба шумо {REFERRAL_BONUS}см бонус дода шуд.")
            conn.commit()
        conn.close()

    markup = types.ReplyKeyboardMarkup(resize_keyboard=True, row_width=2)
    markup.add("📈 Нақшаҳо", "📊 Баланси ман", "💸 Вывод", "🎁 Бонус", "👥 Реферал", "📞 Кӯмак")
    bot.send_message(message.chat.id, "💰 Хуш омадед ба боти сармоягузорӣ!", reply_markup=markup)

# --- БАЛАНС ВА ҲИСОБИ ХУДКОР ---
@bot.message_handler(func=lambda message: message.text == "📊 Баланси ман")
def check_balance(message):
    balance, invested, last_update = get_user_data(message.from_user.id)
    now = int(time.time())
    
    if invested > 0:
        diff = now - last_update
        profit = 0
        if invested >= 240: profit = 12.33
        elif invested >= 120: profit = 6
        elif invested >= 60: profit = 3
        
        new_profit = (profit / 86400) * diff
        balance += new_profit
        
        conn = sqlite3.connect('invest_bot.db')
        cursor = conn.cursor()
        cursor.execute("UPDATE users SET balance=?, last_update=? WHERE user_id=?", (balance, now, message.from_user.id))
        conn.commit()
        conn.close()
    
    bot.send_message(message.chat.id, f"💰 **Баланс:** {round(balance, 4)} сомонӣ\n🏗 **Сармоя:** {invested} сомонӣ", parse_mode="Markdown")

# --- НАҚШАҲО ---
@bot.message_handler(func=lambda message: message.text == "📈 Нақшаҳо")
def show_plans(message):
    plans = ("🚀 **Нақшаҳои сармоягузорӣ:**\n\n"
             "1️⃣ 60см -> рӯзе 3см (30 рӯз 90см)\n"
             "2️⃣ 120см -> рӯзе 6см (30 рӯз 180см)\n"
             "3️⃣ 240см -> рӯзе 12.33см (30 рӯз 370см)\n\n"
             f"Барои фаъол кардан ба `{MY_CARD}` пул гузаронед ва чекро фиристед.")
    bot.send_message(message.chat.id, plans, parse_mode="Markdown")

# --- БОНУС ВА РЕФЕРАЛ ---
@bot.message_handler(func=lambda message: message.text == "🎁 Бонус")
def get_bonus(message):
    user_id = message.from_user.id
    conn = sqlite3.connect('invest_bot.db')
    cursor = conn.cursor()
    cursor.execute("SELECT balance, last_bonus FROM users WHERE user_id=?", (user_id,))
    res = cursor.fetchone()
    now = int(time.time())
    
    if res and (now - res[1] < 86400):
        bot.send_message(message.chat.id, "❌ Бонус дар 24 соат 1 бор дода мешавад.")
    else:
        win = round(random.uniform(0.10, 0.50), 2)
        cursor.execute("UPDATE users SET balance=balance+?, last_bonus=? WHERE user_id=?", (win, now, user_id))
        conn.commit()
        bot.send_message(message.chat.id, f"🎉 Шумо {win} сомонӣ бонус гирифтед!")
    conn.close()

@bot.message_handler(func=lambda message: message.text == "👥 Реферал")
def ref_link(message):
    link = f"https://t.me/{bot.get_me().username}?start={message.from_user.id}"
    bot.send_message(message.chat.id, f"🤝 Барои ҳар як дӯст {REFERRAL_BONUS}см гиред:\n`{link}`", parse_mode="Markdown")

# --- ВЫВОД ВА КӮМАК ---
@bot.message_handler(func=lambda message: message.text == "💸 Вывод")
def withdraw(message):
    msg = bot.send_message(message.chat.id, "Маблағ ва рақами кортро нависед:")
    bot.register_next_step_handler(msg, lambda m: bot.send_message(ADMIN_ID, f"🚨 ВЫВОД: {m.text} аз @{m.from_user.username} (ID: {m.from_user.id})"))

@bot.message_handler(func=lambda message: message.text == "📞 Кӯмак")
def help_rules(message):
    bot.send_message(message.chat.id, "📜 Сармоягузорӣ аз 60см. Пардохт дар 24 соат. Админ: @Bmwtj0")

# --- АДМИН: ТАСДИҚИ ЧЕК ВА ФАРМОНИ PAY ---
@bot.message_handler(content_types=['photo'])
def handle_photo(message):
    bot.reply_to(message, "✅ Чек қабул шуд! Интизор шавед.")
    bot.send_message(ADMIN_ID, f"🔔 ЧЕКИ НАВ! ID: `{message.from_user.id}`\n`/pay {message.from_user.id} 60`", parse_mode="Markdown")
    bot.forward_message(ADMIN_ID, message.chat.id, message.message_id)

@bot.message_handler(commands=['pay'])
def admin_pay(message):
    if message.from_user.id == ADMIN_ID:
        try:
            args = message.text.split()
            u_id, amt = int(args[1]), float(args[2])
            conn = sqlite3.connect('invest_bot.db')
            cursor = conn.cursor()
            cursor.execute("UPDATE users SET invested_amount=?, last_update=? WHERE user_id=?", (amt, int(time.time()), u_id))
            conn.commit()
            conn.close()
            bot.send_message(u_id, f"🚀 Сармояи {amt}см фаъол шуд!")
            bot.send_message(ADMIN_ID, "✅ Иҷро шуд.")
        except:
            bot.send_message(ADMIN_ID, "Хато! Намуна: /pay ID 60")

init_db()
bot.polling(none_stop=True)

