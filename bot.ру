import asyncio
import html
import logging
import os
import sqlite3
import time
from datetime import datetime, timedelta

from aiogram import Bot, Dispatcher, F, Router
from aiogram.client.default import DefaultBotProperties
from aiogram.enums import ParseMode
from aiogram.filters import Command, CommandStart
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup
from aiogram.types import (
    CallbackQuery,
    InlineKeyboardButton,
    InlineKeyboardMarkup,
    KeyboardButton,
    Message,
    ReplyKeyboardMarkup,
)
from aiogram.exceptions import TelegramBadRequest, TelegramForbiddenError


# =========================================================
# НАСТРОЙКИ
# =========================================================

BOT_TOKEN = os.getenv("BOT_TOKEN")

# Необязательно.
# Если добавишь ADMIN_CHAT_ID в Render,
# туда будут приходить жалобы.
ADMIN_CHAT_ID = os.getenv("ADMIN_CHAT_ID")

DB_PATH = "premium.db"

MAX_MESSAGE_LENGTH = 2000
MESSAGE_COOLDOWN = 5

ROOMS = {
    "general": {
        "name": "💬 Общая",
        "description": "Свободное общение обо всём.",
    },
    "goals": {
        "name": "🎯 Цели и дисциплина",
        "description": "Цели, привычки, дисциплина и движение вперёд.",
    },
    "people": {
        "name": "🤝 Общение и отношения",
        "description": "Друзья, знакомства, отношения и общение.",
    },
    "self": {
        "name": "🧠 Про себя",
        "description": "Мысли, самооценка, характер и личный рост.",
    },
    "hard": {
        "name": "❤️ Когда тяжело",
        "description": "Поддержка в сложные периоды.",
    },
    "work": {
        "name": "💼 Учёба и работа",
        "description": "Учёба, карьера, работа и деньги.",
    },
}


# =========================================================
# BOT
# =========================================================

if not BOT_TOKEN:
    raise RuntimeError(
        "Не найден BOT_TOKEN. Добавь BOT_TOKEN в Environment Variables."
    )

bot = Bot(
    token=BOT_TOKEN,
    default=DefaultBotProperties(parse_mode=ParseMode.HTML),
)

dp = Dispatcher()
router = Router()
dp.include_router(router)


# =========================================================
# FSM
# =========================================================

class WriteMessage(StatesGroup):
    waiting_for_text = State()


class ReplyMessage(StatesGroup):
    waiting_for_text = State()


# =========================================================
# DATABASE
# =========================================================

def db():
    return sqlite3.connect(DB_PATH)


def init_database():
    conn = db()
    cur = conn.cursor()

    cur.execute("""
        CREATE TABLE IF NOT EXISTS users (
            user_id INTEGER PRIMARY KEY,
            alias_number INTEGER NOT NULL,
            joined_at TEXT NOT NULL,
            banned INTEGER DEFAULT 0
        )
    """)

    cur.execute("""
        CREATE TABLE IF NOT EXISTS room_members (
            room_id TEXT NOT NULL,
            user_id INTEGER NOT NULL,
            joined_at TEXT NOT NULL,
            PRIMARY KEY (room_id, user_id)
        )
    """)

    cur.execute("""
        CREATE TABLE IF NOT EXISTS messages (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            room_id TEXT NOT NULL,
            user_id INTEGER NOT NULL,
            alias_number INTEGER NOT NULL,
            text TEXT NOT NULL,
            created_at TEXT NOT NULL,
            reply_to INTEGER,
            deleted INTEGER DEFAULT 0
        )
    """)

    cur.execute("""
        CREATE TABLE IF NOT EXISTS reports (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            message_id INTEGER NOT NULL,
            reporter_id INTEGER NOT NULL,
            created_at TEXT NOT NULL,
            UNIQUE(message_id, reporter_id)
        )
    """)

    conn.commit()
    conn.close()


init_database()


# =========================================================
# USERS
# =========================================================

def create_user(user_id: int):
    conn = db()
    cur = conn.cursor()

    cur.execute(
        "SELECT user_id FROM users WHERE user_id = ?",
        (user_id,),
    )

    exists = cur.fetchone()

    if not exists:
        alias_number = 100 + (abs(user_id) % 900)

        cur.execute(
            """
            INSERT INTO users
            (user_id, alias_number, joined_at, banned)
            VALUES (?, ?, ?, 0)
            """,
            (
                user_id,
                alias_number,
                datetime.utcnow().isoformat(),
            ),
        )

    conn.commit()
    conn.close()


def get_alias_number(user_id: int) -> int:
    create_user(user_id)

    conn = db()
    cur = conn.cursor()

    cur.execute(
        "SELECT alias_number FROM users WHERE user_id = ?",
        (user_id,),
    )

    row = cur.fetchone()

    conn.close()

    return row[0]


def is_banned(user_id: int) -> bool:
    create_user(user_id)

    conn = db()
    cur = conn.cursor()

    cur.execute(
        "SELECT banned FROM users WHERE user_id = ?",
        (user_id,),
    )

    row = cur.fetchone()

    conn.close()

    return bool(row and row[0])


def set_banned(user_id: int, value: bool):
    create_user(user_id)

    conn = db()
    cur = conn.cursor()

    cur.execute(
        """
        UPDATE users
        SET banned = ?
        WHERE user_id = ?
        """,
        (1 if value else 0, user_id),
    )

    conn.commit()
    conn.close()


# =========================================================
# ROOMS
# =========================================================

def is_member(room_id: str, user_id: int) -> bool:
    conn = db()
    cur = conn.cursor()

    cur.execute(
        """
        SELECT 1
        FROM room_members
        WHERE room_id = ? AND user_id = ?
        """,
        (room_id, user_id),
    )

    result = cur.fetchone() is not None

    conn.close()

    return result


def join_room(room_id: str, user_id: int):
    if room_id not in ROOMS:
        return

    create_user(user_id)

    conn = db()
    cur = conn.cursor()

    cur.execute(
        """
        INSERT OR IGNORE INTO room_members
        (room_id, user_id, joined_at)
        VALUES (?, ?, ?)
        """,
        (
            room_id,
            user_id,
            datetime.utcnow().isoformat(),
        ),
    )

    conn.commit()
    conn.close()


def leave_room(room_id: str, user_id: int):
    conn = db()
    cur = conn.cursor()

    cur.execute(
        """
        DELETE FROM room_members
        WHERE room_id = ? AND user_id = ?
        """,
        (room_id, user_id),
    )

    conn.commit()
    conn.close()


def get_room_members(room_id: str):
    conn = db()
    cur = conn.cursor()

    cur.execute(
        """
        SELECT user_id
        FROM room_members
        WHERE room_id = ?
        """,
        (room_id,),
    )

    users = [row[0] for row in cur.fetchall()]

    conn.close()

    return users


def get_room_member_count(room_id: str):
    conn = db()
    cur = conn.cursor()

    cur.execute(
        """
        SELECT COUNT(*)
        FROM room_members
        WHERE room_id = ?
        """,
        (room_id,),
    )

    count = cur.fetchone()[0]

    conn.close()

    return count


# =========================================================
# MESSAGES
# =========================================================

def create_message(
    room_id: str,
    user_id: int,
    text: str,
    reply_to: int | None = None,
):
    alias_number = get_alias_number(user_id)

    conn = db()
    cur = conn.cursor()

    cur.execute(
        """
        INSERT INTO messages
        (
            room_id,
            user_id,
            alias_number,
            text,
            created_at,
            reply_to,
            deleted
        )
        VALUES (?, ?, ?, ?, ?, ?, 0)
        """,
        (
            room_id,
            user_id,
            alias_number,
            text,
            datetime.utcnow().isoformat(),
            reply_to,
        ),
    )

    message_id = cur.lastrowid

    conn.commit()
    conn.close()

    return message_id


def get_message(message_id: int):
    conn = db()
    cur = conn.cursor()

    cur.execute(
        """
        SELECT
            id,
            room_id,
            user_id,
            alias_number,
            text,
            created_at,
            reply_to,
            deleted
        FROM messages
        WHERE id = ?
        """,
        (message_id,),
    )

    row = cur.fetchone()

    conn.close()

    return row


def get_recent_messages(room_id: str, limit: int = 20):
    conn = db()
    cur = conn.cursor()

    cur.execute(
        """
        SELECT
            id,
            user_id,
            alias_number,
            text,
            created_at,
            reply_to
        FROM messages
        WHERE room_id = ?
        AND deleted = 0
        ORDER BY id DESC
        LIMIT ?
        """,
        (room_id, limit),
    )

    rows = cur.fetchall()

    conn.close()

    return list(reversed(rows))


def delete_message(message_id: int):
    conn = db()
    cur = conn.cursor()

    cur.execute(
        """
        UPDATE messages
        SET deleted = 1
        WHERE id = ?
        """,
        (message_id,),
    )

    conn.commit()
    conn.close()


# =========================================================
# REPORTS
# =========================================================

def create_report(message_id: int, reporter_id: int):
    conn = db()
    cur = conn.cursor()

    try:
        cur.execute(
            """
            INSERT INTO reports
            (
                message_id,
                reporter_id,
                created_at
            )
            VALUES (?, ?, ?)
            """,
            (
                message_id,
                reporter_id,
                datetime.utcnow().isoformat(),
            ),
        )

        conn.commit()
        created = True

    except sqlite3.IntegrityError:
        created = False

    conn.close()

    return created


def get_report_count(message_id: int):
    conn = db()
    cur = conn.cursor()

    cur.execute(
        """
        SELECT COUNT(*)
        FROM reports
        WHERE message_id = ?
        """,
        (message_id,),
    )

    count = cur.fetchone()[0]

    conn.close()

    return count


# =========================================================
# KEYBOARDS
# =========================================================

def main_keyboard():
    return ReplyKeyboardMarkup(
        keyboard=[
            [
                KeyboardButton(text="👥 Сообщество"),
            ],
            [
                KeyboardButton(text="👤 Профиль"),
                KeyboardButton(text="ℹ️ Как это работает"),
            ],
        ],
        resize_keyboard=True,
    )


def rooms_keyboard():
    rows = []

    for room_id, room in ROOMS.items():
        count = get_room_member_count(room_id)

        rows.append([
            InlineKeyboardButton(
                text=f"{room['name']} · {count}",
                callback_data=f"room:{room_id}",
            )
        ])

    return InlineKeyboardMarkup(inline_keyboard=rows)


def room_keyboard(room_id: str):
    return InlineKeyboardMarkup(
        inline_keyboard=[
            [
                InlineKeyboardButton(
                    text="✍️ Написать",
                    callback_data=f"write:{room_id}",
                ),
                InlineKeyboardButton(
                    text="📰 Лента",
                    callback_data=f"feed:{room_id}",
                ),
            ],
            [
                InlineKeyboardButton(
                    text="🚪 Выйти",
                    callback_data=f"leave:{room_id}",
                ),
            ],
            [
                InlineKeyboardButton(
                    text="⬅️ Комнаты",
                    callback_data="rooms",
                ),
            ],
        ]
    )


def message_keyboard(message_id: int):
    return InlineKeyboardMarkup(
        inline_keyboard=[
            [
                InlineKeyboardButton(
                    text="↩️ Ответить",
                    callback_data=f"reply:{message_id}",
                ),
                InlineKeyboardButton(
                    text="⚠️ Жалоба",
                    callback_data=f"report:{message_id}",
                ),
            ]
        ]
    )


# =========================================================
# HELPERS
# =========================================================

last_message_time = {}


def can_send(user_id: int) -> bool:
    now = time.time()

    last = last_message_time.get(user_id, 0)

    if now - last < MESSAGE_COOLDOWN:
        return False

    last_message_time[user_id] = now

    return True


def format_message(
    alias_number: int,
    text: str,
    message_id: int,
):
    return (
        f"👤 <b>Участник {alias_number}</b>\n\n"
        f"{html.escape(text)}"
    )


async def send_to_room(
    room_id: str,
    text: str,
    message_id: int,
):
    members = get_room_members(room_id)

    keyboard = message_keyboard(message_id)

    delivered = 0

    for user_id in members:
        try:
            await bot.send_message(
                user_id,
                text,
                reply_markup=keyboard,
            )

            delivered += 1

            await asyncio.sleep(0.05)

        except TelegramForbiddenError:
            # Пользователь заблокировал бота.
            leave_room(room_id, user_id)

        except TelegramBadRequest:
            pass

        except Exception:
            logging.exception(
                "Ошибка отправки пользователю %s",
                user_id,
            )

    return delivered


async def show_rooms(message: Message):
    await message.answer(
        "👥 <b>Анонимное сообщество 2К50</b>\n\n"
        "Здесь участники общаются под анонимными номерами.\n\n"
        "Выбери комнату:",
        reply_markup=rooms_keyboard(),
    )


# =========================================================
# START
# =========================================================

@router.message(CommandStart())
async def start_handler(
    message: Message,
    state: FSMContext,
):
    await state.clear()

    user_id = message.from_user.id

    create_user(user_id)

    if is_banned(user_id):
        await message.answer(
            "🚫 Доступ к сообществу ограничен."
        )
        return

    alias = get_alias_number(user_id)

    await message.answer(
        "👋 <b>Добро пожаловать в 2К50 Premium</b>\n\n"
        "Это анонимное пространство для общения.\n\n"
        f"Твой номер: <b>Участник {alias}</b>\n\n"
        "Другие участники не видят твой Telegram username, "
        "имя профиля или аватар.",
        reply_markup=main_keyboard(),
    )


# =========================================================
# MENU
# =========================================================

@router.message(Command("menu"))
@router.message(F.text == "👥 Сообщество")
async def community_handler(message: Message):
    if is_banned(message.from_user.id):
        await message.answer("🚫 Доступ ограничен.")
        return

    await show_rooms(message)


@router.message(F.text == "👤 Профиль")
async def profile_handler(message: Message):
    user_id = message.from_user.id

    alias = get_alias_number(user_id)

    conn = db()
    cur = conn.cursor()

    cur.execute(
        """
        SELECT COUNT(*)
        FROM room_members
        WHERE user_id = ?
        """,
        (user_id,),
    )

    rooms_count = cur.fetchone()[0]

    cur.execute(
        """
        SELECT COUNT(*)
        FROM messages
        WHERE user_id = ?
        AND deleted = 0
        """,
        (user_id,),
    )

    messages_count = cur.fetchone()[0]

    conn.close()

    await message.answer(
        "👤 <b>Твой профиль</b>\n\n"
        f"🆔 Анонимный номер: <b>Участник {alias}</b>\n"
        f"👥 Комнат: <b>{rooms_count}</b>\n"
        f"💬 Сообщений: <b>{messages_count}</b>\n\n"
        "Твой Telegram-профиль другим участникам не показывается.",
        reply_markup=main_keyboard(),
    )


@router.message(F.text == "ℹ️ Как это работает")
async def info_handler(message: Message):
    await message.answer(
        "ℹ️ <b>Как работает анонимность</b>\n\n"
        "Ты пишешь сообщение боту.\n"
        "Бот публикует его другим участникам под номером "
        "«Участник XXX».\n\n"
        "Другие участники не получают твой username, "
        "аватар или Telegram ID.\n\n"
        "При этом сам сервис технически знает Telegram ID "
        "аккаунта — поэтому это не абсолютная анонимность "
        "перед владельцами сервиса.",
        reply_markup=main_keyboard(),
    )


# =========================================================
# ROOMS
# =========================================================

@router.callback_query(F.data == "rooms")
async def rooms_callback(callback: CallbackQuery):
    await callback.answer()

    await callback.message.edit_text(
        "👥 <b>Комнаты</b>\n\n"
        "Выбери пространство:",
        reply_markup=rooms_keyboard(),
    )


@router.callback_query(F.data.startswith("room:"))
async def room_callback(callback: CallbackQuery):
    await callback.answer()

    room_id = callback.data.split(":", 1)[1]

    if room_id not in ROOMS:
        return

    user_id = callback.from_user.id

    if is_banned(user_id):
        await callback.message.answer(
            "🚫 Доступ ограничен."
        )
        return

    join_room(room_id, user_id)

    room = ROOMS[room_id]

    await callback.message.edit_text(
        f"{room['name']}\n\n"
        f"{room['description']}\n\n"
        "🔒 Общение анонимное.\n"
        "Нажимая «Написать», ты публикуешь сообщение "
        "под своим анонимным номером.",
        reply_markup=room_keyboard(room_id),
    )


# =========================================================
# WRITE
# =========================================================

@router.callback_query(F.data.startswith("write:"))
async def write_callback(
    callback: CallbackQuery,
    state: FSMContext,
):
    await callback.answer()

    room_id = callback.data.split(":", 1)[1]

    if room_id not in ROOMS:
        return

    user_id = callback.from_user.id

    if is_banned(user_id):
        await callback.message.answer(
            "🚫 Доступ ограничен."
        )
        return

    join_room(room_id, user_id)

    await state.update_data(
        room_id=room_id,
        reply_to=None,
    )

    await state.set_state(
        WriteMessage.waiting_for_text
    )

    await callback.message.answer(
        f"✍️ <b>{ROOMS[room_id]['name']}</b>\n\n"
        "Напиши сообщение.\n\n"
        "Максимум: 2000 символов.\n"
        "Для отмены нажми /cancel",
    )


@router.message(Command("cancel"))
async def cancel_handler(
    message: Message,
    state: FSMContext,
):
    current_state = await state.get_state()

    if current_state is None:
        await message.answer(
            "Нечего отменять.",
            reply_markup=main_keyboard(),
        )
        return

    await state.clear()

    await message.answer(
        "❌ Действие отменено.",
        reply_markup=main_keyboard(),
    )


@router.message(WriteMessage.waiting_for_text)
async def write_message_handler(
    message: Message,
    state: FSMContext,
):
    user_id = message.from_user.id

    if is_banned(user_id):
        await state.clear()
        await message.answer(
            "🚫 Доступ ограничен."
        )
        return

    if not message.text:
        await message.answer(
            "Отправь именно текстовое сообщение."
        )
        return

    if not can_send(user_id):
        await message.answer(
            "⏳ Подожди несколько секунд перед следующим сообщением."
        )
        return

    text = message.text.strip()

    if not text:
        await message.answer(
            "Сообщение не может быть пустым."
        )
        return

    if len(text) > MAX_MESSAGE_LENGTH:
        await message.answer(
            f"Слишком длинное сообщение.\n"
            f"Максимум — {MAX_MESSAGE_LENGTH} символов."
        )
        return

    data = await state.get_data()

    room_id = data.get("room_id")
    reply_to = data.get("reply_to")

    if not room_id or room_id not in ROOMS:
        await state.clear()
        await message.answer(
            "Комната не найдена."
        )
        return

    if not is_member(room_id, user_id):
        join_room(room_id, user_id)

    message_id = create_message(
        room_id=room_id,
        user_id=user_id,
        text=text,
        reply_to=reply_to,
    )

    alias = get_alias_number(user_id)

    formatted = format_message(
        alias,
        text,
        message_id,
    )

    if reply_to:
        original = get_message(reply_to)

        if original:
            original_alias = original[3]

            formatted = (
                f"↩️ <b>Ответ Участнику {original_alias}</b>\n\n"
                f"👤 <b>Участник {alias}</b>\n\n"
                f"{html.escape(text)}"
            )

    await send_to_room(
        room_id,
        formatted,
        message_id,
    )

    await state.clear()

    await message.answer(
        "✅ <b>Сообщение опубликовано.</b>\n\n"
        f"Ты — <b>Участник {alias}</b>.",
        reply_markup=room_keyboard(room_id),
    )


# =========================================================
# FEED
# =========================================================

@router.callback_query(F.data.startswith("feed:"))
async def feed_callback(callback: CallbackQuery):
    await callback.answer()

    room_id = callback.data.split(":", 1)[1]

    if room_id not in ROOMS:
        return

    user_id = callback.from_user.id

    if is_banned(user_id):
        return

    if not is_member(room_id, user_id):
        join_room(room_id, user_id)

    messages = get_recent_messages(
        room_id,
        limit=20,
    )

    if not messages:
        await callback.message.answer(
            f"{ROOMS[room_id]['name']}\n\n"
            "📰 Здесь пока нет сообщений.\n\n"
            "Будь первым — напиши что-нибудь.",
            reply_markup=room_keyboard(room_id),
        )
        return

    await callback.message.answer(
        f"📰 <b>{ROOMS[room_id]['name']}</b>\n"
        f"Последние {len(messages)} сообщений:"
    )

    for (
        message_id,
        user_id,
        alias_number,
        text,
        created_at,
        reply_to,
    ) in messages:

        prefix = ""

        if reply_to:
            original = get_message(reply_to)

            if original:
                prefix = (
                    f"↩️ Ответ Участнику "
                    f"{original[3]}\n\n"
                )

        output = (
            f"{prefix}"
            f"👤 <b>Участник {alias_number}</b>\n\n"
            f"{html.escape(text)}"
        )

        await callback.message.answer(
            output,
            reply_markup=message_keyboard(message_id),
        )

        await asyncio.sleep(0.03)


# =========================================================
# REPLY
# =========================================================

@router.callback_query(F.data.startswith("reply:"))
async def reply_callback(
    callback: CallbackQuery,
    state: FSMContext,
):
    await callback.answer()

    message_id = int(
        callback.data.split(":", 1)[1]
    )

    original = get_message(message_id)

    if not original or original[7]:
        await callback.message.answer(
            "Сообщение больше недоступно."
        )
        return

    room_id = original[1]

    if not is_member(
        room_id,
        callback.from_user.id,
    ):
        join_room(
            room_id,
            callback.from_user.id,
        )

    await state.update_data(
        room_id=room_id,
        reply_to=message_id,
    )

    await state.set_state(
        WriteMessage.waiting_for_text
    )

    await callback.message.answer(
        f"↩️ <b>Ответ Участнику {original[3]}</b>\n\n"
        "Напиши свой ответ.\n"
        "Для отмены — /cancel",
    )


# =========================================================
# REPORT
# =========================================================

@router.callback_query(F.data.startswith("report:"))
async def report_callback(callback: CallbackQuery):
    message_id = int(
        callback.data.split(":", 1)[1]
    )

    user_id = callback.from_user.id

    original = get_message(message_id)

    if not original:
        await callback.answer(
            "Сообщение не найдено.",
            show_alert=True,
        )
        return

    if original[2] == user_id:
        await callback.answer(
            "Нельзя пожаловаться на своё сообщение.",
            show_alert=True,
        )
        return

    created = create_report(
        message_id,
        user_id,
    )

    if not created:
        await callback.answer(
            "Ты уже отправлял жалобу.",
            show_alert=True,
        )
        return

    count = get_report_count(message_id)

    await callback.answer(
        "⚠️ Жалоба отправлена модератору.",
        show_alert=True,
    )

    if ADMIN_CHAT_ID:
        try:
            await bot.send_message(
                int(ADMIN_CHAT_ID),
                "⚠️ <b>НОВАЯ ЖАЛОБА</b>\n\n"
                f"Сообщение: <code>{message_id}</code>\n"
                f"Комната: <code>{original[1]}</code>\n"
                f"Жалоб: <b>{count}</b>\n\n"
                f"Текст:\n"
                f"{html.escape(original[4])}\n\n"
                f"Отправитель сообщения:\n"
                f"<code>{original[2]}</code>",
            )

        except Exception:
            logging.exception(
                "Не удалось отправить жалобу админу"
            )


# =========================================================
# LEAVE
# =========================================================

@router.callback_query(F.data.startswith("leave:"))
async def leave_callback(callback: CallbackQuery):
    await callback.answer()

    room_id = callback.data.split(":", 1)[1]

    if room_id not in ROOMS:
        return

    leave_room(
        room_id,
        callback.from_user.id,
    )

    await callback.message.edit_text(
        "🚪 <b>Ты вышел из комнаты.</b>\n\n"
        "Старые сообщения остаются в истории.",
        reply_markup=InlineKeyboardMarkup(
            inline_keyboard=[
                [
                    InlineKeyboardButton(
                        text="👥 Все комнаты",
                        callback_data="rooms",
                    )
                ]
            ]
        ),
    )


# =========================================================
# ADMIN
# =========================================================

def is_admin(user_id: int) -> bool:
    if not ADMIN_CHAT_ID:
        return False

    try:
        return user_id == int(ADMIN_CHAT_ID)
    except ValueError:
        return False


@router.message(Command("stats"))
async def stats_handler(message: Message):
    if not is_admin(message.from_user.id):
        return

    conn = db()
    cur = conn.cursor()

    cur.execute("SELECT COUNT(*) FROM users")
    users = cur.fetchone()[0]

    cur.execute(
        "SELECT COUNT(*) FROM room_members"
    )
    members = cur.fetchone()[0]

    cur.execute(
        """
        SELECT COUNT(*)
        FROM messages
        WHERE deleted = 0
        """
    )
    messages = cur.fetchone()[0]

    cur.execute(
        "SELECT COUNT(*) FROM reports"
    )
    reports = cur.fetchone()[0]

    conn.close()

    await message.answer(
        "📊 <b>Статистика Premium</b>\n\n"
        f"👤 Пользователей: <b>{users}</b>\n"
        f"👥 Вступлений в комнаты: <b>{members}</b>\n"
        f"💬 Сообщений: <b>{messages}</b>\n"
        f"⚠️ Жалоб: <b>{reports}</b>"
    )


@router.message(Command("ban"))
async def ban_handler(message: Message):
    if not is_admin(message.from_user.id):
        return

    parts = message.text.split()

    if len(parts) != 2:
        await message.answer(
            "Использование:\n"
            "/ban USER_ID"
        )
        return

    try:
        user_id = int(parts[1])
    except ValueError:
        await message.answer(
            "USER_ID должен быть числом."
        )
        return

    set_banned(user_id, True)

    await message.answer(
        f"🚫 Пользователь <code>{user_id}</code> заблокирован."
    )


@router.message(Command("unban"))
async def unban_handler(message: Message):
    if not is_admin(message.from_user.id):
        return

    parts = message.text.split()

    if len(parts) != 2:
        await message.answer(
            "Использование:\n"
            "/unban USER_ID"
        )
        return

    try:
        user_id = int(parts[1])
    except ValueError:
        await message.answer(
            "USER_ID должен быть числом."
        )
        return

    set_banned(user_id, False)

    await message.answer(
        f"✅ Пользователь <code>{user_id}</code> разблокирован."
    )


@router.message(Command("delete"))
async def delete_handler(message: Message):
    if not is_admin(message.from_user.id):
        return

    parts = message.text.split()

    if len(parts) != 2:
        await message.answer(
            "Использование:\n"
            "/delete MESSAGE_ID"
        )
        return

    try:
        message_id = int(parts[1])
    except ValueError:
        await message.answer(
            "MESSAGE_ID должен быть числом."
        )
        return

    original = get_message(message_id)

    if not original:
        await message.answer(
            "Сообщение не найдено."
        )
        return

    delete_message(message_id)

    await message.answer(
        f"🗑 Сообщение <code>{message_id}</code> удалено."
    )


# =========================================================
# FALLBACK
# =========================================================

@router.message()
async def fallback_handler(message: Message):
    if is_banned(message.from_user.id):
        await message.answer(
            "🚫 Доступ ограничен."
        )
        return

    await message.answer(
        "Выбери действие в меню 👇",
        reply_markup=main_keyboard(),
    )


# =========================================================
# CLEANUP
# =========================================================

async def cleanup_old_messages():
    while True:
        try:
            limit = (
                datetime.utcnow()
                - timedelta(days=30)
            ).isoformat()

            conn = db()
            cur = conn.cursor()

            cur.execute(
                """
                DELETE FROM messages
                WHERE created_at < ?
                AND deleted = 1
                """,
                (limit,),
            )

            conn.commit()
            conn.close()

        except Exception:
            logging.exception(
                "Ошибка очистки старых сообщений"
            )

        await asyncio.sleep(3600)


# =========================================================
# MAIN
# =========================================================

async def main():
    logging.basicConfig(
        level=logging.INFO,
    )

    logging.info(
        "Premium bot запускается..."
    )

    cleanup_task = asyncio.create_task(
        cleanup_old_messages()
    )

    try:
        await dp.start_polling(
            bot,
            allowed_updates=dp.resolve_used_update_types(),
        )
    finally:
        cleanup_task.cancel()

        await bot.session.close()


if __name__ == "__main__":
    asyncio.run(main())
