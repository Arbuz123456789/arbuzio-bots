```python
import logging
import sqlite3
from telegram import (
    Update, InlineKeyboardButton, InlineKeyboardMarkup,
    ReplyKeyboardMarkup, ReplyKeyboardRemove, KeyboardButton
)
from telegram.ext import (
    Application, CommandHandler, CallbackQueryHandler,
    MessageHandler, ConversationHandler, ContextTypes, filters
)

# НАСТРОЙКИ
BOT_TOKEN = "8665186624:AAHR-Iw8UbKswmuKmZW0kC-c_rPotdQ0Hf8"      #токен
ADMIN_ID = 8744652410                              # ID админа
DELIVERY_PRICE = 400                       # фикс стоимость доставки
PAYMENT_BASE_URL = "https://www.youtube.com/watch?v=dQw4w9WgXcQ"  # ссылка на оплату
CONTACT_USERNAME = "@arbuzio88"        # ник для связи

logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    level=logging.INFO
)


# КЛАВИАТУРА
def main_keyboard(user_id):
    if user_id == ADMIN_ID:
        keyboard = [
            [KeyboardButton("Каталог"), KeyboardButton("Добавить товар")],
            [KeyboardButton("Изменить товар"), KeyboardButton("Удалить товар")],
            [KeyboardButton("Список товаров"), KeyboardButton("Клиенты")],
        ]
    else:
        keyboard = [
            [KeyboardButton("Каталог")],
            [KeyboardButton("Связаться с нами")],
        ]
    return ReplyKeyboardMarkup(keyboard, resize_keyboard=True)


# БАЗА ДАННЫХ 
def init_db():
    conn = sqlite3.connect("shop.db")
    cur = conn.cursor()
    # Товары
    cur.execute("""
        CREATE TABLE IF NOT EXISTS products (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT,
            description TEXT,
            price INTEGER,
            sizes TEXT,
            photo TEXT
        )
    """)
    # Заказы
    cur.execute("""
        CREATE TABLE IF NOT EXISTS orders (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id INTEGER,
            product_name TEXT,
            size TEXT,
            fio TEXT,
            phone TEXT,
            address TEXT,
            total_price INTEGER,
            status TEXT,
            track TEXT
        )
    """)
    conn.commit()
    conn.close()


def db_query(query, params=(), fetch=False, one=False):
    conn = sqlite3.connect("shop.db")
    cur = conn.cursor()
    cur.execute(query, params)
    result = None
    if fetch:
        result = cur.fetchone() if one else cur.fetchall()
    conn.commit()
    lastid = cur.lastrowid
    conn.close()
    return result if fetch else lastid


# СОСТОЯНИЯ ДИАЛОГА 
(CHOOSING_SIZE, GET_FIO, GET_PHONE, GET_ADDRESS,
 GET_SCREENSHOT) = range(5)

# Состояния для админа
(ADD_NAME, ADD_DESC, ADD_PRICE, ADD_SIZES, ADD_PHOTO,
 EDIT_CHOOSE, EDIT_FIELD, EDIT_VALUE) = range(10, 18)


# КОМАНДА /start 
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    if user_id == ADMIN_ID:
        text = "Режим администратора"
    else:
        text = "Добро пожаловать!"
    await update.message.reply_text(text, reply_markup=main_keyboard(user_id))
    await show_catalog(update, context)


async def show_catalog(update: Update, context: ContextTypes.DEFAULT_TYPE):
    products = db_query("SELECT id, name, price FROM products", fetch=True)
    if not products:
        text = "Каталог пока пуст."
        if update.message:
            await update.message.reply_text(text)
        else:
            await update.callback_query.message.reply_text(text)
        return

    keyboard = [
        [InlineKeyboardButton(f"{name} — {price}₽",
                              callback_data=f"item_{pid}")]
        for pid, name, price in products
    ]
    markup = InlineKeyboardMarkup(keyboard)
    text = " *Каталог*"

    if update.message:
        await update.message.reply_text(text, reply_markup=markup, parse_mode="Markdown")
    else:
        await update.callback_query.message.reply_text(text, reply_markup=markup, parse_mode="Markdown")


# ВЫБОР ТОВАРА 
async def choose_item(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    pid = int(query.data.split("_")[1])

    product = db_query(
        "SELECT id, name, description, price, sizes, photo FROM products WHERE id=?",
        (pid,), fetch=True, one=True
    )
    if not product:
        await query.message.reply_text("Товар не найден.")
        return ConversationHandler.END

    pid, name, desc, price, sizes, photo = product
    context.user_data['product_id'] = pid
    context.user_data['product_name'] = name
    context.user_data['product_price'] = price

    sizes_list = [s.strip() for s in sizes.split(",") if s.strip()]
    keyboard = [
        [InlineKeyboardButton(s, callback_data=f"size_{s}")]
        for s in sizes_list
    ]
    keyboard.append([InlineKeyboardButton("⬅️ Назад", callback_data="back_catalog")])
    markup = InlineKeyboardMarkup(keyboard)

    text = (
        f"*{name}*\n\n"
        f"{desc}\n\n"
        f"Цена: {price}₽\n\n"
        f"*Размерная сетка:*\n{sizes}\n\n"
        f"Выберите размер:"
    )

    if photo:
        await query.message.reply_photo(
            photo=photo, caption=text, reply_markup=markup, parse_mode="Markdown"
        )
    else:
        await query.message.reply_text(text, reply_markup=markup, parse_mode="Markdown")
    return CHOOSING_SIZE


async def back_catalog(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    await show_catalog(update, context)
    return ConversationHandler.END


# ВЫБОР РАЗМЕРА
async def choose_size(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    size = query.data.split("_", 1)[1]
    context.user_data['size'] = size

    await query.message.reply_text(
        f"Вы выбрали размер: *{size}*\n\n"
        "Введите ваше ФИО:",
        parse_mode="Markdown"
    )
    return GET_FIO


async def get_fio(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data['fio'] = update.message.text
    await update.message.reply_text("Введите ваш номер телефона:", parse_mode="Markdown")
    return GET_PHONE


async def get_phone(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data['phone'] = update.message.text
    await update.message.reply_text("Введите адрес пункта выдачи:", parse_mode="Markdown")
    return GET_ADDRESS


async def get_address(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data['address'] = update.message.text

    price = context.user_data['product_price']
    total = price + DELIVERY_PRICE
    context.user_data['total'] = total

    pay_url = f"{PAYMENT_BASE_URL}{total}"

    text = (
        f"*Ваш заказ:*\n"
        f"Вещь: {context.user_data['product_name']}\n"
        f"Размер: {context.user_data['size']}\n"
        f"Цена товара: {price}₽\n"
        f"Доставка: {DELIVERY_PRICE}₽\n"
        f"*Итого: {total}₽*\n\n"
        f"Ссылка на оплату:\n{pay_url}\n\n"
        f"После оплаты пришлите *скриншот чека* в этот чат."
    )
    await update.message.reply_text(text, parse_mode="Markdown")
    return GET_SCREENSHOT


# СКРИНШОТ ОПЛАТЫ
async def get_screenshot(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not update.message.photo:
        await update.message.reply_text("Пожалуйста, пришлите *скриншот* (фото) чека.", parse_mode="Markdown")
        return GET_SCREENSHOT

    user = update.effective_user
    photo_id = update.message.photo[-1].file_id

    order_id = db_query(
        """INSERT INTO orders
        (user_id, product_name, size, fio, phone, address, total_price, status, track)
        VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)""",
        (
            user.id,
            context.user_data['product_name'],
            context.user_data['size'],
            context.user_data['fio'],
            context.user_data['phone'],
            context.user_data['address'],
            context.user_data['total'],
            "Проверка оплаты",
            ""
        )
    )

    # Сообщение клиенту
    await update.message.reply_text(
        f"Спасибо! Заказ принят.\n\n"
        f"Номер заказа: #{order_id}\n\n"
        f"Оплата проверяется. Ожидайте подтверждения.",
        reply_markup=main_keyboard(user.id)
    )

    admin_text = (
        f"НОВЫЙ ЗАКАЗ #{order_id}\n\n"
        f"Клиент ID: {user.id}\n"
        f"ФИО: {context.user_data['fio']}\n"
        f"Телефон: {context.user_data['phone']}\n"
        f"Адрес СДЭК: {context.user_data['address']}\n\n"
        f"Вещь: {context.user_data['product_name']}\n"
        f"Размер: {context.user_data['size']}\n"
        f"Сумма: {context.user_data['total']}₽\n\n"
        f"Проверьте чек"
    )

    # Кнопки для админа: подтвердить или отклонить
    admin_keyboard = InlineKeyboardMarkup([
        [
            InlineKeyboardButton("Подтвердить", callback_data=f"pay_ok_{order_id}"),
            InlineKeyboardButton("Чек фальшивый", callback_data=f"pay_bad_{order_id}")
        ]
    ])

    try:
        await context.bot.send_photo(
            chat_id=ADMIN_ID,
            photo=photo_id,
            caption=admin_text,
            reply_markup=admin_keyboard
        )
    except Exception as e:
        logging.error(f"Не удалось отправить админу: {e}")
        await context.bot.send_message(chat_id=ADMIN_ID, text=admin_text, reply_markup=admin_keyboard)
        await context.bot.send_photo(chat_id=ADMIN_ID, photo=photo_id)

    context.user_data.clear()
    return ConversationHandler.END

# АДМИН: подтверждение / отклонение оплаты
async def payment_decision(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()

    if update.effective_user.id != ADMIN_ID:
        return

    data = query.data  # pay_ok_5 или pay_bad_5
    parts = data.split("_")
    decision = parts[1]        # ok или bad
    order_id = int(parts[2])

    order = db_query("SELECT user_id, product_name FROM orders WHERE id=?",
                     (order_id,), fetch=True, one=True)
    if not order:
        await query.edit_message_caption(caption="Заказ не найден.")
        return

    user_id, product_name = order

    if decision == "ok":
        # Оплата подтверждена
        db_query("UPDATE orders SET status=? WHERE id=?",
                 ("Оплачен, ждёт отправки", order_id))
        await context.bot.send_message(
            chat_id=user_id,
            text=(
                f"Ваша оплата по заказу #{order_id} подтверждена!\n\n"
                f"Ожидайте доставку. Как только товар будет отправлен, "
                f"Мы пришлём вам трек-номер."
            )
        )
        # Обновляем сообщение у админа
        try:
            await query.edit_message_caption(
                caption=f"Оплата ПОДТВЕРЖДЕНА (заказ #{order_id}).\n\n"
                        f"Чтобы отправить трек:\n/track {order_id} ТРЕКНОМЕР"
            )
        except Exception:
            await query.message.reply_text(
                f"Оплата подтверждена (заказ #{order_id}).\n"
                f"Трек: /track {order_id} ТРЕКНОМЕР"
            )

    else:
        # Чек фальшивый
        db_query("UPDATE orders SET status=? WHERE id=?",
                 ("Отклонён (фальшивый чек)", order_id))
        await context.bot.send_message(
            chat_id=user_id,
            text=(
                f"К сожалению, ваш чек по заказу #{order_id} не прошёл проверку.\n\n"
                f"Оплата не подтверждена. Пожалуйста, свяжитесь с нами "
                f"или оформите заказ заново."
            )
        )
        try:
            await query.edit_message_caption(
                caption=f"Чек ОТКЛОНЁН как фальшивый (заказ #{order_id})."
            )
        except Exception:
            await query.message.reply_text(f"Чек отклонён (заказ #{order_id}).")

# АДМИН: отправка трек-номера 
async def send_track(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != ADMIN_ID:
        return
    try:
        order_id = int(context.args[0])
        track = " ".join(context.args[1:])
        if not track:
            raise ValueError
    except (IndexError, ValueError):
        await update.message.reply_text("Формат: /track <номер_заказа> <трек-номер>")
        return

    order = db_query("SELECT user_id, product_name FROM orders WHERE id=?",
                     (order_id,), fetch=True, one=True)
    if not order:
        await update.message.reply_text("Заказ не найден.")
        return

    user_id, product_name = order
    db_query("UPDATE orders SET track=?, status=? WHERE id=?",
             (track, "Отправлен", order_id))

    await context.bot.send_message(
        chat_id=user_id,
        text=(
            f"*Ваш заказ #{order_id} отправлен!*\n\n"
            f"Вещь: {product_name}\n"
            f"Трек-номер: `{track}`\n\n"
        ),
        parse_mode="Markdown"
    )
    await update.message.reply_text(f"Трек-номер отправлен клиенту (заказ #{order_id}).")


# АДМИН: база клиентов 
async def clients_list(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != ADMIN_ID:
        return
    orders = db_query(
        "SELECT id, user_id, phone, product_name, status FROM orders ORDER BY id DESC",
        fetch=True
    )
    if not orders:
        await update.message.reply_text("Заказов пока нет.")
        return

    text = "*База клиентов:*\n\n"
    for oid, uid, phone, product, status in orders:
        text += f"#{oid} | ID:{uid} | {phone} ({product}) — {status}\n"
    await update.message.reply_text(text, parse_mode="Markdown")


# АДМИН: панель и товары 
async def admin_panel(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != ADMIN_ID:
        return
    text = (
        "*Админ-панель*\n\n"
        "Используйте кнопки внизу или команды:\n"
        "/add — добавить товар\n"
        "/edit — изменить товар\n"
        "/delete <id> — удалить товар\n"
        "/list — список товаров\n"
        "/clients — база клиентов\n"
        "/track <id> <трек> — отправить трек"
    )
    await update.message.reply_text(text, parse_mode="Markdown",
                                    reply_markup=main_keyboard(update.effective_user.id))


async def list_products(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != ADMIN_ID:
        return
    products = db_query("SELECT id, name, price, sizes FROM products", fetch=True)
    if not products:
        await update.message.reply_text("Товаров нет.")
        return
    text = "*Товары:*\n\n"
    for pid, name, price, sizes in products:
        text += f"ID {pid}: {name} — {price}₽ (размеры: {sizes})\n"
    await update.message.reply_text(text, parse_mode="Markdown")


async def delete_product(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != ADMIN_ID:
        return
    try:
        pid = int(context.args[0])
    except (IndexError, ValueError):
        await update.message.reply_text("Формат: /delete <id товара>")
        return
    db_query("DELETE FROM products WHERE id=?", (pid,))
    await update.message.reply_text(f"Товар #{pid} удалён.")


# Добавление товара
async def add_start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != ADMIN_ID:
        return ConversationHandler.END
    await update.message.reply_text("*Название* товара:", parse_mode="Markdown")
    return ADD_NAME


async def add_name(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data['new_name'] = update.message.text
    await update.message.reply_text("*Описание* товара:", parse_mode="Markdown")
    return ADD_DESC


async def add_desc(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data['new_desc'] = update.message.text
    await update.message.reply_text("*Цена*:", parse_mode="Markdown")
    return ADD_PRICE


async def add_price(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        price = int(update.message.text)
    except ValueError:
        await update.message.reply_text("Ты долбан, онли число:")
        return ADD_PRICE
    context.user_data['new_price'] = price
    await update.message.reply_text(
        "*Размерная сетка* через запятую.\n"
        "Например: `S (44-46), M (48-50), L (52-54)`",
        parse_mode="Markdown"
    )
    return ADD_SIZES


async def add_sizes(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data['new_sizes'] = update.message.text
    await update.message.reply_text(
        "*Фото товара*.\n\n"
        "Без фото - напиши `нет`",
        parse_mode="Markdown"
    )
    return ADD_PHOTO


async def add_photo(update: Update, context: ContextTypes.DEFAULT_TYPE):
    photo_id = None
    # Если прислали фото
    if update.message.photo:
        photo_id = update.message.photo[-1].file_id
    # Если написали "нет" — оставляем без фото
    elif update.message.text and update.message.text.lower() == "нет":
        photo_id = None
    else:
        await update.message.reply_text("Пришли фото или напиши `нет`.", parse_mode="Markdown")
        return ADD_PHOTO

    db_query(
        "INSERT INTO products (name, description, price, sizes, photo) VALUES (?, ?, ?, ?, ?)",
        (
            context.user_data['new_name'],
            context.user_data['new_desc'],
            context.user_data['new_price'],
            context.user_data['new_sizes'],
            photo_id
        )
    )
    await update.message.reply_text(
        "Товар добавлен!",
        reply_markup=main_keyboard(update.effective_user.id)
    )
    context.user_data.clear()
    return ConversationHandler.END


# Изменение товара
async def edit_start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != ADMIN_ID:
        return ConversationHandler.END
    products = db_query("SELECT id, name FROM products", fetch=True)
    if not products:
        await update.message.reply_text("Товаров нет.")
        return ConversationHandler.END
    text = "*ID товара*, который изменить:\n\n"
    for pid, name in products:
        text += f"ID {pid}: {name}\n"
    await update.message.reply_text(text, parse_mode="Markdown")
    return EDIT_CHOOSE


async def edit_choose(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        pid = int(update.message.text)
    except ValueError:
        await update.message.reply_text("Введите числовой ID:")
        return EDIT_CHOOSE

    product = db_query("SELECT id FROM products WHERE id=?", (pid,), fetch=True, one=True)
    if not product:
        await update.message.reply_text("Товар не найден. Ввод ID:")
        return EDIT_CHOOSE

    context.user_data['edit_id'] = pid
    keyboard = [
        [InlineKeyboardButton("Название", callback_data="ef_name")],
        [InlineKeyboardButton("Описание", callback_data="ef_description")],
        [InlineKeyboardButton("Цену", callback_data="ef_price")],
        [InlineKeyboardButton("Размерную сетку", callback_data="ef_sizes")],
        [InlineKeyboardButton("Фото", callback_data="ef_photo")],
    ]
    await update.message.reply_text(
        "Что изменить?",
        reply_markup=InlineKeyboardMarkup(keyboard)
    )
    return EDIT_FIELD


async def edit_field(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    field = query.data.split("_", 1)[1]
    context.user_data['edit_field'] = field

    field_names = {
        "name": "название",
        "description": "описание",
        "price": "цену (число)",
        "sizes": "размерную сетку",
        "photo": "фото (пришлите новое фото)"
    }
    await query.message.reply_text(f"Введи новое значение — {field_names[field]}:")
    return EDIT_VALUE


async def edit_value(update: Update, context: ContextTypes.DEFAULT_TYPE):
    field = context.user_data['edit_field']
    pid = context.user_data['edit_id']

    # Если меняем фото
    if field == "photo":
        if not update.message.photo:
            await update.message.reply_text("Пришли фото:")
            return EDIT_VALUE
        value = update.message.photo[-1].file_id
    else:
        value = update.message.text
        if field == "price":
            try:
                value = int(value)
            except ValueError:
                await update.message.reply_text("Цена должна быть числом долбан:")
                return EDIT_VALUE

    db_query(f"UPDATE products SET {field}=? WHERE id=?", (value, pid))
    await update.message.reply_text(
        "Товар обновлён!",
        reply_markup=main_keyboard(update.effective_user.id)
    )
    context.user_data.clear()
    return ConversationHandler.END


# ОТМЕНА 
async def cancel(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "Действие отменено.",
        reply_markup=main_keyboard(update.effective_user.id)
    )
    context.user_data.clear()
    return ConversationHandler.END


# ОБРАБОТКА КНОПОК МЕНЮ
async def menu_buttons(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text
    user_id = update.effective_user.id

    if text == "Каталог":
        await show_catalog(update, context)
    elif text == "Связаться с нами":
        await update.message.reply_text(f"По всем вопросам пишите: {CONTACT_USERNAME}")
    elif user_id == ADMIN_ID:
        if text == "Список товаров":
            await list_products(update, context)
        elif text == "Клиенты":
            await clients_list(update, context)
        elif text == "Удалить товар":
            await update.message.reply_text(
                "Чтобы удалить товар, напиши:\n`/delete <номер товара>`\n\n"
                "Посмотреть номера — кнопка «Список товаров»",
                parse_mode="Markdown"
            )


# ЗАПУСК
def main():
    init_db()
    app = (
        Application.builder()
        .token(BOT_TOKEN)
        .connect_timeout(30)
        .read_timeout(30)
        .build()
    )

    # Диалог заказа (клиент)
    order_conv = ConversationHandler(
        entry_points=[CallbackQueryHandler(choose_item, pattern=r"^item_\d+$")],
        states={
            CHOOSING_SIZE: [
                CallbackQueryHandler(choose_size, pattern=r"^size_"),
                CallbackQueryHandler(back_catalog, pattern=r"^back_catalog$"),
            ],
            GET_FIO: [MessageHandler(filters.TEXT & ~filters.COMMAND, get_fio)],
            GET_PHONE: [MessageHandler(filters.TEXT & ~filters.COMMAND, get_phone)],
            GET_ADDRESS: [MessageHandler(filters.TEXT & ~filters.COMMAND, get_address)],
            GET_SCREENSHOT: [MessageHandler(filters.PHOTO, get_screenshot)],
        },
        fallbacks=[CommandHandler("cancel", cancel)],
    )

    # Диалог добавления товара 
    add_conv = ConversationHandler(
        entry_points=[
            CommandHandler("add", add_start),
            MessageHandler(filters.Regex("^Добавить товар$"), add_start),
        ],
        states={
            ADD_NAME: [MessageHandler(filters.TEXT & ~filters.COMMAND, add_name)],
            ADD_DESC: [MessageHandler(filters.TEXT & ~filters.COMMAND, add_desc)],
            ADD_PRICE: [MessageHandler(filters.TEXT & ~filters.COMMAND, add_price)],
            ADD_SIZES: [MessageHandler(filters.TEXT & ~filters.COMMAND, add_sizes)],
            ADD_PHOTO: [MessageHandler(filters.PHOTO | (filters.TEXT & ~filters.COMMAND), add_photo)],
        },
        fallbacks=[CommandHandler("cancel", cancel)],
    )

    # Диалог изменения товара (админ) 
    edit_conv = ConversationHandler(
        entry_points=[
            CommandHandler("edit", edit_start),
            MessageHandler(filters.Regex("^Изменить товар$"), edit_start),
        ],
        states={
            EDIT_CHOOSE: [MessageHandler(filters.TEXT & ~filters.COMMAND, edit_choose)],
            EDIT_FIELD: [CallbackQueryHandler(edit_field, pattern=r"^ef_")],
            EDIT_VALUE: [MessageHandler(filters.PHOTO | (filters.TEXT & ~filters.COMMAND), edit_value)],
        },
        fallbacks=[CommandHandler("cancel", cancel)],
    )

    # Регистрация обработчиков
    app.add_handler(CommandHandler("start", start))
    app.add_handler(order_conv)
    app.add_handler(add_conv)
    app.add_handler(edit_conv)

    app.add_handler(CommandHandler("admin", admin_panel))
    app.add_handler(CommandHandler("list", list_products))
    app.add_handler(CommandHandler("delete", delete_product))
    app.add_handler(CommandHandler("clients", clients_list))
    app.add_handler(CommandHandler("track", send_track))
    app.add_handler(CallbackQueryHandler(payment_decision, pattern=r"^pay_(ok|bad)_\d+$"))

    # Обработчик кнопок меню
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, menu_buttons))

    print("Бот запущен...")
    app.run_polling()


if __name__ == "__main__":
    main()
```
