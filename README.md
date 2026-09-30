import os
import asyncio
import logging
from aiohttp import web
from aiogram import Bot, Dispatcher, types, F
from aiogram.filters import CommandStart, Command
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup
from aiogram.fsm.storage.memory import MemoryStorage
from aiogram.types import (
    InlineKeyboardMarkup, InlineKeyboardButton,
    ReplyKeyboardMarkup, KeyboardButton, ReplyKeyboardRemove
)

# ==================== SOZLAMALAR ====================
BOT_TOKEN = os.getenv("BOT_TOKEN", "8736913988:AAFCpRN6ytjo6-19gzUfEV3pwYDsPZIxcqo")
CHANNEL_ID = os.getenv("CHANNEL_ID", "@Anifible")
ADMIN_ID = int(os.getenv("ADMIN_ID", "0"))  # O'z telegram ID raqamingizni kiriting
PORT = int(os.getenv("PORT", 10000))

CHANNEL_TAG = "@Anifible"

# Ma'lumotlar ombori (SQLite yoki oddiy xotira)
# {anime_code: {"title": str, "poster": file_id, "desc": str, "episodes": [file_id1, file_id2, ...]}}
anime_db = {}

logging.basicConfig(level=logging.INFO)
bot = Bot(token=BOT_TOKEN)
dp = Dispatcher(storage=MemoryStorage())

# ==================== FSM BOSQICHLARI ====================
class AnimeUpload(StatesGroup):
    waiting_for_code = State()
    waiting_for_title = State()
    waiting_for_poster_desc = State()
    uploading_videos = State()
    confirm_channel_post = State()

# ==================== ADMIN BUYRUQLARI ====================
@dp.message(Command("addanime"))
async def add_anime_start(message: types.Message, state: FSMContext):
    if ADMIN_ID != 0 and message.from_user.id != ADMIN_ID:
        return await message.answer("Siz admin emassiz.")
    
    await state.set_state(AnimeUpload.waiting_for_code)
    await message.answer("Anime kodini kiriting (masalan: 101):", reply_markup=ReplyKeyboardRemove())

@dp.message(AnimeUpload.waiting_for_code)
async def process_code(message: types.Message, state: FSMContext):
    code = message.text.strip()
    await state.update_data(code=code, episodes=[])
    await state.set_state(AnimeUpload.waiting_for_title)
    await message.answer("Anime nomi va faslini kiriting:\n(Masalan: <b>Omadsizning qayta tug'ilishi [1-fasl]</b>)", parse_mode="HTML")

@dp.message(AnimeUpload.waiting_for_title)
async def process_title(message: types.Message, state: FSMContext):
    title = message.text.strip()
    await state.update_data(title=title)
    await state.set_state(AnimeUpload.waiting_for_poster_desc)
    await message.answer("Anime posterini (rasm) tashlang va izoh (caption) qismiga tavsifini yozing:")

@dp.message(AnimeUpload.waiting_for_poster_desc, F.photo)
async def process_poster_desc(message: types.Message, state: FSMContext):
    photo_id = message.photo[-1].file_id
    desc = message.caption or "Anime tavsifi mavjud emas."
    await state.update_data(poster=photo_id, desc=desc)
    
    await state.set_state(AnimeUpload.uploading_videos)
    finish_kb = ReplyKeyboardMarkup(
        keyboard=[[KeyboardButton(text="✅ Yuklashni yakunlash")]],
        resize_keyboard=True
    )
    await message.answer(
        "Ajoyib! Endi animening barcha qismlarini (video ko'rinishida) navbatma-navbat tashlang.\n\n"
        "Barcha qismlarni tashlab bo'lgach, pastdagi <b>«✅ Yuklashni yakunlash»</b> tugmasini bosing.",
        reply_markup=finish_kb,
        parse_mode="HTML"
    )

@dp.message(AnimeUpload.uploading_videos, F.video)
async def process_video(message: types.Message, state: FSMContext):
    data = await state.get_data()
    episodes = data.get("episodes", [])
    episodes.append(message.video.file_id)
    await state.update_data(episodes=episodes)
    
    count = len(episodes)
    await message.answer(f"✅ {count}-qism qabul qilindi.")

@dp.message(AnimeUpload.uploading_videos, F.text == "✅ Yuklashni yakunlash")
async def finish_videos(message: types.Message, state: FSMContext):
    data = await state.get_data()
    episodes = data.get("episodes", [])
    
    if not episodes:
        return await message.answer("Siz bitta ham qism yuklamadingiz! Kamida 1 ta video yuboring.")
    
    # Ma'lumotlar bazasiga saqlash
    code = data["code"]
    anime_db[code] = {
        "title": data["title"],
        "poster": data["poster"],
        "desc": data["desc"],
        "episodes": episodes
    }
    
    await state.set_state(AnimeUpload.confirm_channel_post)
    
    confirm_kb = InlineKeyboardMarkup(inline_keyboard=[
        [
            InlineKeyboardButton(text="Ha, yuborilsin 🚀", callback_data="post_yes"),
            InlineKeyboardButton(text="Yo'q, kerakmas ❌", callback_data="post_no")
        ]
    ])
    
    await message.answer(
        f"Anime muvaffaqiyatli saqlandi!\n"
        f"Kodi: <b>{code}</b>\n"
        f"Nomi: <b>{data['title']}</b>\n"
        f"Jami qismlar: <b>{len(episodes)} ta</b>\n\n"
        f"<b>Kanalga e'lon posti yuborilsinmi?</b>",
        reply_markup=confirm_kb,
        parse_mode="HTML"
    )

@dp.callback_query(AnimeUpload.confirm_channel_post, F.data.in_(["post_yes", "post_no"]))
async def handle_channel_post_decision(call: types.CallbackQuery, state: FSMContext):
    data = await state.get_data()
    code = data["code"]
    title = data["title"]
    desc = data["desc"]
    poster = data["poster"]
    total_episodes = len(data["episodes"])
    
    bot_info = await bot.get_me()
    
    if call.data == "post_yes":
        channel_post_text = (
            f"🎬 <b>Yangi Anime Qo'shildi!</b>\n\n"
            f"🏷 <b>Nomi:</b> {title}\n"
            f"🔢 <b>Kodi:</b> <code>{code}</code>\n"
            f"🎞 <b>Qismlar soni:</b> {total_episodes} ta\n\n"
            f"📝 <b>Tavsif:</b>\n{desc}\n\n"
            f"Tomosha qilish uchun pastdagi tugmani bosing yoki botga <code>{code}</code> kodini yuboring.\n\n"
            f"Kanal: {CHANNEL_TAG}"
        )
        
        btn = InlineKeyboardMarkup(inline_keyboard=[
            [InlineKeyboardButton(text="Tomosha qilish 🍿", url=f"https://t.me/{bot_info.username}?start={code}")]
        ])
        
        try:
            await bot.send_photo(chat_id=CHANNEL_ID, photo=poster, caption=channel_post_text, reply_markup=btn, parse_mode="HTML")
            await call.message.edit_text("✅ Kanalga post muvaffaqiyatli yuborildi!", reply_markup=None)
        except Exception as e:
            await call.message.edit_text(f"❌ Kanalga yuborishda xatolik: {e}", reply_markup=None)
    else:
        await call.message.edit_text("Post kanalga yuborilmadi. Anime faqat bot bazasida qoldi.", reply_markup=None)
        
    await state.clear()

# ==================== FOYDALANUVCHILAR UCHUN QIDIRUV VA YUKLASH ====================
async def send_anime_episodes(chat_id: int, code: str):
    if code not in anime_db:
        return await bot.send_message(chat_id, "Bunday kodli anime topilmadi. Kodni to'g'ri kiritganingizni tekshiring.")
    
    anime = anime_db[code]
    title = anime["title"]
    episodes = anime["episodes"]
    
    await bot.send_message(chat_id, f"<b>{title}</b> qismlari yuklanmoqda...")
    
    for idx, video_id in enumerate(episodes, 1):
        caption = (
            f"{title} {idx}-qism\n"
            f"Kanal: {CHANNEL_TAG}"
        )
        try:
            await bot.send_video(
                chat_id=chat_id,
                video=video_id,
                caption=caption
            )
            await asyncio.sleep(0.5)
        except Exception as e:
            logging.error(f"Video yuborishda xatolik: {e}")

@dp.message(CommandStart())
async def start_handler(message: types.Message):
    args = message.text.split()
    if len(args) > 1:
        code = args[1].strip()
        await send_anime_episodes(message.chat.id, code)
    else:
        await message.answer(
            f"Assalomu alaykum, <b>{message.from_user.first_name}</b>!\n\n"
            f"Bu <b>{CHANNEL_TAG}</b> rasmiy anime boti.\n"
            "Tomosha qilmoqchi bo'lgan animenang kodini yuboring (masalan: <code>101</code>):",
            parse_mode="HTML"
        )

@dp.message(F.text)
async def search_by_code(message: types.Message):
    code = message.text.strip()
    if code.isdigit():
        await send_anime_episodes(message.chat.id, code)
    else:
        await message.answer("Iltimos, faqat anime kodini (raqam) yuboring.")

# ==================== RENDER WEB SERVER (UPTIMEROBOT UCHUN) ====================
async def health_check(request):
    return web.Response(text="Anime Bot is Running 24/7!", status=200)

async def start_web_server():
    app = web.Application()
    app.router.add_get("/", health_check)
    runner = web.AppRunner(app)
    await runner.setup()
    site = web.TCPSite(runner, "0.0.0.0", PORT)
    await site.start()

# ==================== ASOSIY ISHGA TUSHIRISH ====================
async def main():
    await bot.delete_webhook(drop_pending_updates=True)
    await start_web_server()
    logging.info("Anime bot ishga tushdi...")
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
