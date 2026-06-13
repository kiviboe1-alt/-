import asyncio
import sqlite3
import os
from aiogram import Bot, Dispatcher, types, F
from aiogram.filters import Command
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup
from aiogram.fsm.storage.memory import MemoryStorage
from aiogram.utils.keyboard import InlineKeyboardBuilder, ReplyKeyboardBuilder

# !!! ЗАМЕНИТЕ ЭТОТ ТЕКСТ НА ВАШ ТОКЕН !!!
BOT_TOKEN = "8666430655:AAH6_jVUly5i6B5qMD0FVjWJJzLfI9JICXU"

bot = Bot(token=BOT_TOKEN)
dp = Dispatcher(storage=MemoryStorage())

def init_db():
    conn = sqlite3.connect("dating.db")
    cursor = conn.cursor()
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS users (
            tg_id INTEGER PRIMARY KEY, username TEXT, name TEXT, age INTEGER, bio TEXT, gender TEXT, search_gender TEXT
        )
    """)
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS reactions (
            from_id INTEGER, to_id INTEGER, type TEXT, PRIMARY KEY (from_id, to_id)
        )
    """)
    conn.commit()
    conn.close()

class Reg(StatesGroup):
    name, age, bio, gender, search_gender = State(), State(), State(), State(), State()

@dp.message(Command("start"))
async def cmd_start(message: types.Message, state: FSMContext):
    conn = sqlite3.connect("dating.db")
    cursor = conn.cursor()
    cursor.execute("SELECT tg_id FROM users WHERE tg_id = ?", (message.from_user.id,))
    user = cursor.fetchone()
    conn.close()
    if user:
        await message.answer("👋 С возвращением! Напиши /search для поиска людей.")
    else:
        await message.answer("Привет! Давай создадим анкету. Как тебя зовут?")
        await state.set_state(Reg.name)

@dp.message(Reg.name)
async def reg_name(message: types.Message, state: FSMContext):
    await state.update_data(name=message.text)
    await message.answer("Сколько тебе лет?")
    await state.set_state(Reg.age)

@dp.message(Reg.age)
async def reg_age(message: types.Message, state: FSMContext):
    if not message.text.isdigit():
        await message.answer("Введи возраст цифрами:")
        return
    await state.update_data(age=int(message.text))
    await message.answer("Расскажи о себе (твои интересы):")
    await state.set_state(Reg.bio)

@dp.message(Reg.bio)
async def reg_bio(message: types.Message, state: FSMContext):
    await state.update_data(bio=message.text)
    kb = ReplyKeyboardBuilder()
    kb.button(text="Я парень")
    kb.button(text="Я девушка")
    await message.answer("Твой пол:", reply_markup=kb.as_markup(resize_keyboard=True, one_time_keyboard=True))
    await state.set_state(Reg.gender)

@dp.message(Reg.gender, F.text.in_(["Я парень", "Я девушка"]))
async def reg_gender(message: types.Message, state: FSMContext):
    await state.update_data(gender="М" if message.text == "Я парень" else "Ж")
    kb = ReplyKeyboardBuilder()
    kb.button(text="Парней")
    kb.button(text="Девушек")
    await message.answer("Кого ты ищешь?", reply_markup=kb.as_markup(resize_keyboard=True, one_time_keyboard=True))
    await state.set_state(Reg.search_gender)

@dp.message(Reg.search_gender, F.text.in_(["Парней", "Девушек"]))
async def reg_search_gender(message: types.Message, state: FSMContext):
    search_gender = "М" if message.text == "Парней" else "Ж"
    data = await state.get_data()
    conn = sqlite3.connect("dating.db")
    cursor = conn.cursor()
    cursor.execute("""
        INSERT OR REPLACE INTO users VALUES (?, ?, ?, ?, ?, ?, ?)
    """, (message.from_user.id, message.from_user.username, data['name'], data['age'], data['bio'], data['gender'], search_gender))
    conn.commit()
    conn.close()
    await message.answer("🎉 Успешно! Твоя анкета создана (фото скрыто). Напиши /search для поиска.", reply_markup=types.ReplyKeyboardRemove())
    await state.clear()

async def send_profile(message: types.Message, user_id: int):
    conn = sqlite3.connect("dating.db")
    cursor = conn.cursor()
    cursor.execute("SELECT gender, search_gender FROM users WHERE tg_id = ?", (user_id,))
    user = cursor.fetchone()
    if not user:
        await message.answer("Сначала напиши /start")
        conn.close()
        return
    my_gender, my_search = user
    cursor.execute("""
        SELECT tg_id, name, age, bio FROM users 
        WHERE gender = ? AND search_gender = ? AND tg_id != ? 
        AND tg_id NOT IN (SELECT to_id FROM reactions WHERE from_id = ?) LIMIT 1
    """, (my_search, my_gender, user_id, user_id))
    profile = cursor.fetchone()
    conn.close()
    if not profile:
        await message.answer("Анкеты закончились! Попробуй позже.")
        return
    p_id, p_name, p_age, p_bio = profile
    kb = InlineKeyboardBuilder()
    kb.button(text="👎", callback_data=f"dis_{p_id}")
    kb.button(text="❤️", callback_data=f"lik_{p_id}")
    await message.answer(f"👤 {p_name}, {p_age}\n\n{p_bio}\n\n⚠️ Фото скрыто", reply_markup=kb.as_markup())

@dp.message(Command("search"))
async def cmd_search(message: types.Message):
    await send_profile(message, message.from_user.id)

@dp.callback_query(F.data.startswith("lik_"))
async def proc_like(call: types.CallbackQuery):
    t_id = int(call.data.split("_")[1])
    m_id = call.from_user.id
    conn = sqlite3.connect("dating.db")
    cursor = conn.cursor()
    cursor.execute("INSERT OR REPLACE INTO reactions VALUES (?, ?, 'like')", (m_id, t_id))
    cursor.execute("SELECT type FROM reactions WHERE from_id = ? AND to_id = ? AND type = 'like'", (t_id, m_id))
    match = cursor.fetchone()
    if match:
        cursor.execute("SELECT username, name FROM users WHERE tg_id = ?", (t_id,))
        t_user = cursor.fetchone()
        cursor.execute("SELECT username, name FROM users WHERE tg_id = ?", (m_id,))
        m_user = cursor.fetchone()
        t_lnk = f"@{t_user[0]}" if t_user[0] else t_user[1]
        m_lnk = f"@{m_user[0]}" if m_user[0] else m_user[1]
        await call.message.answer(f"🎉 Взаимный мэтч с {t_user[1]}! Ссылка: {t_lnk}")
        try: await bot.send_message(t_id, f"🎉 Взаимный мэтч с {m_user[1]}! Ссылка: {m_lnk}")
        except: pass
    conn.commit()
    conn.close()
    await call.message.delete()
    await send_profile(call.message, m_id)

@dp.callback_query(F.data.startswith("dis_"))
async def proc_dis(call: types.CallbackQuery):
    t_id = int(call.data.split("_")[1])
    m_id = call.from_user.id
    conn = sqlite3.connect("dating.db")
    cursor = conn.cursor()
    cursor.execute("INSERT OR REPLACE INTO reactions VALUES (?, ?, 'dislike')", (m_id, t_id))
    conn.commit()
    conn.close()
    await call.message.delete()
    await send_profile(call.message, m_id)

async def main():
    init_db()
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
