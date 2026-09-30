# -*- coding: utf-8 -*-

import asyncio
import logging
import sqlite3
import random
import string
from decimal import Decimal, ROUND_HALF_UP

import aiohttp
from aiogram import Bot, Dispatcher, F
from aiogram.filters import CommandStart
from aiogram.types import (
    Message,
    CallbackQuery,
    InlineKeyboardMarkup,
    InlineKeyboardButton,
)
from aiogram.enums import ParseMode
from aiogram.client.default import DefaultBotProperties


# ============================================================
#                     НАСТРОЙКИ
# ============================================================

# ВСТАВЬ СЮДА НОВЫЙ ТОКЕН TELEGRAM-БОТА
BOT_TOKEN = "8868297876:AAHfQPx0RVng8uuYuKk-cu_1n5_CxG56nJ0"

# ВСТАВЬ СЮДА ТОКЕН CRYPTO BOT API
CRYPTO_PAY_TOKEN = "640478:AAkwM9d8MaHsJFssWQafVdHIuaE2JtYvOTF"

# Твой username поддержки
SUPPORT_USERNAME = "@fuhbfbhfbh"

# Цены
NORMAL_PRICE = Decimal("0.30")
PARSING_PRICE = Decimal("0.50")


# ============================================================
#                     ЛОГИРОВАНИЕ
# ============================================================

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s | %(levelname)s | %(message)s"
)


# ============================================================
#                     БАЗА ДАННЫХ
# ============================================================

DB_NAME = "proxy_shop.db"

db = sqlite3.connect(DB_NAME)
cursor = db.cursor()

cursor.execute("""
CREATE TABLE IF NOT EXISTS orders (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL,
    country TEXT NOT NULL,
    category TEXT NOT NULL,
    quantity INTEGER NOT NULL,
    amount TEXT NOT NULL,
    invoice_id TEXT,
    status TEXT DEFAULT 'pending',
    proxies TEXT
)
""")

db.commit()


# ============================================================
#                     BOT / DISPATCHER
# ============================================================

bot = Bot(
    token=BOT_TOKEN,
    default=DefaultBotProperties(
        parse_mode=ParseMode.HTML
    )
)

dp = Dispatcher()


# ============================================================
#                     ДАННЫЕ
# ============================================================

COUNTRIES = {
    "cuba": "🇨🇺 Куба",
    "indonesia": "🇮🇩 Индонезия",
    "lithuania": "🇱🇹 Литва",
    "russia": "🇷🇺 Россия",
}

CATEGORIES = {
    "normal": {
        "name": "🔒 Обычные прокси",
        "price": NORMAL_PRICE,
        "description": (
            "Обычные прокси подойдут для вашей защиты и "
            "конфиденциальности. 🛡️\n\n"
            "Хороший вариант для повседневных задач, "
            "где важны приватность и стабильное подключение."
        )
    },
    "parsing": {
        "name": "📊 Прокси для парсинга логов",
        "price": PARSING_PRICE,
        "description": (
            "Идеальные прокси для отработки логов и парсинга. 📊\n\n"
            "Абсолютно конфиденциальное использование и удобный "
            "вариант для задач, связанных с обработкой данных."
        )
    }
}


# ============================================================
#                     ВРЕМЕННОЕ СОСТОЯНИЕ
# ============================================================

# user_id -> выбранные параметры
user_state = {}


# ============================================================
#                     КЛАВИАТУРЫ
# ============================================================

def main_menu():
    return InlineKeyboardMarkup(
        inline_keyboard=[
            [
                InlineKeyboardButton(
                    text="🛒 Купить прокси",
                    callback_data="buy"
                )
            ],
            [
                InlineKeyboardButton(
                    text="💬 Поддержка",
                    callback_data="support"
                )
            ]
        ]
    )


def countries_menu():
    return InlineKeyboardMarkup(
        inline_keyboard=[
            [
                InlineKeyboardButton(
                    text="🇨🇺 Куба",
                    callback_data="country:cuba"
                ),
                InlineKeyboardButton(
                    text="🇮🇩 Индонезия",
                    callback_data="country:indonesia"
                )
            ],
            [
                InlineKeyboardButton(
                    text="🇱🇹 Литва",
                    callback_data="country:lithuania"
                ),
                InlineKeyboardButton(
                    text="🇷🇺 Россия",
                    callback_data="country:russia"
                )
            ],
            [
                InlineKeyboardButton(
                    text="◀️ Назад",
                    callback_data="home"
                )
            ]
        ]
    )


def categories_menu(country):
    return InlineKeyboardMarkup(
        inline_keyboard=[
            [
                InlineKeyboardButton(
                    text="🔒 Обычные — $0.30",
                    callback_data=f"category:normal:{country}"
                )
            ],
            [
                InlineKeyboardButton(
                    text="📊 Для парсинга — $0.50",
                    callback_data=f"category:parsing:{country}"
                )
            ],
            [
                InlineKeyboardButton(
                    text="◀️ Назад",
                    callback_data="buy"
                )
            ]
        ]
    )


def quantity_menu():
    return InlineKeyboardMarkup(
        inline_keyboard=[
            [
                InlineKeyboardButton(
                    text="10",
                    callback_data="qty:10"
                ),
                InlineKeyboardButton(
                    text="50",
                    callback_data="qty:50"
                ),
                InlineKeyboardButton(
                    text="100",
                    callback_data="qty:100"
                )
            ],
            [
                InlineKeyboardButton(
                    text="120",
                    callback_data="qty:120"
                ),
                InlineKeyboardButton(
                    text="250",
                    callback_data="qty:250"
                ),
                InlineKeyboardButton(
                    text="500",
                    callback_data="qty:500"
                )
            ],
            [
                InlineKeyboardButton(
                    text="✏️ Другое количество",
                    callback_data="qty:custom"
                )
            ],
            [
                InlineKeyboardButton(
                    text="◀️ Назад",
                    callback_data="back_category"
                )
            ]
        ]
    )


def payment_menu():
    return InlineKeyboardMarkup(
        inline_keyboard=[
            [
                InlineKeyboardButton(
                    text="💳 Оплатить через Crypto Bot",
                    callback_data="pay"
                )
            ],
            [
                InlineKeyboardButton(
                    text="◀️ Назад",
                    callback_data="back_quantity"
                )
            ]
        ]
    )


# ============================================================
#                     ГЛАВНОЕ МЕНЮ
# ============================================================

@dp.message(CommandStart())
async def start(message: Message):

    user_state.pop(message.from_user.id, None)

    text = (
        "👋 <b>Добро пожаловать в Proxy Shop!</b>\n\n"
        "🔐 Здесь вы можете приобрести прокси для различных задач.\n\n"
        "🛒 Выберите нужный раздел ниже:"
    )

    await message.answer(
        text,
        reply_markup=main_menu()
    )


# ============================================================
#                     КУПИТЬ
# ============================================================

@dp.callback_query(F.data == "buy")
async def buy_proxy(callback: CallbackQuery):

    await callback.answer()

    await callback.message.edit_text(
        "🌍 <b>Выберите страну прокси</b>\n\n"
        "Доступные локации:",
        reply_markup=countries_menu()
    )


# ============================================================
#                     ВЫБОР СТРАНЫ
# ============================================================

@dp.callback_query(F.data.startswith("country:"))
async def select_country(callback: CallbackQuery):

    await callback.answer()

    country = callback.data.split(":")[1]

    user_state[callback.from_user.id] = {
        "country": country
    }

    country_name = COUNTRIES[country]

    await callback.message.edit_text(
        f"🌍 <b>{country_name}</b>\n\n"
        "Выберите категорию прокси:\n\n"

        "🔒 <b>Обычные прокси</b> — $0.30/шт.\n"
        "Для защиты, приватности и повседневных задач.\n\n"

        "📊 <b>Прокси для парсинга логов</b> — $0.50/шт.\n"
        "Для обработки данных и задач парсинга.",
        reply_markup=categories_menu(country)
    )


# ============================================================
#                     ВЫБОР КАТЕГОРИИ
# ============================================================

@dp.callback_query(F.data.startswith("category:"))
async def select_category(callback: CallbackQuery):

    await callback.answer()

    _, category, country = callback.data.split(":")

    user_state[callback.from_user.id] = {
        "country": country,
        "category": category
    }

    category_data = CATEGORIES[category]

    await callback.message.edit_text(
        f"{category_data['name']}\n\n"
        f"{category_data['description']}\n\n"
        f"💵 Цена: <b>${category_data['price']:.2f}</b> за 1 прокси\n\n"
        "🔢 <b>Выберите количество:</b>",
        reply_markup=quantity_menu()
    )


# ============================================================
#                     КОЛИЧЕСТВО
# ============================================================

@dp.callback_query(F.data.startswith("qty:"))
async def select_quantity(callback: CallbackQuery):

    await callback.answer()

    value = callback.data.split(":")[1]
    user_id = callback.from_user.id

    if value == "custom":

        user_state.setdefault(user_id, {})
        user_state[user_id]["waiting_quantity"] = True

        await callback.message.edit_text(
            "✏️ <b>Введите количество прокси</b>\n\n"
            "Например:\n"
            "<code>120</code>\n\n"
            "Минимум: <b>1</b>\n"
            "Максимум: <b>5000</b>"
        )

        return

    quantity = int(value)

    await show_order(callback.message, user_id, quantity)


# ============================================================
#                     РУЧНОЕ КОЛИЧЕСТВО
# ============================================================

@dp.message(F.text)
async def custom_quantity(message: Message):

    user_id = message.from_user.id
    state = user_state.get(user_id)

    if not state:
        return

    if not state.get("waiting_quantity"):
        return

    try:
        quantity = int(message.text.strip())
    except ValueError:

        await message.answer(
            "❌ Введите количество целым числом.\n\n"
            "Например: <code>120</code>"
        )

        return

    if quantity < 1 or quantity > 5000:

        await message.answer(
            "❌ Количество должно быть от <b>1</b> до <b>5000</b>."
        )

        return

    state["waiting_quantity"] = False

    await show_order(message, user_id, quantity)


# ============================================================
#                     ФОРМИРОВАНИЕ ЗАКАЗА
# ============================================================

async def show_order(message, user_id, quantity):

    state = user_state.get(user_id)

    if not state:
        return

    country = state["country"]
    category = state["category"]

    price = CATEGORIES[category]["price"]

    amount = (
        price * Decimal(quantity)
    ).quantize(
        Decimal("0.01"),
        rounding=ROUND_HALF_UP
    )

    state["quantity"] = quantity
    state["amount"] = amount

    await message.answer(
        "🧾 <b>Ваш заказ</b>\n\n"
        f"🌍 Страна: <b>{COUNTRIES[country]}</b>\n"
        f"📦 Категория: <b>{CATEGORIES[category]['name']}</b>\n"
        f"🔢 Количество: <b>{quantity}</b>\n"
        f"💵 Цена за шт.: <b>${price:.2f}</b>\n\n"
        f"💰 <b>Итого: ${amount:.2f}</b>\n\n"
        "Нажмите кнопку ниже для создания счёта.",
        reply_markup=payment_menu()
    )


# ============================================================
#                     CRYPTO PAY API
# ============================================================

CRYPTO_API = "https://pay.crypt.bot/api"


async def crypto_api(method, data=None):

    headers = {
        "Crypto-Pay-API-Token": CRYPTO_PAY_TOKEN
    }

    async with aiohttp.ClientSession() as session:

        async with session.post(
            f"{CRYPTO_API}/{method}",
            headers=headers,
            json=data or {}
        ) as response:

            result = await response.json()

            if not result.get("ok"):
                logging.error(
                    "Crypto Pay error: %s",
                    result
                )

            return result


# ============================================================
#                     СОЗДАНИЕ INVOICE
# ============================================================

async def create_invoice(amount, payload):

    result = await crypto_api(
        "createInvoice",
        {
            "currency_type": "fiat",
            "fiat": "USD",
            "amount": str(amount),
            "description": "Proxy Shop",
            "payload": str(payload),
            "paid_btn_name": "callback",
            "paid_btn_url": "https://t.me/"
        }
    )

    if not result.get("ok"):
        return None

    return result["result"]


# ============================================================
#                     ОПЛАТА
# ============================================================

@dp.callback_query(F.data == "pay")
async def create_payment(callback: CallbackQuery):

    await callback.answer()

    user_id = callback.from_user.id
    state = user_state.get(user_id)

    if not state:
        await callback.message.edit_text(
            "❌ Сессия заказа истекла.\n\n"
            "Начните оформление заново.",
            reply_markup=main_menu()
        )
        return

    country = state["country"]
    category = state["category"]
    quantity = state["quantity"]
    amount = state["amount"]

    cursor.execute(
        """
        INSERT INTO orders
        (user_id, country, category, quantity, amount, status)
        VALUES (?, ?, ?, ?, ?, ?)
        """,
        (
            user_id,
            country,
            category,
            quantity,
            str(amount),
            "pending"
        )
    )

    order_id = cursor.lastrowid
    db.commit()

    invoice = await create_invoice(
        amount,
        order_id
    )

    if not invoice:

        cursor.execute(
            "UPDATE orders SET status=? WHERE id=?",
            ("error", order_id)
        )

        db.commit()

        await callback.message.edit_text(
            "❌ Не удалось создать счёт.\n\n"
            "Попробуйте ещё раз позже.",
            reply_markup=main_menu()
        )

        return

    invoice_id = invoice["invoice_id"]
    pay_url = invoice.get("bot_invoice_url")

    cursor.execute(
        """
        UPDATE orders
        SET invoice_id=?
        WHERE id=?
        """,
        (str(invoice_id), order_id)
    )

    db.commit()

    keyboard = InlineKeyboardMarkup(
        inline_keyboard=[
            [
                InlineKeyboardButton(
                    text="💳 Оплатить",
                    url=pay_url
                )
            ],
            [
                InlineKeyboardButton(
                    text="🔄 Проверить оплату",
                    callback_data=f"check:{order_id}"
                )
            ],
            [
                InlineKeyboardButton(
                    text="◀️ Назад",
                    callback_data="home"
                )
            ]
        ]
    )

    await callback.message.edit_text(
        "💳 <b>Счёт создан!</b>\n\n"
        f"🧾 Заказ: <code>#{order_id}</code>\n"
        f"📦 Количество: <b>{quantity}</b>\n"
        f"💰 Сумма: <b>${amount:.2f}</b>\n\n"
        "1️⃣ Нажмите «Оплатить».\n"
        "2️⃣ Завершите оплату в Crypto Bot.\n"
        "3️⃣ Вернитесь сюда и нажмите «Проверить оплату».",
        reply_markup=keyboard
    )


# ============================================================
#                     ПРОВЕРКА ОПЛАТЫ
# ============================================================

async def get_invoice(invoice_id):

    result = await crypto_api(
        "getInvoices",
        {
            "invoice_ids": str(invoice_id)
        }
    )

    if not result.get("ok"):
        return None

    invoices = result["result"]["items"]

    if not invoices:
        return None

    return invoices[0]


@dp.callback_query(F.data.startswith("check:"))
async def check_payment(callback: CallbackQuery):

    await callback.answer()

    order_id = int(
        callback.data.split(":")[1]
    )

    cursor.execute(
        """
        SELECT user_id, country, category,
               quantity, amount, invoice_id, status
        FROM orders
        WHERE id=?
        """,
        (order_id,)
    )

    order = cursor.fetchone()

    if not order:

        await callback.message.edit_text(
            "❌ Заказ не найден.",
            reply_markup=main_menu()
        )

        return

    (
        user_id,
        country,
        category,
        quantity,
        amount,
        invoice_id,
        status
    ) = order

    if status == "paid":

        await callback.message.edit_text(
            "✅ Этот заказ уже был выдан.",
            reply_markup=main_menu()
        )

        return

    invoice = await get_invoice(invoice_id)

    if not invoice:

        await callback.message.answer(
            "❌ Не удалось проверить счёт."
        )

        return

    invoice_status = invoice.get("status")

    if invoice_status != "paid":

        await callback.answer(
            "⏳ Оплата пока не подтверждена.",
            show_alert=True
        )

        return

    # Защита от повторной выдачи
    cursor.execute(
        "SELECT status FROM orders WHERE id=?",
        (order_id,)
    )

    current_status = cursor.fetchone()[0]

    if current_status == "paid":

        await callback.message.edit_text(
            "✅ Этот заказ уже был выдан.",
            reply_markup=main_menu()
        )

        return

    # Генерируем товар
    proxies = generate_proxies(
        quantity
    )

    proxy_text = "\n".join(proxies)

    cursor.execute(
        """
        UPDATE orders
        SET status=?, proxies=?
        WHERE id=?
        """,
        (
            "paid",
            proxy_text,
            order_id
        )
    )

    db.commit()

    await send_proxies(
        callback.message,
        country,
        category,
        quantity,
        amount,
        proxy_text
    )


# ============================================================
#                     ГЕНЕРАЦИЯ ПРОКСИ
# ============================================================

def random_ip():
    """
    ДЕМОНСТРАЦИОННЫЙ генератор формата IP:PORT.

    ВАЖНО:
    Эти адреса НЕ являются гарантированно рабочими прокси.
    Для реального магазина здесь нужно подключить поставщика
    прокси или собственную базу реальных IP.
    """

    octets = [
        str(random.randint(1, 223)),
        str(random.randint(0, 255)),
        str(random.randint(0, 255)),
        str(random.randint(1, 254))
    ]

    port = random.choice([
        80,
        8080,
        3128,
        8000,
        8888
    ])

    return ".".join(octets) + ":" + str(port)


def generate_proxies(quantity):

    result = []

    used = set()

    while len(result) < quantity:

        proxy = random_ip()

        if proxy not in used:

            used.add(proxy)
            result.append(proxy)

    return result


# ============================================================
#                     ВЫДАЧА ТОВАРА
# ============================================================

async def send_proxies(
    message,
    country,
    category,
    quantity,
    amount,
    proxy_text
):

    category_name = CATEGORIES[category]["name"]
    country_name = COUNTRIES[country]

    # Telegram ограничивает размер сообщения.
    # Поэтому большие заказы отправляем несколькими сообщениями.

    header = (
        "🎉 <b>Ваш товар успешно получен!</b>\n\n"
        f"📦 Категория: <b>{category_name}</b>\n"
        f"🌍 Страна: <b>{country_name}</b>\n"
        f"🔢 Количество: <b>{quantity}</b>\n"
        f"💰 Оплачено: <b>${amount}</b>\n\n"
        "🔐 <b>Ваши прокси:</b>\n\n"
    )

    # Отправляем заголовок
    await message.edit_text(
        header
    )

    # Telegram message limit примерно 4096 символов.
    # Разбиваем прокси на части.
    lines = proxy_text.split("\n")

    chunk = ""

    for line in lines:

        if len(chunk) + len(line) + 1 > 3500:

            await message.answer(
                f"<code>{chunk}</code>"
            )

            chunk = ""

        chunk += line + "\n"

    if chunk:

        await message.answer(
            f"<code>{chunk}</code>"
        )

    await message.answer(
        "✅ <b>Спасибо за покупку!</b>\n\n"
        "Если возникли проблемы с заказом, "
        f"обратитесь в поддержку: {SUPPORT_USERNAME}",
        reply_markup=main_menu()
    )


# ============================================================
#                     ПОДДЕРЖКА
# ============================================================

@dp.callback_query(F.data == "support")
async def support(callback: CallbackQuery):

    await callback.answer()

    keyboard = InlineKeyboardMarkup(
        inline_keyboard=[
            [
                InlineKeyboardButton(
                    text="💬 Написать в поддержку",
                    url="https://t.me/uvgvdgvdgv7dgvdgv"
                )
            ],
            [
                InlineKeyboardButton(
                    text="◀️ Назад",
                    callback_data="home"
                )
            ]
        ]
    )

    await callback.message.edit_text(
        "💬 <b>Поддержка</b>\n\n"
        "Возникли вопросы по покупке или оплате?\n\n"
        "Мы поможем разобраться с заказом, "
        "оплатой и другими вопросами.\n\n"
        f"👤 Поддержка: <b>{SUPPORT_USERNAME}</b>",
        reply_markup=keyboard
    )


# ============================================================
#                     НАЗАД / ГЛАВНОЕ
# ============================================================

@dp.callback_query(F.data == "home")
async def home(callback: CallbackQuery):

    await callback.answer()

    user_state.pop(
        callback.from_user.id,
        None
    )

    await callback.message.edit_text(
        "🏠 <b>Главное меню</b>\n\n"
        "Выберите нужный раздел:",
        reply_markup=main_menu()
    )


@dp.callback_query(F.data == "back_category")
async def back_category(callback: CallbackQuery):

    await callback.answer()

    user_id = callback.from_user.id
    state = user_state.get(user_id)

    if not state:

        await callback.message.edit_text(
            "🏠 <b>Главное меню</b>",
            reply_markup=main_menu()
        )

        return

    country = state["country"]

    await callback.message.edit_text(
        f"🌍 <b>{COUNTRIES[country]}</b>\n\n"
        "Выберите категорию:",
        reply_markup=categories_menu(country)
    )


@dp.callback_query(F.data == "back_quantity")
async def back_quantity(callback: CallbackQuery):

    await callback.answer()

    await callback.message.edit_text(
        "🔢 <b>Выберите количество прокси:</b>",
        reply_markup=quantity_menu()
    )


# ============================================================
#                     ЗАПУСК
# ============================================================

async def main():

    if BOT_TOKEN == "ВСТАВЬ_НОВЫЙ_ТОКЕН_СЮДА":
        print(
            "ОШИБКА: вставьте новый токен Telegram-бота "
            "в переменную BOT_TOKEN."
        )
        return

    if CRYPTO_PAY_TOKEN == "ВСТАВЬ_ТОКЕН_CRYPTO_BOT_СЮДА":
        print(
            "ОШИБКА: вставьте токен Crypto Bot API "
            "в переменную CRYPTO_PAY_TOKEN."
        )
        return

    logging.info("Бот запускается...")

    await dp.start_polling(
        bot,
        allowed_updates=dp.resolve_used_update_types()
    )


if __name__ == "__main__":
    asyncio.run(main())
