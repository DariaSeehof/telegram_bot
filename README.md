import sys
import asyncio
import openai
from telegram import Update
from telegram.ext import (
   Application,
   CommandHandler,
   MessageHandler,
   ConversationHandler,
   filters,
   CallbackContext,
)
# Для корректной работы в Windows устанавливаем соответствующую политику для цикла событий
if sys.platform.startswith('win'):
   asyncio.set_event_loop_policy(asyncio.WindowsSelectorEventLoopPolicy())
# Токен Telegram-бота
TOKEN = '7714756639:AAFSuz329D6TW1arFoGGkRRqaZ7ggS8lF7Q'
# API ключ OpenAI (при необходимости замените на актуальное значение)
openai.api_key = 'sk-proj-gfDrjBVHR-_aBh7x_fdrKSYbEITsoSPYmEcghqHItx0utxczjqUcI7L98hQ33cAXbgWmF8tTST3BlbkFJ1x3yhcJ8dIhKoh1SzNNl2y_9Z9VAnj1NrucID5OE9Xmlr9E03NStQiSMZEs0GnKIRNiiFvOIAA'
# Идентификатор чата администратора заменён на числовой ID
ADMIN_CHAT_ID = 5212334758
# Определяем состояния разговора
NAME, AGE, PROFESSION, EXPERIENCE = range(4)
# Обработчик команды /start
async def start(update: Update, context: CallbackContext) -> int:
   await update.message.reply_text(
       "Здравствуйте! Для составления анкеты ответьте, пожалуйста, на несколько вопросов.\nКак вас зовут?"
   )
   return NAME
# Обработка ответа на вопрос "Как вас зовут?"
async def get_name(update: Update, context: CallbackContext) -> int:
   context.user_data["name"] = update.message.text
   await update.message.reply_text("Сколько вам лет?")
   return AGE
# Обработка ответа на вопрос "Сколько вам лет?"
async def get_age(update: Update, context: CallbackContext) -> int:
   context.user_data["age"] = update.message.text
   await update.message.reply_text("Какая у вас профессия?")
   return PROFESSION
# Обработка ответа на вопрос "Какая у вас профессия?"
async def get_profession(update: Update, context: CallbackContext) -> int:
   context.user_data["profession"] = update.message.text
   await update.message.reply_text("Опишите, пожалуйста, ваш опыт работы.")
   return EXPERIENCE
# Обработка ответа на вопрос об опыте работы и формирование анкеты
async def get_experience(update: Update, context: CallbackContext) -> int:
   context.user_data["experience"] = update.message.text
   # Формируем сообщение для GPT на основе собранных данных
   prompt = (
       "Составь, пожалуйста, структурированную анкету, используя следующие данные:\n"
       f"Имя: {context.user_data.get('name')}\n"
       f"Возраст: {context.user_data.get('age')}\n"
       f"Профессия: {context.user_data.get('profession')}\n"
       f"Опыт работы: {context.user_data.get('experience')}\n"
       "Анкета должна быть оформлена в виде понятного отчёта."
   )
   try:
       # Если вы хотите использовать новый API OpenAI, выполните миграцию командой `openai migrate`
       # или зафиксируйте версию библиотеки, например, pip install openai==0.28
       response = openai.ChatCompletion.create(
           model="gpt-3.5-turbo",
           messages=[{"role": "user", "content": prompt}],
           max_tokens=300,
       )
       questionnaire = response.choices[0].message.content.strip()
   except Exception as e:
       questionnaire = f"Произошла ошибка при генерации анкеты: {e}"
   # Отправляем анкету пользователю
   await update.message.reply_text("Ваша анкета готова:\n\n" + questionnaire)
   # Отправляем анкету администратору
   try:
       await context.bot.send_message(chat_id=ADMIN_CHAT_ID, text="Новая анкета:\n\n" + questionnaire)
   except Exception as e:
       await update.message.reply_text(f"Не удалось отправить анкету администратору: {e}")
   # Завершаем разговор
   return ConversationHandler.END
# Обработчик команды /cancel для прерывания разговора
async def cancel(update: Update, context: CallbackContext) -> int:
   await update.message.reply_text("Процесс составления анкеты прерван.")
   return ConversationHandler.END
# Главная функция для запуска бота
async def main():
   application = Application.builder().token(TOKEN).build()
   # Создаём ConversationHandler для управления последовательностью вопросов
   conv_handler = ConversationHandler(
       entry_points=[CommandHandler('start', start)],
       states={
           NAME: [MessageHandler(filters.TEXT & ~filters.COMMAND, get_name)],
           AGE: [MessageHandler(filters.TEXT & ~filters.COMMAND, get_age)],
           PROFESSION: [MessageHandler(filters.TEXT & ~filters.COMMAND, get_profession)],
           EXPERIENCE: [MessageHandler(filters.TEXT & ~filters.COMMAND, get_experience)],
       },
       fallbacks=[CommandHandler('cancel', cancel)],
   )
   application.add_handler(conv_handler)
   # Инициализируем и запускаем бота
   await application.initialize()
   await application.start()
   await application.updater.start_polling()
   # Бесконечное ожидание (бот будет работать до прерывания вручную, например, Ctrl+C)
   await asyncio.Future()
# Запуск приложения
if __name__ == '__main__':
   asyncio.run(main())
