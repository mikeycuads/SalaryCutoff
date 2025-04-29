# main.py

from telegram import Update
from telegram.ext import ApplicationBuilder, CommandHandler, MessageHandler, filters, ContextTypes
import re

PHP_EXCHANGE_RATE = 57  # 1 USD = 57 PHP

# Temporary storage for each day's gross sales
gross_sales = []

# Function to extract TOTAL GROSS SALE from forwarded summary
def extract_gross_sale(message):
    match = re.search(r'TOTAL GROSS SALE: \$([0-9]+\.[0-9]+)', message)
    if match:
        return float(match.group(1))
    return None

# Function to calculate salary based on gross sale
def calculate_salary(gross_sale):
    net_sale = gross_sale * 0.8
    bonus = net_sale * 0.05
    salary_usd = bonus + 40
    salary_php = salary_usd * PHP_EXCHANGE_RATE
    return salary_usd, salary_php

# Start command
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "Hi! 👋 Forward the full 'Summary of Tips and VIPs' messages one by one.\n"
        "When finished, send /done to get your total salary."
    )

# Handle forwarded summaries
async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    message_text = update.message.text

    gross_sale = extract_gross_sale(message_text)
    if gross_sale is not None:
        gross_sales.append(gross_sale)
        await update.message.reply_text(
            f"✅ Day {len(gross_sales)} recorded: Gross Sale = ${gross_sale:.2f}"
        )
    else:
        await update.message.reply_text(
            "❌ Could not find 'TOTAL GROSS SALE' in this message. Make sure you're forwarding the correct daily summary."
        )

# Done command
async def done(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not gross_sales:
        await update.message.reply_text("No summaries recorded yet. Please forward some day summaries first!")
        return

    total_usd = 0
    total_php = 0
    summary_lines = []

    for i, gross in enumerate(gross_sales, start=1):
        salary_usd, salary_php = calculate_salary(gross)
        total_usd += salary_usd
        total_php += salary_php
        summary_lines.append(f"Day {i}: ${salary_usd:.2f} / ₱{salary_php:,.2f}")

    summary_text = "\n".join(summary_lines)

    await update.message.reply_text(
        f"📋 Salary Summary:\n\n"
        f"{summary_text}\n\n"
        f"==== FINAL TOTAL ====\n"
        f"Total USD Salary: ${total_usd:.2f}\n"
        f"Total PHP Salary: ₱{total_php:,.2f}"
    )

    # Reset after done
    gross_sales.clear()

def main():
    app = ApplicationBuilder().token("7272132366:AAGOR6l_60kYEXt60OkoK4mTwYBFOoc799Y").build()

    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("done", done))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))

    print("Bot is running...")
    app.run_polling()

if __name__ == "__main__":
    main()
