import os
import time
import requests
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import Updater, CommandHandler, CallbackContext, CallbackQueryHandler

TELEGRAM_TOKEN = "7854905130:AAFoZQicRC9RCSAERM1YNSejB1RM8LnyHE0"
CURRENCY_API_URL = "https://api.exchangerate-api.com/v4/latest/USD"
CRYPTO_API_URL = "https://api.coingecko.com/api/v3/coins/markets?vs_currency=usd"

users = {}
previous_rates = {}

# دریافت نرخ ارزها
def get_currency_rates():
    try:
        response = requests.get(CURRENCY_API_URL)
        data = response.json()
        usd_to_toman = data["rates"].get("IRR", 1)
        rates = {
            "USD": usd_to_toman,
            "EUR": data["rates"].get("EUR", "N/A") * usd_to_toman,
            "TRY": data["rates"].get("TRY", "N/A") * usd_to_toman,
            "AED": data["rates"].get("AED", "N/A") * usd_to_toman,
            "THB": data["rates"].get("THB", "N/A") * usd_to_toman,
            "IQD": data["rates"].get("IQD", "N/A") * usd_to_toman,
            "GEL": data["rates"].get("GEL", "N/A") * usd_to_toman,
            "AMD": data["rates"].get("AMD", "N/A") * usd_to_toman,
            "RUB": data["rates"].get("RUB", "N/A") * usd_to_toman,
        }
        return rates
    except Exception as e:
        return {"error": str(e)}

# دریافت نرخ تمامی ارزهای دیجیتال
def get_crypto_rates():
    try:
        response = requests.get(CRYPTO_API_URL)
        data = response.json()
        usd_to_toman = get_currency_rates().get("USD", 1)
        crypto_rates = {}
        for coin in data:
            name = coin["name"]
            price = round(coin["current_price"] * usd_to_toman, 0)
            crypto_rates[name] = price
        return crypto_rates
    except Exception as e:
        return {"error": str(e)}

# مقایسه نرخ‌ها برای نمایش رشد
def compare_rates(new_rates, category):
    global previous_rates
    messages = {}
    for name, price in new_rates.items():
        previous_price = previous_rates.get(category, {}).get(name, price)
        change_indicator = "🟢" if price > previous_price else "🔴"
        messages[name] = f"{change_indicator} {name}: {format_price(price)} تومان"
        previous_rates[category] = new_rates
    return messages

# فرمت قیمت با نقطه‌گذاری
def format_price(price):
    return f"{price:,}".replace(",", ".")

# پیام خوش‌آمدگویی و انتخاب ارزها
def start(update: Update, context: CallbackContext):
    chat_id = update.message.chat_id
    users[chat_id] = []
    welcome_message = """
سلام خیلی خوش آمدید 💁
اینجا میتونید قیمت‌های ارز دیجیتال و واحدهای پولی رو به تومان ببینید بدون نیاز به جستجو 🌍💫

✅ می‌توانید به‌صورت خودکار یا دستی قیمت‌ها را ببینید.
✅ اگر خودکار را انتخاب کنید، هر ۲۴ ساعت یکبار قیمت‌ها برای شما ارسال می‌شود.
✅ اگر دستی را انتخاب کنید، هر زمان که خواستید، می‌توانید قیمت‌ها را ببینید.
    """
    update.message.reply_text(welcome_message)

    keyboard = [
        [InlineKeyboardButton("💵 ارزهای رایج", callback_data='currency')],
        [InlineKeyboardButton("💰 ارزهای دیجیتال", callback_data='crypto')],
        [InlineKeyboardButton("✅ تأیید انتخاب", callback_data='confirm')],
        [InlineKeyboardButton("افزودن ارز ها و پول دیگر➕♦️", callback_data='add_more')]
    ]
    reply_markup = InlineKeyboardMarkup(keyboard)
    update.message.reply_text("لطفاً ارزهای مورد نظر خود را انتخاب کنید:", reply_markup=reply_markup)

# مدیریت انتخاب کاربر
def button(update: Update, context: CallbackContext):
    query = update.callback_query
    query.answer()
    chat_id = query.message.chat_id
    selected = users.get(chat_id, [])
    
    if query.data == "confirm":
        if not selected:
            query.edit_message_text("لطفاً حداقل یک ارز انتخاب کنید!")
            return
        message = "📝 لیست ارزهای انتخاب‌شده شما:\n"
        for item in selected:
            message += f"♦️ قیمت {item}\n"
        query.edit_message_text(message)
    elif query.data == "add_more":
        start(update, context)
    else:
        selected.append(query.data)
        users[chat_id] = list(set(selected))
        query.edit_message_text(f"✅ {query.data} به لیست شما اضافه شد.")

# ارسال قیمت‌ها و نمایش تغییرات
def send_prices(context: CallbackContext):
    rates = get_currency_rates()
    crypto_rates = get_crypto_rates()
    messages = compare_rates({**rates, **crypto_rates}, "all")
    
    for user in users.keys():
        for name, msg in messages.items():
            change_message = ("🔴آی آی آی" if "🔴" in msg else "🟢آخ آخ آخ")
            final_msg = f"{change_message}، {name} تغییر کرده!\n{msg}✅"
            context.bot.send_message(chat_id=user, text=final_msg)

# راه‌اندازی ربات
def main():
    updater = Updater(TELEGRAM_TOKEN, use_context=True)
    dp = updater.dispatcher
    dp.add_handler(CommandHandler("start", start))
    dp.add_handler(CallbackQueryHandler(button))
    job_queue = updater.job_queue
    job_queue.run_repeating(send_prices, interval=86400, first=10)
    updater.start_polling()
    updater.idle()

if __name__ == "__main__":
    main()
