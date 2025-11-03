# telegram-bot
import logging
import sqlite3
from datetime import datetime, timedelta
from dateutil.relativedelta import relativedelta
from telegram import Update, ReplyKeyboardMarkup
from telegram.ext import (
    Updater, CommandHandler, MessageHandler, Filters,
    ConversationHandler, CallbackContext
)

# Настройка логирования
logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    level=logging.INFO
)
logger = logging.getLogger(__name__)

# Состояния для ConversationHandler
SET_DATE = 1

class ReminderBot:
    def __init__(self, token):
        self.token = token
        self.updater = Updater(token, use_context=True)
        self.dispatcher = self.updater.dispatcher
        self.init_db()
    
    def init_db(self):
        """Создаем базу данных для хранения напоминаний"""
        conn = sqlite3.connect('reminders.db')
        cursor = conn.cursor()
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS reminders (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id INTEGER,
                payment_date TEXT,
                reminder_date TEXT,
                created_at TEXT,
                status TEXT DEFAULT 'active'
            )
        ''')
        conn.commit()
        conn.close()
    
    def start(self, update: Update, context: CallbackContext):
        """Обработчик команды /start"""
        keyboard = [
            ["📅 Установить напоминание"],
            ["📋 Мои напоминания"],
            ["❌ Остановить напоминания"]
        ]
        reply_markup = ReplyKeyboardMarkup(keyboard, resize_keyboard=True)
        
        update.message.reply_text(
            "🔔 *Бот-напоминалка об оплате интернета*\n\n"
            "Я буду напоминать вам ЕЖЕМЕСЯЧНО об оплате интернета!\n"
            "Просто установите дату первого платежа, и я буду напоминать каждый месяц.",
            parse_mode='Markdown',
            reply_markup=reply_markup
        )
    
    def main_menu(self, update: Update, context: CallbackContext):
        """Главное меню"""
        text = update.message.text
        
        if text == "📅 Установить напоминание":
            update.message.reply_text(
                "📅 Введите дату первого платежа в формате *ДД.ММ.ГГГГ*\n"
                "Например: 25.12.2024\n\n"
                "Я буду напоминать КАЖДЫЙ МЕСЯЦ в этот день!",
                parse_mode='Markdown'
            )
            return SET_DATE
            
        elif text == "📋 Мои напоминания":
            self.show_reminders(update, context)
            
        elif text == "❌ Остановить напоминания":
            self.stop_reminders(update, context)
            
        else:
            update.message.reply_text("Пожалуйста, выберите действие из меню:")
    
    def set_reminder_date(self, update: Update, context: CallbackContext):
        """Установка даты ежемесячного напоминания"""
        user_input = update.message.text
        user_id = update.effective_user.id
        
        try:
            # Парсим дату
            payment_date = datetime.strptime(user_input, '%d.%m.%Y')
            today = datetime.now()
            
            # Проверяем что дата в будущем
            if payment_date <= today:
                update.message.reply_text(
                    "❌ Дата должна быть в будущем! Введите корректную дату:"
                )
                return SET_DATE
            
            # Сохраняем в базу данных
            conn = sqlite3.connect('reminders.db')
            cursor = conn.cursor()
            cursor.execute(
                'INSERT INTO reminders (user_id, payment_date, reminder_date, created_at) VALUES (?, ?, ?, ?)',
                (user_id, 
                 payment_date.strftime('%Y-%m-%d'), 
                 payment_date.strftime('%Y-%m-%d'), 
                 datetime.now().strftime('%Y-%m-%d %H:%M:%S'))
            )
            conn.commit()
            conn.close()
            
            # Запускаем ЕЖЕМЕСЯЧНОЕ напоминание
            self.start_monthly_reminder(user_id, payment_date, context)
            
            update.message.reply_text(
                f"✅ Ежемесячное напоминание установлено!\n\n"
                f"📅 Дата платежа: *{payment_date.strftime('%d.%m.%Y')}*\n"
                f"🔔 Я буду напоминать: *КАЖДЫЙ МЕСЯЦ {payment_date.strftime('%d')} числа*\n\n"
                f"_Не забудьте оплатить интернет вовремя!_ 💰",
                parse_mode='Markdown'
            )
            
            return ConversationHandler.END
            
        except ValueError:
            update.message.reply_text(
                "❌ Неверный формат даты! Используйте ДД.ММ.ГГГГ\n"
                "Попробуйте еще раз:"
            )
            return SET_DATE
    
    def start_monthly_reminder(self, user_id, first_payment_date, context):
        """Запуск ежемесячного напоминания"""
        # Первое напоминание - в указанную дату
        first_reminder_delay = (first_payment_date - datetime.now()).total_seconds()
        
        if first_reminder_delay > 0:
            context.job_queue.run_once(
                self.send_monthly_reminder,
                first_reminder_delay,
                context=user_id,
                name=f"monthly_{user_id}"
            )
        
        # Следующие напоминания - каждый месяц
        next_month_date = first_payment_date + relativedelta(months=1)
        monthly_delay = (next_month_date - datetime.now()).total_seconds()
        
        if monthly_delay > 0:
            context.job_queue.run_repeating(
                self.send_monthly_reminder,
                interval=timedelta(days=30),  # Примерно месяц
                first=monthly_delay,
                context=user_id,
                name=f"monthly_repeat_{user_id}"
            )
    
    def send_monthly_reminder(self, context: CallbackContext):
        """Отправка ежемесячного напоминания"""
        job = context.job
        user_id = job.context
        
        reminder_text = (
            "🔔 *ЕЖЕМЕСЯЧНОЕ НАПОМИНАНИЕ ОБ ОПЛАТЕ ИНТЕРНЕТА!*\n\n"
            "💳 Пора оплатить интернет за этот месяц!\n\n"
            "⚡ Не забудьте вовремя пополнить счет,\n"
            "чтобы избежать отключения интернета.\n\n"
            "_Спасибо, что пользуетесь нашим сервисом!_ 🌐"
        )
        
        try:
            context.bot.send_message(
                chat_id=user_id,
                text=reminder_text,
                parse_mode='Markdown'
            )
            
            logger.info(f"Ежемесячное напоминание отправлено пользователю {user_id}")
                
        except Exception as e:
            logger.error(f"Ошибка отправки напоминания: {e}")
    
    def show_reminders(self, update: Update, context: CallbackContext):
        """Показать активные напоминания пользователя"""
        user_id = update.effective_user.id
        
        conn = sqlite3.connect('reminders.db')
        cursor = conn.cursor()
        cursor.execute(
            'SELECT payment_date, created_at FROM reminders WHERE user_id = ? ORDER BY created_at DESC',
            (user_id,)
        )
        reminders = cursor.fetchall()
        conn.close()
        
        if not reminders:
            update.message.reply_text("📭 У вас нет активных напоминаний.")
            return
        
        text = "📋 Ваши ежемесячные напоминания:\n\n"
        for payment_date, created_at in reminders:
            payment_dt = datetime.strptime(payment_date, '%Y-%m-%d')
            text += f"⏰ Напоминаю каждый месяц {payment_dt.strftime('%d')} числа\n"
            text += f"📅 Установлено: {created_at[:10]}\n\n"
        
        update.message.reply_text(text)
    
    def stop_reminders(self, update: Update, context: CallbackContext):
        """Остановка всех напоминаний"""
        user_id = update.effective_user.id
        
        # Удаляем задания из очереди
        current_jobs = context.job_queue.get_jobs_by_name(f"monthly_{user_id}")
        for job in current_jobs:
            job.schedule_removal()
        
        current_jobs = context.job_queue.get_jobs_by_name(f"monthly_repeat_{user_id}")
        for job in current_jobs:
            job.schedule_removal()
        
        # Удаляем из базы данных
        conn = sqlite3.connect('reminders.db')
        cursor = conn.cursor()
        cursor.execute('DELETE FROM reminders WHERE user_id = ?', (user_id,))
        conn.commit()
        conn.close()
        
        update.message.reply_text(
            "✅ Все напоминания остановлены!\n"
            "Вы можете установить новые напоминания в любое время."
        )
    
    def cancel(self, update: Update, context: CallbackContext):
        """Отмена диалога"""
        update.message.reply_text(
            "Действие отменено."
        )
        return ConversationHandler.END
    
    def setup_handlers(self):
        """Настройка обработчиков"""
        conv_handler = ConversationHandler(
            entry_points=[
                MessageHandler(Filters.regex("^(📅 Установить напоминание)$"), self.main_menu)
            ],
            states={
                SET_DATE: [
                    MessageHandler(Filters.text & ~Filters.command, self.set_reminder_date)
                ],
            },
            fallbacks=[CommandHandler("cancel", self.cancel)],
        )
        
        self.dispatcher.add_handler(CommandHandler("start", self.start))
        self.dispatcher.add_handler(conv_handler)
        self.dispatcher.add_handler(
            MessageHandler(Filters.regex("^(📋 Мои напоминания)$"), self.show_reminders)
        )
        self.dispatcher.add_handler(
            MessageHandler(Filters.regex("^(❌ Остановить напоминания)$"), self.stop_reminders)
        )
        self.dispatcher.add_handler(
            MessageHandler(Filters.text & ~Filters.command, self.main_menu)
        )
    
    def run(self):
        """Запуск бота"""
        self.setup_handlers()
        
        print("Бот запущен...")
        print("Ожидаю команду /start")
        self.updater.start_polling()
        self.updater.idle()

def main():
    # ЗАМЕНИ НА СВОЙ ТОКЕН
    TOKEN = "8256601804:AAFyEWxNU6QdYVue1baq2EkEe6A7mga0Pwc"
    
    bot = ReminderBot(TOKEN)
    bot.run()

if __name__ == '__main__':
    main()
