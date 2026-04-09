# -*- coding: utf-8 -*-
import sqlite3
import telebot
from telebot import types

# ==================== НАСТРОЙКИ ====================
TOKEN = "8514583561:AAGAh2WdcLqsUM8QphE9xkzrtoruhoNTgNU"
OPERATORS_CHAT_ID = -1003841416202
ADMIN_ID = 6780581583  # Твой ID

bot = telebot.TeleBot(TOKEN)

# ==================== БАЗА ДАННЫХ ====================
def init_db():
    conn = sqlite3.connect('support.db')
    c = conn.cursor()
    
    c.execute('''CREATE TABLE IF NOT EXISTS operators (
        telegram_id INTEGER PRIMARY KEY,
        name TEXT,
        is_active INTEGER DEFAULT 0,
        current_ticket_id INTEGER
    )''')
    
    c.execute('''CREATE TABLE IF NOT EXISTS tickets (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        client_id INTEGER,
        client_name TEXT,
        client_age TEXT,
        client_request TEXT,
        operator_id INTEGER,
        status TEXT DEFAULT 'waiting',
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    )''')
    
    # Проверяем и добавляем crisis_level если нет
    c.execute("PRAGMA table_info(tickets)")
    columns = [col[1] for col in c.fetchall()]
    if 'crisis_level' not in columns:
        c.execute("ALTER TABLE tickets ADD COLUMN crisis_level TEXT DEFAULT 'green'")
    
    c.execute('''CREATE TABLE IF NOT EXISTS client_states (
        client_id INTEGER PRIMARY KEY,
        state TEXT,
        temp_data TEXT
    )''')
    
    conn.commit()
    conn.close()
    print("База данных готова")

init_db()

# ==================== РАБОТА С СОСТОЯНИЯМИ ====================
def get_state(user_id):
    conn = sqlite3.connect('support.db')
    c = conn.cursor()
    c.execute('SELECT state FROM client_states WHERE client_id = ?', (user_id,))
    row = c.fetchone()
    conn.close()
    return row[0] if row else None

def set_state(user_id, state, temp_data=''):
    conn = sqlite3.connect('support.db')
    c = conn.cursor()
    c.execute('REPLACE INTO client_states (client_id, state, temp_data) VALUES (?, ?, ?)',
              (user_id, state, temp_data))
    conn.commit()
    conn.close()

def clear_state(user_id):
    conn = sqlite3.connect('support.db')
    c = conn.cursor()
    c.execute('DELETE FROM client_states WHERE client_id = ?', (user_id,))
    conn.commit()
    conn.close()

def get_temp_data(user_id):
    conn = sqlite3.connect('support.db')
    c = conn.cursor()
    c.execute('SELECT temp_data FROM client_states WHERE client_id = ?', (user_id,))
    row = c.fetchone()
    conn.close()
    return row[0] if row else ''

# ==================== КЛАВИАТУРЫ ====================
def operator_menu():
    markup = types.InlineKeyboardMarkup(row_width=1)
    markup.add(
        types.InlineKeyboardButton("🟢 НАЧАТЬ СМЕНУ", callback_data="op_start"),
        types.InlineKeyboardButton("🔴 ЗАКОНЧИТЬ СМЕНУ", callback_data="op_end"),
        types.InlineKeyboardButton("📋 СПИСОК ОЖИДАЮЩИХ", callback_data="op_list"),
        types.InlineKeyboardButton("👤 МОЙ ТЕКУЩИЙ ЧАТ", callback_data="op_current"),
        types.InlineKeyboardButton("✅ ЗАВЕРШИТЬ ДИАЛОГ", callback_data="op_finish")
    )
    return markup

def client_chat_menu():
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True)
    markup.add(types.KeyboardButton("🆘 СРОЧНАЯ ПОМОЩЬ"))
    markup.add(types.KeyboardButton("❌ Завершить диалог"))
    return markup

def operator_in_chat_menu():
    markup = types.InlineKeyboardMarkup(row_width=2)
    markup.add(
        types.InlineKeyboardButton("📎 Техники", callback_data="op_tech"),
        types.InlineKeyboardButton("⚠️ Эскалация", callback_data="op_escalate"),
        types.InlineKeyboardButton("✅ Завершить", callback_data="op_finish")
    )
    return markup

def techniques_menu():
    markup = types.InlineKeyboardMarkup(row_width=1)
    markup.add(
        types.InlineKeyboardButton("🌬️ Дыхание по квадрату", callback_data="tech_breath"),
        types.InlineKeyboardButton("🤚 5-4-3-2-1 заземление", callback_data="tech_ground"),
        types.InlineKeyboardButton("📝 Колесо баланса", callback_data="tech_wheel"),
        types.InlineKeyboardButton("⬅️ Назад", callback_data="op_current")
    )
    return markup

def ticket_card_keyboard(ticket_id):
    markup = types.InlineKeyboardMarkup()
    markup.add(types.InlineKeyboardButton("🤝 ВЗЯТЬ В РАБОТУ", callback_data=f"take_{ticket_id}"))
    return markup

# ==================== РЕГИСТРАЦИЯ ОПЕРАТОРА ====================
@bot.message_handler(commands=['register_op'])
def register_operator(message):
    conn = sqlite3.connect('support.db')
    c = conn.cursor()
    c.execute('SELECT * FROM operators WHERE telegram_id = ?', (message.from_user.id,))
    if c.fetchone():
        bot.send_message(message.chat.id, "✅ Ты уже зарегистрирован!", reply_markup=operator_menu())
    else:
        c.execute('INSERT INTO operators (telegram_id, name) VALUES (?, ?)',
                  (message.from_user.id, message.from_user.full_name))
        conn.commit()
        bot.send_message(message.chat.id, 
            "🔰 Ты зарегистрирован как оператор!\n\n"
            "Правила:\n"
            "1. Активное слушание, без прямых советов.\n"
            "2. При риске суицида — кнопка ЭСКАЛАЦИЯ.\n"
            "3. Всегда завершай диалог кнопкой.",
            reply_markup=operator_menu())
    conn.close()

# ==================== КОЛБЭКИ ОПЕРАТОРА ====================
@bot.callback_query_handler(func=lambda call: True)
def handle_operator_callbacks(call):
    user_id = call.from_user.id
    
    if call.data == "op_start":
        conn = sqlite3.connect('support.db')
        c = conn.cursor()
        c.execute('UPDATE operators SET is_active = 1 WHERE telegram_id = ?', (user_id,))
        conn.commit()
        conn.close()
        bot.edit_message_text("🟢 Смена начата. Жди уведомлений.",
                              call.message.chat.id, call.message.message_id, reply_markup=operator_menu())
        bot.answer_callback_query(call.id, "Ты на смене!")
    
    elif call.data == "op_end":
        conn = sqlite3.connect('support.db')
        c = conn.cursor()
        c.execute('SELECT current_ticket_id FROM operators WHERE telegram_id = ?', (user_id,))
        row = c.fetchone()
        if row and row[0]:
            bot.answer_callback_query(call.id, "⚠️ Сначала заверши диалог!", show_alert=True)
        else:
            c.execute('UPDATE operators SET is_active = 0 WHERE telegram_id = ?', (user_id,))
            conn.commit()
            bot.edit_message_text("🔴 Смена завершена.",
                                  call.message.chat.id, call.message.message_id, reply_markup=operator_menu())
            bot.answer_callback_query(call.id)
        conn.close()
    
    elif call.data == "op_list":
        conn = sqlite3.connect('support.db')
        c = conn.cursor()
        c.execute('''SELECT id, client_name, crisis_level, created_at FROM tickets 
                     WHERE status = 'waiting' ORDER BY crisis_level DESC, created_at ASC''')
        tickets = c.fetchall()
        conn.close()
        
        if not tickets:
            bot.edit_message_text("📭 Нет ожидающих.", call.message.chat.id, call.message.message_id,
                                  reply_markup=operator_menu())
        else:
            text = "📋 ОЧЕРЕДЬ:\n\n"
            for t in tickets:
                emoji = {"green": "🟢", "red": "🔴"}.get(t[2], "⚪")
                text += f"{emoji} #{t[0]} — {t[1]}\n"
            bot.edit_message_text(text, call.message.chat.id, call.message.message_id,
                                  reply_markup=operator_menu())
        bot.answer_callback_query(call.id)
    
    elif call.data == "op_current":
        conn = sqlite3.connect('support.db')
        c = conn.cursor()
        c.execute('SELECT current_ticket_id FROM operators WHERE telegram_id = ?', (user_id,))
        row = c.fetchone()
        if not row or not row[0]:
            bot.answer_callback_query(call.id, "Нет активного диалога", show_alert=True)
            conn.close()
            return
        
        ticket_id = row[0]
        c.execute('SELECT client_name, client_request FROM tickets WHERE id = ?', (ticket_id,))
        t = c.fetchone()
        conn.close()
        
        text = f"👤 Диалог #{ticket_id}\nКлиент: {t[0]}\nЗапрос: {t[1][:200]}...\n\nПиши сюда — перешлю клиенту."
        bot.edit_message_text(text, call.message.chat.id, call.message.message_id,
                              reply_markup=operator_in_chat_menu())
        bot.answer_callback_query(call.id)
    
    elif call.data == "op_finish":
        conn = sqlite3.connect('support.db')
        c = conn.cursor()
        c.execute('SELECT current_ticket_id FROM operators WHERE telegram_id = ?', (user_id,))
        row = c.fetchone()
        if not row or not row[0]:
            bot.answer_callback_query(call.id, "Нет активного диалога", show_alert=True)
            conn.close()
            return
        
        ticket_id = row[0]
        c.execute('SELECT client_id, client_name FROM tickets WHERE id = ?', (ticket_id,))
        t = c.fetchone()
        client_id = t[0]
        
        c.execute('UPDATE tickets SET status = "closed" WHERE id = ?', (ticket_id,))
        c.execute('UPDATE operators SET current_ticket_id = NULL WHERE telegram_id = ?', (user_id,))
        conn.commit()
        conn.close()
        
        clear_state(client_id)
        bot.send_message(client_id, "🙏 Диалог завершён. Береги себя!\nНапиши /start если захочешь вернуться.",
                         reply_markup=types.ReplyKeyboardRemove())
        bot.edit_message_text(f"✅ Диалог #{ticket_id} завершён.",
                              call.message.chat.id, call.message.message_id, reply_markup=operator_menu())
        bot.send_message(OPERATORS_CHAT_ID, f"✅ Оператор завершил диалог #{ticket_id}")
        bot.answer_callback_query(call.id)
    
    elif call.data.startswith("take_"):
        ticket_id = int(call.data.split("_")[1])
        conn = sqlite3.connect('support.db')
        c = conn.cursor()
        
        c.execute('SELECT is_active, current_ticket_id FROM operators WHERE telegram_id = ?', (user_id,))
        op = c.fetchone()
        if not op or op[0] == 0:
            bot.answer_callback_query(call.id, "Сначала начни смену!", show_alert=True)
            conn.close()
            return
        if op[1]:
            bot.answer_callback_query(call.id, "У тебя уже есть диалог!", show_alert=True)
            conn.close()
            return
        
        c.execute('SELECT client_id, client_name, status FROM tickets WHERE id = ?', (ticket_id,))
        t = c.fetchone()
        if not t or t[2] != 'waiting':
            bot.answer_callback_query(call.id, "Тикет уже взяли!", show_alert=True)
            bot.delete_message(call.message.chat.id, call.message.message_id)
            conn.close()
            return
        
        client_id, client_name, _ = t
        
        c.execute('UPDATE tickets SET operator_id = ?, status = "in_progress" WHERE id = ?',
                  (user_id, ticket_id))
        c.execute('UPDATE operators SET current_ticket_id = ? WHERE telegram_id = ?',
                  (ticket_id, user_id))
        conn.commit()
        conn.close()
        
        bot.edit_message_text(f"✅ Тикет #{ticket_id} ({client_name}) взят.\nПиши — перешлю клиенту.",
                              call.message.chat.id, call.message.message_id,
                              reply_markup=operator_in_chat_menu())
        bot.send_message(client_id,
                         f"👋 {client_name}, с вами оператор поддержки.\nРасскажите подробнее, что случилось?",
                         reply_markup=client_chat_menu())
        bot.send_message(OPERATORS_CHAT_ID,
                         f"✅ @{call.from_user.username or 'Оператор'} взял тикет #{ticket_id}")
        bot.answer_callback_query(call.id)
    
    elif call.data == "op_tech":
        bot.edit_message_text("📚 Быстрые техники:", call.message.chat.id, call.message.message_id,
                              reply_markup=techniques_menu())
        bot.answer_callback_query(call.id)
    
    elif call.data.startswith("tech_"):
        conn = sqlite3.connect('support.db')
        c = conn.cursor()
        c.execute('SELECT current_ticket_id FROM operators WHERE telegram_id = ?', (user_id,))
        row = c.fetchone()
        if not row or not row[0]:
            bot.answer_callback_query(call.id, "Нет активного диалога", show_alert=True)
            conn.close()
            return
        
        ticket_id = row[0]
        c.execute('SELECT client_id FROM tickets WHERE id = ?', (ticket_id,))
        client_id = c.fetchone()[0]
        conn.close()
        
        techs = {
            "tech_breath": "🌬️ Дыхание по квадрату:\n\nВдох 4с → Задержка 4с → Выдох 4с → Задержка 4с\nПовтори 3-5 раз.",
            "tech_ground": "🤚 5-4-3-2-1:\n\nНазови: 5 вещей что видишь, 4 что трогаешь, 3 звука, 2 запаха, 1 вкус.",
            "tech_wheel": "📝 Колесо баланса:\n\nОцени от 1 до 10 сферы: здоровье, работа, отношения, деньги, отдых, развитие."
        }
        
        bot.send_message(client_id, techs.get(call.data, "Техника временно недоступна"))
        bot.answer_callback_query(call.id, "Отправлено клиенту")
    
    elif call.data == "op_escalate":
        conn = sqlite3.connect('support.db')
        c = conn.cursor()
        c.execute('SELECT current_ticket_id FROM operators WHERE telegram_id = ?', (user_id,))
        row = c.fetchone()
        if not row or not row[0]:
            bot.answer_callback_query(call.id, "Нет активного диалога", show_alert=True)
            conn.close()
            return
        
        ticket_id = row[0]
        c.execute('SELECT client_name, client_request FROM tickets WHERE id = ?', (ticket_id,))
        t = c.fetchone()
        conn.close()
        
        bot.send_message(ADMIN_ID,
                         f"⚠️ ЭСКАЛАЦИЯ #{ticket_id}\n"
                         f"Оператор: @{call.from_user.username}\n"
                         f"Клиент: {t[0]}\nЗапрос: {t[1]}")
        bot.answer_callback_query(call.id, "Сигнал отправлен", show_alert=True)

# ==================== ОБРАБОТКА ВСЕХ СООБЩЕНИЙ ====================
@bot.message_handler(func=lambda m: True)
def handle_all_messages(message):
    user_id = message.from_user.id
    
    if message.chat.id == OPERATORS_CHAT_ID:
        return
    
    conn = sqlite3.connect('support.db')
    c = conn.cursor()
    
    c.execute('SELECT current_ticket_id FROM operators WHERE telegram_id = ?', (user_id,))
    op = c.fetchone()
    
    if op and op[0]:
        ticket_id = op[0]
        c.execute('SELECT client_id FROM tickets WHERE id = ?', (ticket_id,))
        row = c.fetchone()
        client_id = row[0] if row else None
        conn.close()
        
        if client_id:
            bot.send_message(client_id, f"💬 *Оператор:* {message.text}", parse_mode="Markdown")
        return
    
    c.execute("SELECT id, operator_id FROM tickets WHERE client_id = ? AND status = 'in_progress'", (user_id,))
    ticket = c.fetchone()
    
    if ticket:
        ticket_id, operator_id = ticket
        conn.close()
        
        if message.text == "🆘 СРОЧНАЯ ПОМОЩЬ":
            bot.send_message(ADMIN_ID, f"🆘 КЛИЕНТ НАЖАЛ СРОЧНУЮ ПОМОЩЬ!\nТикет #{ticket_id}\nКлиент: {user_id}")
            bot.send_message(user_id, "Сигнал отправлен. Держись.\n\nТелефон доверия: 8-800-2000-122")
        elif message.text == "❌ Завершить диалог":
            conn = sqlite3.connect('support.db')
            c = conn.cursor()
            c.execute('UPDATE tickets SET status = "closed" WHERE id = ?', (ticket_id,))
            c.execute('UPDATE operators SET current_ticket_id = NULL WHERE telegram_id = ?', (operator_id,))
            conn.commit()
            conn.close()
            clear_state(user_id)
            bot.send_message(user_id, "Диалог завершён. Напиши /start если захочешь вернуться.",
                             reply_markup=types.ReplyKeyboardRemove())
            bot.send_message(operator_id, f"ℹ️ Клиент завершил диалог #{ticket_id}.")
        else:
            bot.send_message(operator_id, f"💬 *Клиент:* {message.text}",
                             reply_markup=operator_in_chat_menu(), parse_mode="Markdown")
        return
    
    state = get_state(user_id)
    conn.close()
    
    if message.text == "/start":
        bot.send_message(user_id, 
            "👋 *Привет!*\n\n"
            "Это бот психологической поддержки. Здесь работают живые люди.\n\n"
            "Как я могу к тебе обращаться?",
            parse_mode="Markdown")
        set_state(user_id, "waiting_name")
    
    elif state == "waiting_name":
        set_state(user_id, "waiting_age", message.text)
        bot.send_message(user_id, "Сколько тебе лет? (Можно примерно)")
    
    elif state == "waiting_age":
        temp = get_temp_data(user_id)
        set_state(user_id, "waiting_problem", f"{temp}||{message.text}")
        bot.send_message(user_id, 
            "Опиши, что тебя беспокоит. Что случилось?\n\n"
            "Напиши одним сообщением — это увидят операторы.")
    
    elif state == "waiting_problem":
        temp = get_temp_data(user_id)
        parts = temp.split("||")
        name = parts[0] if len(parts) > 0 else "Неизвестно"
        age = parts[1] if len(parts) > 1 else "Неизвестно"
        
        crisis_keywords = ["суицид", "убить", "повеситься", "смерть", "прыгнуть", "вскрыться", "передоз"]
        crisis = "red" if any(kw in message.text.lower() for kw in crisis_keywords) else "green"
        
        conn = sqlite3.connect('support.db')
        c = conn.cursor()
        c.execute('''INSERT INTO tickets (client_id, client_name, client_age, client_request, crisis_level, status) 
                     VALUES (?, ?, ?, ?, ?, 'waiting')''',
                  (user_id, name, age, message.text, crisis))
        ticket_id = c.lastrowid
        conn.commit()
        conn.close()
        
        emoji = "🔴" if crisis == "red" else "🟢"
        card = f"{emoji} *НОВОЕ ОБРАЩЕНИЕ #{ticket_id}*\n\n*Имя:* {name}\n*Возраст:* {age}\n*Запрос:* {message.text}"
        
        if crisis == "red":
            bot.send_message(ADMIN_ID, f"🔴🔴🔴 *КРИТИЧЕСКОЕ ОБРАЩЕНИЕ!*\n{card}", parse_mode="Markdown")
        
        bot.send_message(OPERATORS_CHAT_ID, card, reply_markup=ticket_card_keyboard(ticket_id), parse_mode="Markdown")
        bot.send_message(user_id,
            "✅ Твоя заявка отправлена. Ожидай, с тобой скоро соединятся.\n\n"
            "Если станет совсем плохо — нажми кнопку ниже.",
            reply_markup=client_chat_menu())
        
        set_state(user_id, "in_chat", str(ticket_id))

# ==================== ЗАПУСК ====================
print("Бот запущен и готов к работе!")
bot.infinity_polling()
