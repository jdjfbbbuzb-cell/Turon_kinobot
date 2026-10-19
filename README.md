# import os
import sqlite3
import logging
from telegram import (
    Update,
    InlineKeyboardButton,
    InlineKeyboardMarkup,
)
from telegram.ext import (
    Application,
    CommandHandler,
    MessageHandler,
    CallbackQueryHandler,
    ContextTypes,
    filters,
)

TOKEN = os.environ.get("BOT_TOKEN", "")
ADMIN_ID = int(os.environ.get("ADMIN_ID", "0"))

logging.basicConfig(level=logging.INFO)

db = sqlite3.connect("movies.db", check_same_thread=False)
db.execute("""
CREATE TABLE IF NOT EXISTS movies (
    code TEXT PRIMARY KEY,
    title TEXT NOT NULL,
    description TEXT,
    file_id TEXT
)
""")
db.commit()


def get_movie(code):
    return db.execute(
        "SELECT code, title, description, file_id "
        "FROM movies WHERE code = ?",
        (code,)
    ).fetchone()


async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    keyboard = [
        [InlineKeyboardButton("🔍 Kino qidirish", callback_data="search")],
        [InlineKeyboardButton("🎬 Kino katalogi", callback_data="catalog")],
        [InlineKeyboardButton("ℹ️ Yordam", callback_data="help")],
    ]

    await update.message.reply_text(
        "🎬 TURON KINO BOT\n\n"
        "Xush kelibsiz!\n"
        "Kino kodini yuboring yoki menyudan foydalaning.",
        reply_markup=InlineKeyboardMarkup(keyboard),
    )


async def button(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()

    if query.data == "search":
        await query.message.reply_text(
            "🔍 Kino kodini yoki nomini yuboring."
        )

    elif query.data == "catalog":
        rows = db.execute(
            "SELECT code, title FROM movies ORDER BY title"
        ).fetchall()

        if not rows:
            await query.message.reply_text("Hozircha katalog bo‘sh.")
            return

        text = "🎬 KINOLAR KATALOGI\n\n"
        for code, title in rows:
            text += f"🎞 {title}\n🆔 Kod: {code}\n\n"

        await query.message.reply_text(text)

    elif query.data == "help":
        await query.message.reply_text(
            "Kino topish uchun uning kodi yoki nomini yuboring.\n"
            "Admin: /add — kino qo‘shish."
        )


async def add_movie(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != ADMIN_ID or ADMIN_ID == 0:
        await update.message.reply_text("⛔ Bu buyruq faqat admin uchun.")
        return

    context.user_data["adding_movie"] = True

    await update.message.reply_text(
        "Yangi kino yuborish tartibi:\n\n"
        "1. Kino kodi\n"
        "2. Kino nomi\n"
        "3. Kino tavsifi\n"
        "4. Video yoki Telegram videosi\n\n"
        "Masalan:\n"
        "101 | Avatar | Fantastik film"
    )


async def receive_message(
    update: Update,
    context: ContextTypes.DEFAULT_TYPE
):
    user_id = update.effective_user.id
    message = update.message

    if context.user_data.get("adding_movie"):
        if user_id != ADMIN_ID or ADMIN_ID == 0:
            return

        if message.video:
            code = context.user_data.get("movie_code")
            title = context.user_data.get("movie_title")
            description = context.user_data.get("movie_description", "")

            if not code or not title:
                await message.reply_text(
                    "Avval kino kodi, nomi va tavsifini yuboring."
                )
                return

            try:
                db.execute(
                    "INSERT OR REPLACE INTO movies "
                    "(code, title, description, file_id) "
                    "VALUES (?, ?, ?, ?)",
                    (code, title, description, message.video.file_id)
                )
                db.commit()

                context.user_data.clear()
                await message.reply_text(
                    f"✅ Kino saqlandi!\n🎬 {title}\n🆔 {code}"
                )
            except sqlite3.Error:
                await message.reply_text("Xatolik yuz berdi.")
            return

        parts = message.text.split("|", 2) if message.text else []

        if len(parts) == 3:
            code, title, description = [
                part.strip() for part in parts
            ]

            context.user_data["movie_code"] = code
            context.user_data["movie_title"] = title
            context.user_data["movie_description"] = description

            await message.reply_text(
                "✅ Ma'lumotlar olindi.\n"
                "Endi shu kinoga tegishli videoni yuboring."
            )
            return

        await message.reply_text(
            "Ma'lumotni quyidagicha yuboring:\n"
            "101 | Avatar | Fantastik film"
        )
        return

    text = (message.text or "").strip()

    movie = get_movie(text)

    if not movie:
        movie = db.execute(
            "SELECT code, title, description, file_id "
            "FROM movies WHERE title LIKE ? LIMIT 1",
            (f"%{text}%",)
        ).fetchone()

    if not movie:
        await message.reply_text(
            "😔 Kino topilmadi.\n"
            "Boshqa nom yoki kino kodini sinab ko‘ring."
        )
        return

    code, title, description, file_id = movie

    caption = (
        f"🎬 {title}\n"
        f"🆔 Kino kodi: {code}\n\n"
        f"📝 {description or 'Tavsif mavjud emas.'}"
    )

    if file_id:
        await message.reply_video(
            video=file_id,
            caption=caption
        )
    else:
        await message.reply_text(caption)


def main():
    if not TOKEN:
        raise RuntimeError("BOT_TOKEN sozlanmagan!")

    app = Application.builder().token(TOKEN).build()

    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("add", add_movie))
    app.add_handler(CallbackQueryHandler(button))
    app.add_handler(
        MessageHandler(filters.TEXT & ~filters.COMMAND, receive_message)
    )
    app.add_handler(
        MessageHandler(filters.VIDEO, receive_message)
    )

    print("TURON KINO BOT ishga tushmoqda...")
    app.run_polling()


if __name__ == "__main__":
    main()Turon_kinobot
