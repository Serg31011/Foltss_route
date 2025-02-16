from telegram import InlineKeyboardButton, InlineKeyboardMarkup, Update, ReplyKeyboardMarkup
from telegram.ext import Application, CommandHandler, MessageHandler, filters, ContextTypes, CallbackQueryHandler
import googlemaps

# Ваш токен Telegram-бота та Google API ключ
TELEGRAM_API_TOKEN = '7215765624:AAGMBF8vJ5pOFYc5Hu6fr4SzA61uOAB4210'
GOOGLE_API_KEY = 'AIzaSyCJu9Hx3Nz9OX2nOR2O-Uk8DxTKJoAghJ0'

# Ініціалізація клієнта Google Maps
gmaps = googlemaps.Client(key=GOOGLE_API_KEY)

# Змінні для збереження стану користувача
user_state = {}          # зберігає поточний стан (waiting_for_vehicle, waiting_for_cities, waiting_for_transport_type тощо)
user_routes = {}         # зберігає інформацію про маршрут: посилання та текст
user_vehicle = {}        # зберігає вибір автомобіля
user_transport_type = {} # зберігає вибір типу перевезення

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    # Створення меню бота без кнопок в чаті
    keyboard = [
        ["Створити маршрут"],  # це буде кнопка в основному меню
    ]
    reply_markup = ReplyKeyboardMarkup(keyboard, resize_keyboard=True)
    await update.message.reply_text("Меню бота:", reply_markup=reply_markup)
    user_state[update.effective_user.id] = "waiting_for_action"

# Обробка натискання кнопки "Створити маршрут" в меню
async def handle_create_route(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    # Вибір автомобіля
    keyboard = [
        [InlineKeyboardButton("УАК 6717-Е6", callback_data='UAK6717')],
    ]
    reply_markup = InlineKeyboardMarkup(keyboard)
    await update.message.reply_text("Виберіть автомобіль для перевезення:", reply_markup=reply_markup)
    user_state[user_id] = "waiting_for_vehicle"

# Обробка вибору автомобіля
async def handle_vehicle_selection(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    query = update.callback_query
    await query.answer()
    selected_vehicle = query.data  # очікуємо 'UAK6717'

    # Визначення повної назви автомобіля
    vehicle_name = "УАК 6717-Е6" if selected_vehicle == "UAK6717" else selected_vehicle
    user_vehicle[user_id] = vehicle_name

    await query.edit_message_text("Тепер введіть маршрут через ' - ' (наприклад: Савинці - Ізюм - Чугуїв).")
    user_state[user_id] = "waiting_for_cities"

# Обробка введення маршруту користувачем
async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    message_text = update.message.text

    if user_id in user_state and user_state[user_id] == "waiting_for_cities":
        try:
            if ' - ' in message_text:
                locations = message_text.split(' - ')
                if len(locations) < 2:
                    await update.message.reply_text("Будь ласка, введіть хоча б два міста через ' - '.")
                    return

                # Формування посилання на маршрут за допомогою Google Maps
                route_url = f"https://www.google.com/maps/dir/{'/'.join(locations)}"
                # Формування тексту маршруту (без посилання)
                route_text = ' - '.join(locations)
                user_routes[user_id] = {"url": route_url, "text": route_text}

                # Перший крок: надсилання повідомлення з посиланням на маршрут
                await update.message.reply_text(f"Ось ваш маршрут: {route_url}")

                # Створення меню для вибору типу перевезення
                keyboard = [
                    [InlineKeyboardButton("Перевезення дров", callback_data='firewood')],
                    [InlineKeyboardButton("Перевезення лісу", callback_data='wood')],
                ]
                reply_markup = InlineKeyboardMarkup(keyboard)
                await update.message.reply_text("Виберіть тип перевезення:", reply_markup=reply_markup)
                user_state[user_id] = "waiting_for_transport_type"
            else:
                await update.message.reply_text("Будь ласка, введіть маршрут у форматі: місто1 - місто2 - місто3.")
        except Exception as e:
            await update.message.reply_text(f"Сталася помилка: {str(e)}")

# Обробка вибору типу перевезення
async def handle_transport_type(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    query = update.callback_query
    await query.answer()
    selected_transport = query.data  # 'firewood' або 'wood'
    user_transport_type[user_id] = selected_transport

    # Формування звіту: перше повідомлення містило посилання, а звіт (друге повідомлення) містить тільки текст маршруту
    vehicle = user_vehicle.get(user_id, "УАК 6717-Е6")  # Якщо автомобіль не вибраний, поставимо значення за замовчуванням
    route_text = user_routes.get(user_id, {}).get("text", "Невідомий маршрут")
    transport_str = "Перевезення дров" if selected_transport == 'firewood' else "Перевезення лісу"

    # Формування другого повідомлення
    report = f"Виїзд {vehicle}\n{route_text}\n{transport_str}"
    await query.edit_message_text(text=report)
    user_state[user_id] = "report_created"

def main():
    application = Application.builder().token(TELEGRAM_API_TOKEN).build()
    application.add_handler(CommandHandler("start", start))
    application.add_handler(MessageHandler(filters.Regex('^Створити маршрут$'), handle_create_route))
    application.add_handler(CallbackQueryHandler(handle_vehicle_selection, pattern="^UAK6717$"))
    application.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))
    application.add_handler(CallbackQueryHandler(handle_transport_type, pattern="^(firewood|wood)$"))
    application.run_polling()

if __name__ == "__main__":
    main()
