from telegram import Update, ReplyKeyboardMarkup
from telegram.ext import (
    ApplicationBuilder, CommandHandler, MessageHandler,
    ContextTypes, ConversationHandler, filters
)

# ==== CONFIG ====
TELEGRAM_TOKEN = "7530848717:AAGnVyIZ8KMPcSxvANEYLsulhKxm9w-vFxw"
ADMIN_CHAT_ID = 1194598554  # আপনার টেলিগ্রাম আইডি

# Conversation steps
FROM, TO, DATE, CLASS, BASEFARE, QUOTA = range(6)

# Temporary storage
tickets = {}

# ==== START ORDER ====
async def start_order(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("🚄 Please enter FROM Station Code (e.g., HWH):")
    return FROM

async def from_station(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data["from"] = update.message.text.upper()
    await update.message.reply_text("Enter TO Station Code (e.g., NDLS):")
    return TO

async def to_station(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data["to"] = update.message.text.upper()
    await update.message.reply_text("Enter Journey Date (DD-MM-YYYY):")
    return DATE

async def journey_date(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data["date"] = update.message.text
    reply_keyboard = [["SL", "3A", "2A", "1A"]]
    await update.message.reply_text(
        "Select Class:",
        reply_markup=ReplyKeyboardMarkup(reply_keyboard, one_time_keyboard=True)
    )
    return CLASS

async def travel_class(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data["class"] = update.message.text.upper()
    await update.message.reply_text("Enter Base Fare (₹):")
    return BASEFARE

async def base_fare(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        base_fare_value = int(update.message.text)
    except ValueError:
        await update.message.reply_text("❌ Please enter a valid number for fare:")
        return BASEFARE

    context.user_data["basefare"] = base_fare_value
    reply_keyboard = [["Normal", "Tatkal"]]
    await update.message.reply_text(
        "Select Quota:",
        reply_markup=ReplyKeyboardMarkup(reply_keyboard, one_time_keyboard=True)
    )
    return QUOTA

async def quota(update: Update, context: ContextTypes.DEFAULT_TYPE):
    quota_type = update.message.text
    base_fare = context.user_data["basefare"]

    if quota_type.lower() == "normal":
        total_price = base_fare + 400
    else:
        total_price = base_fare + 600

    context.user_data["quota"] = quota_type
    context.user_data["price"] = total_price

    user_id = update.message.from_user.id
    details = (
        f"🆕 New Ticket Order:\n"
        f"From: {context.user_data['from']}\n"
        f"To: {context.user_data['to']}\n"
        f"Date: {context.user_data['date']}\n"
        f"Class: {context.user_data['class']}\n"
        f"Quota: {quota_type}\n"
        f"Base Fare: ₹{base_fare}\n"
        f"Total Price: ₹{total_price}\n"
        f"User ID: {user_id}"
    )

    await update.message.reply_text(
        f"✅ Order received!\nEstimated Price: ₹{total_price}\nWe will notify you when your ticket is ready."
    )
    await context.bot.send_message(ADMIN_CHAT_ID, details)

    return ConversationHandler.END

# ==== ADMIN UPLOAD ====
async def upload_ticket(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.message.from_user.id != ADMIN_CHAT_ID:
        return
    try:
        user_id = int(context.args[0])
    except:
        await update.message.reply_text("Usage: /uploadticket USER_ID")
        return

    if not update.message.document:
        await update.message.reply_text("Please attach the ticket file with this command.")
        return

    if user_id not in tickets:
        tickets[user_id] = {}

    tickets[user_id]["file_id"] = update.message.document.file_id
    price = context.args[1] if len(context.args) > 1 else "Unknown"
    tickets[user_id]["price"] = price

    payment_info = f"🎫 Your ticket is ready!\nPlease pay ₹{price} to UPI ID: yourupi@bank\nAfter payment, reply /done"
    await context.bot.send_message(user_id, payment_info)

# ==== USER CONFIRM PAYMENT ====
async def done(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.message.from_user.id
    if user_id in tickets:
        await context.bot.send_message(
            ADMIN_CHAT_ID,
            f"💰 {user_id} claims payment done. Use /sendticket {user_id} to deliver."
        )
        await update.message.reply_text("✅ Payment confirmation sent to admin. Please wait...")
    else:
        await update.message.reply_text("❌ No pending ticket found.")

# ==== ADMIN SEND TICKET ====
async def send_ticket(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.message.from_user.id != ADMIN_CHAT_ID:
        return
    try:
        user_id = int(context.args[0])
    except:
        await update.message.reply_text("Usage: /sendticket USER_ID")
        return

    if user_id in tickets and "file_id" in tickets[user_id]:
        await context.bot.send_document(chat_id=user_id, document=tickets[user_id]["file_id"])
        await update.message.reply_text("✅ Ticket sent successfully.")
        del tickets[user_id]
    else:
        await update.message.reply_text("❌ No ticket found for this user.")

# ==== CANCEL ====
async def cancel(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("❌ Order cancelled.")
    return ConversationHandler.END

# ==== MAIN ====
if __name__ == "__main__":
    app = ApplicationBuilder().token(TELEGRAM_TOKEN).build()

    conv_handler = ConversationHandler(
        entry_points=[CommandHandler("order", start_order)],
        states={
            FROM: [MessageHandler(filters.TEXT & ~filters.COMMAND, from_station)],
            TO: [MessageHandler(filters.TEXT & ~filters.COMMAND, to_station)],
            DATE: [MessageHandler(filters.TEXT & ~filters.COMMAND, journey_date)],
            CLASS: [MessageHandler(filters.TEXT & ~filters.COMMAND, travel_class)],
            BASEFARE: [MessageHandler(filters.TEXT & ~filters.COMMAND, base_fare)],
            QUOTA: [MessageHandler(filters.TEXT & ~filters.COMMAND, quota)],
        },
        fallbacks=[CommandHandler("cancel", cancel)],
    )

    app.add_handler(conv_handler)
    app.add_handler(CommandHandler("uploadticket", upload_ticket))
    app.add_handler(CommandHandler("done", done))
    app.add_handler(CommandHandler("sendticket", send_ticket))

    print("✅ Bot running...")
    app.run_polling()

