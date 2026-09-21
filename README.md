# Uploading-an-existing-fileimport os
import sqlite3
from datetime import date, datetime
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup, ReplyKeyboardMarkup
from telegram.ext import (
    Application, CommandHandler, MessageHandler, CallbackQueryHandler,
    ConversationHandler, ContextTypes, filters
)

TOKEN = os.getenv("BOT_TOKEN")
DB = os.getenv("DB_PATH", "attendance.db")

ADD_NAME, RENAME_PICK, RENAME_NAME, DELETE_PICK, TRAINING_DATE, TRAINING_MARK = range(6)

MAIN_KB = ReplyKeyboardMarkup(
    [
        ["🔥 Огневая подготовка", "🧭 Тактическая подготовка"],
        ["👥 Участники", "📊 Журнал"],
    ],
    resize_keyboard=True
)

def conn():
    c = sqlite3.connect(DB)
    c.row_factory = sqlite3.Row
    return c

def init_db():
    with conn() as c:
        c.executescript("""
        CREATE TABLE IF NOT EXISTS participants (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL,
            active INTEGER NOT NULL DEFAULT 1
        );
        CREATE TABLE IF NOT EXISTS attendance (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            participant_id INTEGER NOT NULL,
            training_type TEXT NOT NULL,
            training_date TEXT NOT NULL,
            present INTEGER NOT NULL,
            UNIQUE(participant_id, training_type, training_date),
            FOREIGN KEY(participant_id) REFERENCES participants(id)
        );
        """)

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("Журнал посещаемости", reply_markup=MAIN_KB)

async def participants_menu(update: Update, context: ContextTypes.DEFAULT_TYPE):
    kb = InlineKeyboardMarkup([
        [InlineKeyboardButton("➕ Добавить", callback_data="p:add"),
         InlineKeyboardButton("✏️ Переименовать", callback_data="p:rename")],
        [InlineKeyboardButton("🗑 Удалить", callback_data="p:delete"),
         InlineKeyboardButton("📋 Список", callback_data="p:list")]
    ])
    await update.message.reply_text("👥 Участники", reply_markup=kb)

def active_people():
    with conn() as c:
        return c.execute("SELECT id,name FROM participants WHERE active=1 ORDER BY name").fetchall()

async def p_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    q = update.callback_query
    await q.answer()
    action = q.data.split(":")[1]
    if action == "add":
        await q.message.reply_text("Введите ФИО участника:")
        return ADD_NAME
    people = active_people()
    if action == "list":
        text = "📋 Список участников:\n\n" + ("\n".join(f"{i+1}. {p['name']}" for i,p in enumerate(people)) if people else "Список пуст.")
        await q.message.reply_text(text)
        return ConversationHandler.END
    if not people:
        await q.message.reply_text("Список участников пуст.")
        return ConversationHandler.END
    prefix = "ren" if action == "rename" else "del"
    kb = [[InlineKeyboardButton(p["name"], callback_data=f"{prefix}:{p['id']}")] for p in people]
    await q.message.reply_text("Выберите участника:", reply_markup=InlineKeyboardMarkup(kb))
    return RENAME_PICK if action == "rename" else DELETE_PICK

async def add_name(update: Update, context: ContextTypes.DEFAULT_TYPE):
    name = update.message.text.strip()
    if not name:
        await update.message.reply_text("Имя не может быть пустым.")
        return ADD_NAME
    with conn() as c:
        c.execute("INSERT INTO participants(name) VALUES(?)", (name,))
    await update.message.reply_text(f"✅ Добавлен: {name}", reply_markup=MAIN_KB)
    return ConversationHandler.END

async def rename_pick(update: Update, context: ContextTypes.DEFAULT_TYPE):
    q=update.callback_query; await q.answer()
    context.user_data["rename_id"] = int(q.data.split(":")[1])
    await q.message.reply_text("Введите новое ФИО:")
    return RENAME_NAME

async def rename_name(update: Update, context: ContextTypes.DEFAULT_TYPE):
    name=update.message.text.strip()
    with conn() as c:
        c.execute("UPDATE participants SET name=? WHERE id=?", (name, context.user_data["rename_id"]))
    await update.message.reply_text("✅ Имя изменено.", reply_markup=MAIN_KB)
    return ConversationHandler.END

async def delete_pick(update: Update, context: ContextTypes.DEFAULT_TYPE):
    q=update.callback_query; await q.answer()
    pid=int(q.data.split(":")[1])
    with conn() as c:
        p=c.execute("SELECT name FROM participants WHERE id=?", (pid,)).fetchone()
    kb=InlineKeyboardMarkup([[
        InlineKeyboardButton("Да, удалить", callback_data=f"confirmdel:{pid}"),
        InlineKeyboardButton("Отмена", callback_data="confirmdel:cancel")
    ]])
    await q.message.reply_text(f"Удалить «{p['name']}» из активного списка?\nСтарая история посещений сохранится.", reply_markup=kb)
    return DELETE_PICK

async def confirm_delete(update: Update, context: ContextTypes.DEFAULT_TYPE):
    q=update.callback_query; await q.answer()
    val=q.data.split(":")[1]
    if val != "cancel":
        with conn() as c:
            c.execute("UPDATE participants SET active=0 WHERE id=?", (int(val),))
        await q.message.reply_text("🗑 Участник удалён из активного списка.")
    else:
        await q.message.reply_text("Отменено.")
    return ConversationHandler.END

async def begin_training(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data["training_type"] = "fire" if "Огневая" in update.message.text else "tactical"
    today=date.today().strftime("%d.%m.%Y")
    kb=InlineKeyboardMarkup([
        [InlineKeyboardButton(f"Сегодня — {today}", callback_data="date:today")],
        [InlineKeyboardButton("📅 Другая дата", callback_data="date:other")]
    ])
    await update.message.reply_text("Выберите дату занятия:", reply_markup=kb)
    return TRAINING_DATE

async def training_date_cb(update: Update, context: ContextTypes.DEFAULT_TYPE):
    q=update.callback_query; await q.answer()
    if q.data == "date:today":
        context.user_data["training_date"] = date.today().isoformat()
        return await show_marking(q.message, context)
    await q.message.reply_text("Введите дату в формате ДД.ММ.ГГГГ:")
    return TRAINING_DATE

async def training_date_text(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        d=datetime.strptime(update.message.text.strip(), "%d.%m.%Y").date()
    except ValueError:
        await update.message.reply_text("Неверный формат. Например: 21.09.2026")
        return TRAINING_DATE
    context.user_data["training_date"]=d.isoformat()
    return await show_marking(update.message, context)

async def show_marking(message, context):
    people=active_people()
    if not people:
        await message.reply_text("Сначала добавьте участников через 👥 Участники.", reply_markup=MAIN_KB)
        return ConversationHandler.END
    context.user_data["marks"]={}
    return await render_marks(message, context)

async def render_marks(message, context):
    people=active_people()
    marks=context.user_data["marks"]
    rows=[]
    for p in people:
        mark=marks.get(p["id"])
        icon="✅" if mark==1 else ("❌" if mark==0 else "⬜")
        rows.append([InlineKeyboardButton(f"{icon} {p['name']}", callback_data=f"mark:{p['id']}")])
    rows.append([InlineKeyboardButton("💾 Сохранить", callback_data="mark:save")])
    await message.reply_text("Нажимайте на имя: ⬜ → ✅ → ❌\nПосле отметки нажмите «Сохранить».", reply_markup=InlineKeyboardMarkup(rows))
    return TRAINING_MARK

async def mark_cb(update: Update, context: ContextTypes.DEFAULT_TYPE):
    q=update.callback_query; await q.answer()
    val=q.data.split(":")[1]
    if val=="save":
        people=active_people()
        marks=context.user_data.get("marks",{})
        if any(p["id"] not in marks for p in people):
            await q.answer("Отметьте каждого участника.", show_alert=True)
            return TRAINING_MARK
        with conn() as c:
            for p in people:
                c.execute("""INSERT INTO attendance(participant_id,training_type,training_date,present)
                             VALUES(?,?,?,?)
                             ON CONFLICT(participant_id,training_type,training_date)
                             DO UPDATE SET present=excluded.present""",
                          (p["id"], context.user_data["training_type"], context.user_data["training_date"], marks[p["id"]]))
        await q.edit_message_text("✅ Посещаемость сохранена.")
        await q.message.reply_text("Главное меню:", reply_markup=MAIN_KB)
        return ConversationHandler.END
    pid=int(val)
    cur=context.user_data["marks"].get(pid)
    context.user_data["marks"][pid] = 1 if cur is None else (0 if cur==1 else 1)
    # Replace keyboard in the same message.
    people=active_people(); marks=context.user_data["marks"]; rows=[]
    for p in people:
        mark=marks.get(p["id"])
        icon="✅" if mark==1 else ("❌" if mark==0 else "⬜")
        rows.append([InlineKeyboardButton(f"{icon} {p['name']}", callback_data=f"mark:{p['id']}")])
    rows.append([InlineKeyboardButton("💾 Сохранить", callback_data="mark:save")])
    await q.edit_message_reply_markup(reply_markup=InlineKeyboardMarkup(rows))
    return TRAINING_MARK

async def journal(update: Update, context: ContextTypes.DEFAULT_TYPE):
    with conn() as c:
        rows=c.execute("""
        SELECT a.training_date,a.training_type,
               SUM(a.present) present_count, COUNT(*) total_count
        FROM attendance a
        GROUP BY a.training_date,a.training_type
        ORDER BY a.training_date DESC
        LIMIT 30
        """).fetchall()
    if not rows:
        await update.message.reply_text("Журнал пока пуст.")
        return
    lines=["📊 Последние занятия:\n"]
    for r in rows:
        d=datetime.strptime(r["training_date"],"%Y-%m-%d").strftime("%d.%m.%Y")
        typ="🔥 Огневая" if r["training_type"]=="fire" else "🧭 Тактическая"
        lines.append(f"{d} — {typ}: {r['present_count']}/{r['total_count']} присутствовали")
    await update.message.reply_text("\n".join(lines))

async def cancel(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("Отменено.", reply_markup=MAIN_KB)
    return ConversationHandler.END

def main():
    if not TOKEN:
        raise RuntimeError("Не задан BOT_TOKEN")
    init_db()
    app=Application.builder().token(TOKEN).build()

    participant_conv=ConversationHandler(
        entry_points=[CallbackQueryHandler(p_callback, pattern=r"^p:")],
        states={
            ADD_NAME:[MessageHandler(filters.TEXT & ~filters.COMMAND, add_name)],
            RENAME_PICK:[CallbackQueryHandler(rename_pick, pattern=r"^ren:")],
            RENAME_NAME:[MessageHandler(filters.TEXT & ~filters.COMMAND, rename_name)],
            DELETE_PICK:[
                CallbackQueryHandler(delete_pick, pattern=r"^del:"),
                CallbackQueryHandler(confirm_delete, pattern=r"^confirmdel:")
            ],
        },
        fallbacks=[CommandHandler("cancel", cancel)],
        allow_reentry=True
    )
    training_conv=ConversationHandler(
        entry_points=[MessageHandler(filters.Regex(r"^(🔥 Огневая подготовка|🧭 Тактическая подготовка)$"), begin_training)],
        states={
            TRAINING_DATE:[
                CallbackQueryHandler(training_date_cb, pattern=r"^date:"),
                MessageHandler(filters.TEXT & ~filters.COMMAND, training_date_text)
            ],
            TRAINING_MARK:[CallbackQueryHandler(mark_cb, pattern=r"^mark:")]
        },
        fallbacks=[CommandHandler("cancel", cancel)]
    )

    app.add_handler(CommandHandler("start", start))
    app.add_handler(training_conv)
    app.add_handler(MessageHandler(filters.Regex(r"^👥 Участники$"), participants_menu))
    app.add_handler(participant_conv)
    app.add_handler(MessageHandler(filters.Regex(r"^📊 Журнал$"), journal))
    app.run_polling()

if __name__ == "__main__":
    main()
