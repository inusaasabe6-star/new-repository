
import logging
import sqlite3
from flask import Flask, request, jsonify
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup, ReplyKeyboardMarkup
from telegram.ext import Application, CommandHandler, MessageHandler, CallbackQueryHandler, ContextTypes, filters

# ⚙️ Bayanan ArewaNetlix Bot
BOT_TOKEN = '8907547608:AAGzm56S-6vN9GMvCQDn5HdRL8-9-lTuNLo'
ADMIN_ID = 7169380507
PAYSTACK_SECRET_KEY = 'rk_live_04q-mwoDdCH4ZH-XiGFILaJUJ_MONBDB62d08b20'
CHANNEL_USERNAME = '@Arewaneflix'

# Saita Flask App don Webhook
app = Flask(__name__)

# 🗄️ Saita Database (SQLite)
def init_db():
    conn = sqlite3.connect('movies.db')
    cursor = conn.cursor()
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS movies (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            title TEXT,
            price INTEGER,
            photo TEXT,
            file_id TEXT
        )
    ''')
    conn.commit()
    conn.close()

init_db()

# 🔒 Duba ko User ya shiga Tasha (Force Subscription)
async def check_subscription(update: Update, context: ContextTypes.DEFAULT_TYPE) -> bool:
    user_id = update.effective_user.id
    if user_id == ADMIN_ID:
        return True
    
    try:
        member = await context.bot.get_chat_member(chat_id=CHANNEL_USERNAME, user_id=user_id)
        if member.status in ['member', 'creator', 'administrator']:
            return True
    except Exception as e:
        print(f"Kuskure wajen duba tasha: {e}")
    
    keyboard = [
        [InlineKeyboardButton("📢 Shiga Tasha (Join Channel)", url=f"https://t.me/{CHANNEL_USERNAME.replace('@', '')}")],
        [InlineKeyboardButton("🔄 Na Riga Na Shiga (Check)", callback_data="check_sub")]
    ]
    await update.message.reply_text(
        f"⚠️ **Dole ne ka fara biyan (Join) tashar mu ta {CHANNEL_USERNAME} kafin ka iya amfani da wannan bot ɗin!**",
        reply_markup=InlineKeyboardMarkup(keyboard),
        parse_mode='Markdown'
    )
    return False

# 🎬 /start command
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not await check_subscription(update, context):
        return

    args = context.args
    if args and args[0].startswith('movie_'):
        movie_id = args[0].split('_')[1]
        conn = sqlite3.connect('movies.db')
        cursor = conn.cursor()
        cursor.execute("SELECT * FROM movies WHERE id = ?", (movie_id,))
        movie = cursor.fetchone()
        conn.close()

        if not movie:
            await update.message.reply_text("Yi hakuri, ba a sami wannan film ɗin ba.")
            return
        
        caption = f"🎬 **{movie[1]}**\n\n💵 Price: ₦{movie[2]}.00\n\nDanna maɓallin da ke ƙasa don siye."
        keyboard = [[InlineKeyboardButton(f"💳 Buy Now (₦{movie[2]})", callback_data=f"buy_{movie[0]}")]]
        
        await update.message.reply_photo(
            photo=movie[3],
            caption=caption,
            reply_markup=InlineKeyboardMarkup(keyboard),
            parse_mode='Markdown'
        )
        return

    reply_kb = ReplyKeyboardMarkup([
        ['🎥 Jerin Fina-finai (Movies)', '🔍 Bincike (Search)'],
        ['📁 Bayanin Bot (Help)']
    ], resize_keyboard=True)

    await update.message.reply_text(
        "👋 Barka da zuwa **ArewaNetlix Bot**! Dandalin siyar da fina-finan Hausa na zamani.\n\nZaɓi abin da kake so daga maɓallan da ke ƙasa:",
        reply_markup=reply_kb,
        parse_mode='Markdown'
    )

# 🎥 Jerin Fina-finai (/movies ko maɓalli)
async def list_movies(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not await check_subscription(update, context):
        return

    conn = sqlite3.connect('movies.db')
    cursor = conn.cursor()
    cursor.execute("SELECT id, title, price FROM movies ORDER BY id DESC LIMIT 10")
    movies = cursor.fetchall()
    conn.close()

    if not movies:
        await update.message.reply_text("📭 A halin yanzu babu wani film da aka saka a cikin bot ɗin.")
        return

    keyboard = []
    for m in movies:
        keyboard.append([InlineKeyboardButton(f"🎬 {m[1]} - ₦{m[2]}", callback_data=f"view_movie_{m[0]}")])

    await update.message.reply_text(
        "🎥 **Ga jerin fina-finan da ke akwai a ArewaNetlix:**\n\nZaɓi wanda kake so a ƙasa:",
        reply_markup=InlineKeyboardMarkup(keyboard),
        parse_mode='Markdown'
    )

# 👑 ADMIN STATS (/stats)
async def admin_stats(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != ADMIN_ID:
        return

    conn = sqlite3.connect('movies.db')
    cursor = conn.cursor()
    cursor.execute("SELECT COUNT(*) FROM movies")
    count = cursor.fetchone()[0]
    conn.close()

    await update.message.reply_text(f"📊 **Admin Stats:**\n\n🎬 Jimillar Fina-finai a Database: **{count}**", parse_mode='Markdown')

# 🔄 Maɓallan Inline (Callbacks)
async def button_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()

    if query.data == "check_sub":
        user_id = query.from_user.id
        try:
            member = await context.bot.get_chat_member(chat_id=CHANNEL_USERNAME, user_id=user_id)
            if member.status in ['member', 'creator', 'administrator']:
                await query.message.delete()
                await query.message.reply_text("🎉 An tabbatar da kai! Yanzu zaka iya tura /start ko duba fina-finai.")
            else:
                await query.answer("Ba ka gama shiga tashar ba tukuna!", show_alert=True)
        except:
            await query.answer("Ka tabbata ka shiga tashar tukuna.", show_alert=True)

    elif query.data.startswith("view_movie_"):
        movie_id = query.data.split('_')[2]
        conn = sqlite3.connect('movies.db')
        cursor = conn.cursor()
        cursor.execute("SELECT * FROM movies WHERE id = ?", (movie_id,))
        movie = cursor.fetchone()
        conn.close()

        if not movie:
            await query.answer("Film bai wanzu ba.", show_alert=True)
            return

        caption = f"🎬 **{movie[1]}**\n\n💵 Farashi: ₦{movie[2]}.00\n\nDanna maɓallin da ke ƙasa don siye."
        keyboard = [[InlineKeyboardButton(f"💳 Buy Now (₦{movie[2]})", callback_data=f"buy_{movie[0]}")]]
        
        await query.message.reply_photo(
            photo=movie[3],
            caption=caption,
            reply_markup=InlineKeyboardMarkup(keyboard),
            parse_mode='Markdown'
        )

    elif query.data.startswith("buy_"):
        movie_id = query.data.split('_')[1]
        user_id = query.from_user.id
        
        conn = sqlite3.connect('movies.db')
        cursor = conn.cursor()
        cursor.execute("SELECT * FROM movies WHERE id = ?", (movie_id,))
        movie = cursor.fetchone()
        conn.close()

        if not movie:
            await query.answer("Film bai wanzu ba.", show_alert=True)
            return

        import requests
        headers = {
            "Authorization": f"Bearer {PAYSTACK_SECRET_KEY}",
            "Content-Type": "application/json"
        }
        data = {
            "email": f"{user_id}@arewaneflix.com",
            "amount": movie[2] * 100,
            "metadata": {"telegram_id": user_id, "movie_id": movie[0]}
        }
        res = requests.post("https://api.paystack.co/transaction/initialize", json=data, headers=headers)
        res_data = res.json()

        if res_data.get("status"):
            auth_url = res_data["data"]["authorization_url"]
            keyboard = [[InlineKeyboardButton("🔗 Biya Yanzu (Pay Now)", url=auth_url)]]
            await query.message.reply_text(
                f"🛒 **Siyan Film: {movie[1]}**\n\nFarashi: ₦{movie[2]}.00\n\nDanna maɓallin da ke ƙasa don biya:",
                reply_markup=InlineKeyboardMarkup(keyboard),
                parse_mode='Markdown'
            )

# 👑 ADMIN: Ajiye Film da Tura wa Tasha (Ta hanyar Bidiyo + Caption)
async def add_movie(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.message.from_user.id != ADMIN_ID:
        return

    caption = update.message.caption or ""
    if not caption.startswith('/addmovie'):
        return

    parts = caption.replace('/addmovie', '').strip().split('|')
    if len(parts) < 2:
        await update.message.reply_text("Yi amfani da tsari: `/addmovie Sunan Film | Farashi` a cikin caption ɗin bidiyo.")
        return

    title = parts[0].strip()
    price = int(parts[1].strip())

    file_id = ""
    photo_id = ""

    if update.message.video:
        file_id = update.message.video.file_id
        photo_id = update.message.video.thumb.file_id if update.message.video.thumb else file_id
    elif update.message.photo:
        photo_id = update.message.photo[-1].file_id
        file_id = photo_id

    conn = sqlite3.connect('movies.db')
    cursor = conn.cursor()
    cursor.execute("INSERT INTO movies (title, price, photo, file_id) VALUES (?, ?, ?, ?)", (title, price, photo_id, file_id))
    conn.commit()
    movie_id = cursor.lastrowid
    conn.close()

    bot_username = (await context.bot.get_me()).username
    ad_link = f"https://t.me/{bot_username}?start=movie_{movie_id}"

    try:
        await context.bot.send_photo(
            chat_id=CHANNEL_USERNAME,
            photo=photo_id,
            caption=f"🎬 **{title}**\n\n💵 Price: ₦{price}.00\n\n👇 Danna ƙasa don siyan film ɗin:",
            reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("🛒 Siye a Bot", url=ad_link)]]),
            parse_mode='Markdown'
        )
    except Exception as e:
        print(f"Kuskure wajen tura wa tasha: {e}")

    await update.message.reply_text(f"✅ An saka film tare da tura shi tasha!\n\n🔗 Link: `{ad_link}`", parse_mode='Markdown')

# 🎛️ Maɓallan rubutu na kasa (Reply Keyboard Handlers)
async def handle_text_messages(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text
    if text == "🎥 Jerin Fina-finai (Movies)":
        await list_movies(update, context)
    elif text == "📁 Bayanin Bot (Help)":
        await update.message.reply_text("ℹ️ Wannan bot na ArewaNetlix ne. Zaka iya zaɓar fina-finai ka siya ta hanyar Paystack cikin sauƙi.")

# 💳 Paystack Webhook Route
@app.route('/webhook/paystack', methods=['POST'])
def paystack_webhook():
    event = request.json
    if event and event.get('event') == 'charge.success':
        data = event['data']
        metadata = data.get('metadata', {})
        movie_id = metadata.get('movie_id')
        telegram_id = metadata.get('telegram_id')

        if movie_id and telegram_id:
            conn = sqlite3.connect('movies.db')
            cursor = conn.cursor()
            cursor.execute("SELECT * FROM movies WHERE id = ?", (movie_id,))
            movie = cursor.fetchone()
            conn.close()

            if movie:
                import requests as req_lib
                vid_url = f"https://api.telegram.org/bot{BOT_TOKEN}/sendVideo"
                payload = {
                    "chat_id": telegram_id,
                    "video": movie[4],
                    "caption": f"🎉 **Na gode da siyan {movie[1]}!**\n\nGa bidiyonka nan. Ka kalla lafiya!",
                    "parse_mode": "Markdown"
                }
                req_lib.post(vid_url, json=payload)

    return jsonify(success=True), 200

@app.route('/')
def home():
    return "ArewaNetlix Advanced Bot is running!"

if __name__ == '__main__':
    import threading
    
    application = Application.builder().token(BOT_TOKEN).build()
    application.add_handler(CommandHandler("start", start))
    application.add_handler(CommandHandler("movies", list_movies))
    application.add_handler(CommandHandler("stats", admin_stats))
    application.add_handler(CallbackQueryHandler(button_handler))
    application.add_handler(MessageHandler(filters.VIDEO | filters.PHOTO, add_movie))
    application.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_text_messages))

    def run_flask():
        app.run(host='0.0.0.0', port=3000)

    t = threading.Thread(target=run_flask)
    t.start()

    print("ArewaNetlix Advanced Bot yana aiki yanzu...")
    application.run_polling()
    
