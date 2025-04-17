import loggin
from telegram import Update, InlineKeyboardButton,
InlineKeyboardMarkup
from telegram.ext import Updater, CommandHandler, CallbackQueryHandler, ConversationHandler, MessageHandler, Filters
import requests

# Настройка логирования
logging.basicConfig(format='%(asctime)s - %(name)s - %(levelname)s - %(message)s', level=logging.INFO)
logger = logging.getLogger(name)

# Токен вашего бота
TOKEN = 'YOUR_BOT_TOKEN'

# Ассортимент товаров
products = {
    'product1': {'name': 'Товар 1', 'price': 100},
    'product2': {'name': 'Товар 2', 'price': 200},
    'product3': {'name': 'Товар 3', 'price': 300},
}

# Состояния диалога
SELECTING_PRODUCT, CONFIRMING_PURCHASE, SENDING_PAYMENT_DETAILS = range(3)

# Функция для начала диалога
def start(update: Update, context):
    keyboard = [[InlineKeyboardButton(product['name'], callback_data=product_id)] for product_id, product in products.items()]
    reply_markup = InlineKeyboardMarkup(keyboard)
    update.message.reply_text('Выберите товар:', reply_markup=reply_markup)
    return SELECTING_PRODUCT

# Функция для обработки выбора товара
def select_product(update: Update, context):
    query = update.callback_query
    product_id = query.data
    product = products[product_id]
    context.user_data['product'] = product
    query.answer()
    query.edit_message_text(f'Вы выбрали {product["name"]}. Цена: {product["price"]} руб. Подтвердить покупку?', reply_markup=InlineKeyboardMarkup([
        [InlineKeyboardButton('Да', callback_data='confirm')],
        [InlineKeyboardButton('Нет', callback_data='cancel')]
    ]))
    return CONFIRMING_PURCHASE

# Функция для подтверждения покупки
def confirm_purchase(update: Update, context):
    query = update.callback_query
    if query.data == 'confirm':
        product = context.user_data['product']
        query.answer()
        query.edit_message_text(f'Покупка {product["name"]} подтверждена. Цена: {product["price"]} руб. Отправляем данные для оплаты...')
        # Отправка SMS с данными для оплаты
        send_payment_details(update, context)
        return SENDING_PAYMENT_DETAILS
    else:
        query.answer()
        query.edit_message_text('Покупка отменена.')
        return ConversationHandler.END

# Функция для отправки SMS с данными для оплаты
def send_payment_details(update: Update, context):
    product = context.user_data['product']
    payment_details = f'Данные для оплаты:\nТовар: {product["name"]}\nЦена: {product["price"]} руб.\nНомер счета: 1234567890'
    update.message.reply_text(payment_details)
    return ConversationHandler.END

# Функция для отмены покупки
def cancel(update: Update, context):
    query = update.callback_query
    query.answer()
    query.edit_message_text('Покупка отменена.')
    return ConversationHandler.END

# Функция для обработки ошибок
def error(update, context):
    logger.warning('Update "%s" caused error "%s"', update, context.error)

# Основная функция
def main():
    updater = Updater(TOKEN, use_context=True)
    dp = updater.dispatcher

    conv_handler = ConversationHandler(
        entry_points=[CommandHandler('start', start)],
        states={
            SELECTING_PRODUCT: [CallbackQueryHandler(select_product)],
            CONFIRMING_PURCHASE: [CallbackQueryHandler(confirm_purchase)],
            SENDING_PAYMENT_DETAILS: []
        },
        fallbacks=[CallbackQueryHandler(cancel)]
    )

    dp.add_handler(conv_handler)
    dp.add_error_handler(error)

    updater.start_polling()
    updater.idle()

if name == 'main':
    main()
