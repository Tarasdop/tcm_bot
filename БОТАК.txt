#!/usr/bin/env python3
"""
TradeChinaMarket Lead Bot (@Tcmmarket_bot)
Владелец: @sbc_market
Парсит открытые источники: 2GIS, Авито, Яндекс Карты
"""

import asyncio
import aiohttp
import re
import csv
import io
import logging
from datetime import datetime
from aiogram import Bot, Dispatcher, types, F
from aiogram.filters import Command
from aiogram.types import (
    InlineKeyboardMarkup, InlineKeyboardButton,
    ReplyKeyboardMarkup, KeyboardButton,
    BufferedInputFile
)
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup
from aiogram.fsm.storage.memory import MemoryStorage

# ─── НАСТРОЙКИ ───────────────────────────────────────────────
BOT_TOKEN = "8606254344:AAGZX9M0VzVbRvCKOYia2c6_21FqcFVXy58"  # Новый токен от @BotFather
ADMIN_USERNAME = "sbc_market"           # Твой Telegram без @
ADMIN_IDS = []                          # Заполнится автоматически при первом входе

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

bot = Bot(token=BOT_TOKEN)
dp = Dispatcher(storage=MemoryStorage())

# ─── СОСТОЯНИЯ ───────────────────────────────────────────────
class SearchState(StatesGroup):
    waiting_city = State()
    waiting_category = State()
    waiting_query = State()

# ─── КАТЕГОРИИ БИЗНЕСА ───────────────────────────────────────
CATEGORIES = {
    "🛍 Маркетплейсы (WB/Ozon)": "wildberries ozon селлер маркетплейс",
    "🏭 Оптовая торговля": "оптовая торговля оптовик склад",
    "🏗 Строительные материалы": "строительные материалы стройматериалы",
    "👗 Одежда и текстиль": "одежда текстиль швейное производство",
    "📱 Электроника": "электроника гаджеты телефоны оптом",
    "🍽 Общепит и рестораны": "ресторан кафе общепит поставщик",
    "💄 Косметика и красота": "косметика красота салон красоты",
    "🚗 Автозапчасти": "автозапчасти автомагазин авторемонт",
    "🌿 Продукты питания": "продукты питания оптом food",
    "📦 Интернет-магазины": "интернет-магазин онлайн магазин",
}

CITIES = [
    "Москва", "Санкт-Петербург", "Новосибирск", "Екатеринбург",
    "Казань", "Нижний Новгород", "Челябинск", "Самара",
    "Омск", "Ростов-на-Дону", "Уфа", "Красноярск",
    "Воронеж", "Пермь", "Волгоград"
]

# ─── ПАРСЕРЫ ─────────────────────────────────────────────────

async def parse_2gis(city: str, query: str, limit: int = 20) -> list:
    """Парсинг 2GIS API (открытые данные)"""
    results = []
    try:
        # 2GIS Public API
        url = "https://catalog.api.2gis.com/3.0/items"
        params = {
            "q": query,
            "location": city,
            "type": "branch",
            "fields": "items.point,items.name,items.address,items.contact_groups,items.rubrics",
            "page_size": limit,
            "key": "rurbbn2595"  # Публичный демо-ключ 2GIS
        }
        async with aiohttp.ClientSession() as session:
            async with session.get(url, params=params, timeout=aiohttp.ClientTimeout(total=10)) as resp:
                if resp.status == 200:
                    data = await resp.json()
                    items = data.get("result", {}).get("items", [])
                    for item in items:
                        name = item.get("name", "")
                        address = item.get("address", {}).get("name", "")
                        phones = []
                        for cg in item.get("contact_groups", []):
                            for contact in cg.get("contacts", []):
                                if contact.get("type") == "phone":
                                    phones.append(contact.get("value", ""))
                        if phones:
                            results.append({
                                "source": "2GIS",
                                "name": name,
                                "address": address,
                                "phones": ", ".join(phones[:2]),
                                "city": city,
                                "category": query
                            })
    except Exception as e:
        logger.error(f"2GIS error: {e}")
    return results


async def parse_avito(city: str, query: str, limit: int = 15) -> list:
    """Парсинг Авито (публичные объявления)"""
    results = []
    try:
        city_map = {
            "Москва": "moskva", "Санкт-Петербург": "sankt-peterburg",
            "Новосибирск": "novosibirsk", "Екатеринбург": "ekaterinburg",
            "Казань": "kazan", "Нижний Новгород": "nizhniy_novgorod",
            "Челябинск": "chelyabinsk", "Самара": "samara",
            "Омск": "omsk", "Ростов-на-Дону": "rostov-na-donu",
        }
        city_slug = city_map.get(city, "rossiya")
        headers = {
            "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36",
            "Accept-Language": "ru-RU,ru;q=0.9",
        }
        url = f"https://www.avito.ru/{city_slug}?q={query}&s=104"  # s=104 = по дате
        async with aiohttp.ClientSession() as session:
            async with session.get(url, headers=headers, timeout=aiohttp.ClientTimeout(total=10)) as resp:
                if resp.status == 200:
                    text = await resp.text()
                    # Ищем телефоны в открытом HTML
                    phones_raw = re.findall(r'\+7[\s\-]?\(?\d{3}\)?[\s\-]?\d{3}[\s\-]?\d{2}[\s\-]?\d{2}', text)
                    names_raw = re.findall(r'"sellerName":"([^"]+)"', text)

                    seen_phones = set()
                    for i, phone in enumerate(phones_raw[:limit]):
                        clean = re.sub(r'[\s\-\(\)]', '', phone)
                        if clean not in seen_phones:
                            seen_phones.add(clean)
                            results.append({
                                "source": "Авито",
                                "name": names_raw[i] if i < len(names_raw) else "Не указано",
                                "address": city,
                                "phones": clean,
                                "city": city,
                                "category": query
                            })
    except Exception as e:
        logger.error(f"Avito error: {e}")
    return results


async def parse_yandex_maps(city: str, query: str, limit: int = 15) -> list:
    """Парсинг Яндекс Карт (открытые данные)"""
    results = []
    try:
        url = "https://search-maps.yandex.ru/v1/"
        params = {
            "text": f"{query} {city}",
            "type": "biz",
            "lang": "ru_RU",
            "apikey": "dda3ddba-c9ea-4ead-9010-f43321d247cb",  # Публичный ключ
            "results": limit
        }
        async with aiohttp.ClientSession() as session:
            async with session.get(url, params=params, timeout=aiohttp.ClientTimeout(total=10)) as resp:
                if resp.status == 200:
                    data = await resp.json()
                    features = data.get("features", [])
                    for feat in features:
                        props = feat.get("properties", {})
                        name = props.get("name", "")
                        address = props.get("description", "")
                        phones = []
                        company_meta = props.get("CompanyMetaData", {})
                        for phone in company_meta.get("Phones", []):
                            if phone.get("type") == "phone":
                                phones.append(phone.get("formatted", ""))
                        if phones:
                            results.append({
                                "source": "Яндекс Карты",
                                "name": name,
                                "address": address,
                                "phones": ", ".join(phones[:2]),
                                "city": city,
                                "category": query
                            })
    except Exception as e:
        logger.error(f"Yandex Maps error: {e}")
    return results


async def search_all_sources(city: str, query: str) -> list:
    """Поиск по всем источникам параллельно"""
    tasks = [
        parse_2gis(city, query),
        parse_avito(city, query),
        parse_yandex_maps(city, query),
    ]
    all_results = await asyncio.gather(*tasks, return_exceptions=True)
    combined = []
    for r in all_results:
        if isinstance(r, list):
            combined.extend(r)
    # Убираем дубли по номеру
    seen = set()
    unique = []
    for item in combined:
        phone_key = re.sub(r'\D', '', item.get("phones", ""))
        if phone_key and phone_key not in seen:
            seen.add(phone_key)
            unique.append(item)
    return unique


def make_csv(results: list) -> bytes:
    """Генерация CSV файла с результатами"""
    output = io.StringIO()
    writer = csv.DictWriter(output, fieldnames=["source", "name", "address", "phones", "city", "category"])
    writer.writeheader()
    writer.writerows(results)
    return output.getvalue().encode("utf-8-sig")


# ─── КЛАВИАТУРЫ ──────────────────────────────────────────────

def main_keyboard(is_admin=False):
    buttons = [
        [KeyboardButton(text="🔍 Найти контакты бизнесов")],
        [KeyboardButton(text="📊 Статистика"), KeyboardButton(text="❓ Помощь")],
    ]
    if is_admin:
        buttons.append([KeyboardButton(text="👑 Админ панель")])
    return ReplyKeyboardMarkup(keyboard=buttons, resize_keyboard=True)


def city_keyboard():
    buttons = [[KeyboardButton(text=city)] for city in CITIES]
    buttons.append([KeyboardButton(text="🏙 Ввести свой город")])
    buttons.append([KeyboardButton(text="🏠 Главное меню")])
    return ReplyKeyboardMarkup(keyboard=buttons, resize_keyboard=True, row_width=2)


def category_keyboard():
    buttons = [[KeyboardButton(text=cat)] for cat in CATEGORIES.keys()]
    buttons.append([KeyboardButton(text="✏️ Свой запрос")])
    buttons.append([KeyboardButton(text="🏠 Главное меню")])
    return ReplyKeyboardMarkup(keyboard=buttons, resize_keyboard=True)


# ─── ХЕНДЛЕРЫ ────────────────────────────────────────────────

@dp.message(Command("start"))
async def cmd_start(message: types.Message, state: FSMContext):
    await state.clear()
    user = message.from_user
    is_admin = user.username == ADMIN_USERNAME
    if is_admin and user.id not in ADMIN_IDS:
        ADMIN_IDS.append(user.id)

    greeting = "👑 Добро пожаловать, владелец!" if is_admin else "Привет!"

    await message.answer(
        f"{greeting}\n\n"
        f"🤖 <b>TradeChinaMarket Lead Bot</b>\n\n"
        f"Я нахожу контакты бизнесов из открытых источников:\n"
        f"• 📍 2GIS\n"
        f"• 🏠 Авито\n"
        f"• 🗺 Яндекс Карты\n\n"
        f"Данные берутся только из <b>публично размещённых</b> источников.\n\n"
        f"Выбери действие 👇",
        parse_mode="HTML",
        reply_markup=main_keyboard(is_admin)
    )


@dp.message(F.text == "🔍 Найти контакты бизнесов")
async def start_search(message: types.Message, state: FSMContext):
    await state.set_state(SearchState.waiting_city)
    await message.answer(
        "📍 <b>Шаг 1 из 3 — Выберите город:</b>",
        parse_mode="HTML",
        reply_markup=city_keyboard()
    )


@dp.message(SearchState.waiting_city)
async def got_city(message: types.Message, state: FSMContext):
    if message.text == "🏠 Главное меню":
        await state.clear()
        await message.answer("Главное меню", reply_markup=main_keyboard(message.from_user.username == ADMIN_USERNAME))
        return
    if message.text == "🏙 Ввести свой город":
        await message.answer("Введите название города:")
        return

    await state.update_data(city=message.text)
    await state.set_state(SearchState.waiting_category)
    await message.answer(
        f"✅ Город: <b>{message.text}</b>\n\n"
        f"🏷 <b>Шаг 2 из 3 — Выберите категорию бизнеса:</b>",
        parse_mode="HTML",
        reply_markup=category_keyboard()
    )


@dp.message(SearchState.waiting_category)
async def got_category(message: types.Message, state: FSMContext):
    if message.text == "🏠 Главное меню":
        await state.clear()
        await message.answer("Главное меню", reply_markup=main_keyboard(message.from_user.username == ADMIN_USERNAME))
        return

    data = await state.get_data()
    city = data.get("city", "Москва")

    if message.text == "✏️ Свой запрос":
        await state.set_state(SearchState.waiting_query)
        await message.answer("✏️ Введите свой поисковый запрос (например: 'грузоперевозки оптом'):")
        return

    # Определяем поисковый запрос по категории
    query = CATEGORIES.get(message.text, message.text)
    await state.update_data(category=message.text, query=query)

    await do_search(message, state, city, query, message.text)


@dp.message(SearchState.waiting_query)
async def got_custom_query(message: types.Message, state: FSMContext):
    data = await state.get_data()
    city = data.get("city", "Москва")
    query = message.text
    await state.update_data(category=query, query=query)
    await do_search(message, state, city, query, query)


async def do_search(message: types.Message, state: FSMContext, city: str, query: str, category: str):
    """Выполняет поиск и отправляет результаты"""
    is_admin = message.from_user.username == ADMIN_USERNAME

    loading = await message.answer(
        f"🔍 Ищу контакты...\n\n"
        f"📍 Город: <b>{city}</b>\n"
        f"🏷 Категория: <b>{category}</b>\n\n"
        f"⏳ Обрабатываю 3 источника параллельно...",
        parse_mode="HTML"
    )

    results = await search_all_sources(city, query)
    await state.clear()

    if not results:
        await loading.edit_text(
            "😔 По вашему запросу ничего не найдено.\n\n"
            "Попробуйте:\n"
            "• Другой город\n"
            "• Другую категорию\n"
            "• Более общий запрос",
            reply_markup=main_keyboard(is_admin)
        )
        return

    # Показываем первые 5 в чате
    preview_text = (
        f"✅ <b>Найдено: {len(results)} контактов</b>\n"
        f"📍 {city} · 🏷 {category}\n"
        f"━━━━━━━━━━━━━━━━━━━━\n\n"
    )

    for i, r in enumerate(results[:5], 1):
        preview_text += (
            f"<b>{i}. {r['name']}</b>\n"
            f"📞 {r['phones']}\n"
            f"📍 {r['address']}\n"
            f"🔗 Источник: {r['source']}\n\n"
        )

    if len(results) > 5:
        preview_text += f"<i>...и ещё {len(results) - 5} контактов в файле ниже 👇</i>"

    await loading.edit_text(preview_text, parse_mode="HTML")

    # Отправляем CSV файл
    csv_data = make_csv(results)
    filename = f"TCM_leads_{city}_{datetime.now().strftime('%d%m%Y_%H%M')}.csv"
    await message.answer_document(
        BufferedInputFile(csv_data, filename=filename),
        caption=(
            f"📊 <b>Полная база: {len(results)} контактов</b>\n"
            f"Открой в Excel или Google Sheets\n\n"
            f"💡 Для холодного обзвона используй скрипт:\n"
            f"<i>«Добрый день, мы TradeChinaMarket — доставляем товары из Китая под ключ. "
            f"Работаем с WB и Ozon селлерами. Могу рассчитать стоимость для вас?»</i>"
        ),
        parse_mode="HTML",
        reply_markup=main_keyboard(is_admin)
    )


@dp.message(F.text == "📊 Статистика")
async def show_stats(message: types.Message):
    await message.answer(
        "📊 <b>Статистика бота</b>\n\n"
        "• Источники: 2GIS, Авито, Яндекс Карты\n"
        "• Городов в базе: 15+\n"
        "• Категорий: 10\n"
        "• Данные: только публичные\n\n"
        "🚀 Используй результаты для:\n"
        "• Холодного обзвона\n"
        "• Рассылки в WhatsApp\n"
        "• Таргетированного предложения",
        parse_mode="HTML"
    )


@dp.message(F.text == "❓ Помощь")
async def show_help(message: types.Message):
    await message.answer(
        "❓ <b>Как пользоваться ботом</b>\n\n"
        "1️⃣ Нажми «Найти контакты бизнесов»\n"
        "2️⃣ Выбери город\n"
        "3️⃣ Выбери категорию бизнеса\n"
        "4️⃣ Получи список контактов + CSV файл\n\n"
        "📌 <b>Все данные из открытых источников:</b>\n"
        "• 2GIS — бизнес-справочник\n"
        "• Авито — публичные объявления\n"
        "• Яндекс Карты — бизнес-карточки\n\n"
        "📞 По вопросам: @sbc_market",
        parse_mode="HTML"
    )


@dp.message(F.text == "👑 Админ панель")
async def admin_panel(message: types.Message):
    if message.from_user.username != ADMIN_USERNAME:
        return

    keyboard = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="📨 Рассылка всем", callback_data="admin_broadcast")],
        [InlineKeyboardButton(text="📈 Пользователи", callback_data="admin_users")],
    ])

    await message.answer(
        "👑 <b>Админ панель</b>\n\n"
        "Доступные функции:",
        parse_mode="HTML",
        reply_markup=keyboard
    )


@dp.callback_query(F.data == "admin_broadcast")
async def admin_broadcast(callback: types.CallbackQuery):
    if callback.from_user.username != ADMIN_USERNAME:
        return
    await callback.message.answer("📨 Введите текст для рассылки (функция в разработке):")
    await callback.answer()


@dp.message(F.text == "🏠 Главное меню")
async def go_home(message: types.Message, state: FSMContext):
    await state.clear()
    is_admin = message.from_user.username == ADMIN_USERNAME
    await message.answer("Главное меню 👇", reply_markup=main_keyboard(is_admin))


# ─── ЗАПУСК ──────────────────────────────────────────────────

async def main():
    logger.info("Бот @Tcmmarket_bot запускается...")
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
