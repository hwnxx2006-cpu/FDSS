[ai_studio_code.py](https://github.com/user-attachments/files/21958715/ai_studio_code.py)
# -*- coding: utf-8 -*-

# ===================================================================================
#                      🚀 بوت انستغرام الذكي - نسخة محسّنة 🚀
# ===================================================================================
#  المميزات:
#  - واجهة جميلة وسهلة الاستخدام مع أيقونات (Emojis).
#  - رسائل ديناميكية تتحدث في نفس المكان لتجربة أنظف.
#  - رسالة ترحيب شخصية مع صورة.
#  - إدارة سهلة للحسابات والتعليقات.
#  - منطق مهام محسن مع سلوك شبه بشري لتجنب الحظر.
#  - تقارير مفصلة وواضحة عند انتهاء كل مهمة.
# ===================================================================================

import logging
import time
import csv
import random
import os
import re
from telegram import Update, InlineKeyboardMarkup, InlineKeyboardButton
from telegram.ext import (
    Application,
    CommandHandler,
    ConversationHandler,
    CallbackQueryHandler,
    MessageHandler,
    filters,
    ContextTypes,
)
from telegram.constants import ParseMode
from instagrapi import Client
from instagrapi.exceptions import LoginRequired, ChallengeRequired, FeedbackRequired, PrivateError, UserNotFound

# --- (1) الإعدادات الأساسية ---
# !! هام جداً: ضع التوكن الخاص ببوتك هنا !!
TELEGRAM_BOT_TOKEN = "8233255170:AAHOU_Imwp9_6gEDoBveDvcGOo0Gzg5923U"

# --- أسماء الملفات والدلائل ---
ACCOUNTS_FILE = "accounts.csv"
COMMENTS_FILE = "comments.txt"
SESSIONS_DIR = "sessions"
WELCOME_IMAGE = "welcome.png" # اسم ملف الصورة الترحيبية (ضعه بجانب ملف البوت)

# --- إعدادات تسجيل الأحداث (Logging) ---
logging.basicConfig(format="%(asctime)s - %(name)s - %(levelname)s - %(message)s", level=logging.INFO)
logger = logging.getLogger(__name__)

# --- تعريف مراحل المحادثة (States) ---
(
    MAIN_MENU,
    ACCOUNT_MENU, ADD_ACCOUNT_PROMPT, REMOVE_ACCOUNT_SELECT,
    TASK_MENU, SELECT_ACCOUNTS, GET_LINK, GET_COMMENT, GET_DELAYS, CONFIRM_TASK,
    COMMENT_MENU, ADD_COMMENT_PROMPT, REMOVE_COMMENT_SELECT
) = range(13)

# --- (2) الدوال المساعدة لإدارة الملفات ---
def load_accounts():
    """تحميل حسابات انستغرام من ملف CSV."""
    if not os.path.exists(ACCOUNTS_FILE): return []
    try:
        with open(ACCOUNTS_FILE, mode='r', encoding='utf-8') as f:
            reader = csv.reader(f)
            # [username, password, proxy]
            return [{"username": r[0].strip(), "password": r[1].strip(), "proxy": r[2].strip() if len(r) > 2 and r[2].strip() else None} for r in reader if r and len(r) >= 2]
    except (IOError, csv.Error) as e:
        logger.error(f"Error loading accounts file: {e}")
        return []

def save_accounts(accounts_data):
    """حفظ حسابات انستغرام في ملف CSV."""
    try:
        with open(ACCOUNTS_FILE, mode='w', newline='', encoding='utf-8') as f:
            writer = csv.writer(f)
            for acc in accounts_data: writer.writerow([acc['username'], acc['password'], acc['proxy'] or ''])
    except IOError as e:
        logger.error(f"Error saving accounts file: {e}")


def load_comments():
    """تحميل قائمة التعليقات من ملف نصي."""
    if not os.path.exists(COMMENTS_FILE): return []
    try:
        with open(COMMENTS_FILE, 'r', encoding='utf-8') as f:
            return [line.strip() for line in f if line.strip()]
    except IOError as e:
        logger.error(f"Error loading comments file: {e}")
        return []

def save_comments(comments):
    """حفظ قائمة التعليقات في ملف نصي."""
    try:
        with open(COMMENTS_FILE, 'w', encoding='utf-8') as f:
            f.write('\n'.join(comments))
    except IOError as e:
        logger.error(f"Error saving comments file: {e}")

def process_spintax(text):
    """معالجة نص بصيغة Spintax لاختيار كلمة عشوائية."""
    pattern = re.compile(r'{([^{}]*)}')
    while True:
        match = pattern.search(text)
        if not match: break
        options = match.group(1).split('|')
        text = text[:match.start()] + random.choice(options) + text[match.end():]
    return text

# --- (3) دوال إنشاء واجهات الأزرار "الجميلة" ---
def main_menu_keyboard():
    return InlineKeyboardMarkup([
        [InlineKeyboardButton("🚀 بدء مهمة جديدة", callback_data="start_task")],
        [InlineKeyboardButton("⚙️ إدارة الحسابات", callback_data="manage_accounts")],
        [InlineKeyboardButton("📝 إدارة التعليقات", callback_data="manage_comments")]
    ])

def account_menu_keyboard():
    return InlineKeyboardMarkup([
        [InlineKeyboardButton("➕ إضافة حساب جديد", callback_data="add_account_prompt")],
        [InlineKeyboardButton("➖ حذف حساب", callback_data="remove_account_select")],
        [InlineKeyboardButton("📋 عرض كل الحسابات", callback_data="list_accounts")],
        [InlineKeyboardButton("⬅️ العودة للقائمة الرئيسية", callback_data="back_to_main")]
    ])

def task_menu_keyboard():
    return InlineKeyboardMarkup([
        [InlineKeyboardButton("🦸 تفاعل سوبر (متابعة+لايك+تعليق)", callback_data="task_super_interact")],
        [InlineKeyboardButton("💬 تعليق على منشور", callback_data="task_comment")],
        [InlineKeyboardButton("❤️ إعجاب بمنشور", callback_data="task_like")],
        [InlineKeyboardButton("👤 متابعة حساب", callback_data="task_follow")],
        [InlineKeyboardButton("⬅️ العودة للقائمة الرئيسية", callback_data="back_to_main")]
    ])

def select_accounts_keyboard(all_accounts, selected):
    keyboard = [[InlineKeyboardButton(f"{'✅ ' if acc['username'] in selected else '🔲 '}{acc['username']}", callback_data=f"select_acc_{acc['username']}")] for acc in all_accounts]
    keyboard.extend([
        [InlineKeyboardButton("✨ تحديد الكل", callback_data="select_all"), InlineKeyboardButton("⭕️ إلغاء الكل", callback_data="deselect_all")],
        [InlineKeyboardButton(f"➡️ تأكيد الإختيار ({len(selected)})", callback_data="confirm_selection")],
        [InlineKeyboardButton("⬅️ العودة لقائمة المهام", callback_data="back_to_task_menu")]
    ])
    return InlineKeyboardMarkup(keyboard)

def comment_menu_keyboard():
    return InlineKeyboardMarkup([
        [InlineKeyboardButton("➕ إضافة تعليق جديد", callback_data="add_comment_prompt")],
        [InlineKeyboardButton("📋 عرض وحذف التعليقات", callback_data="list_comments")],
        [InlineKeyboardButton("⬅️ العودة للقائمة الرئيسية", callback_data="back_to_main")]
    ])

def list_comments_keyboard(comments):
    keyboard = [[InlineKeyboardButton(f"🗑️ {comment[:40]}...", callback_data=f"remove_comment_{i}")] for i, comment in enumerate(comments)]
    keyboard.append([InlineKeyboardButton("⬅️ العودة", callback_data="back_to_comment_menu")])
    return InlineKeyboardMarkup(keyboard)

def remove_account_keyboard(all_accounts):
    keyboard = [[InlineKeyboardButton(f"🗑️ {acc['username']}", callback_data=f"remove_acc_{acc['username']}")] for acc in all_accounts]
    keyboard.append([InlineKeyboardButton("⬅️ العودة", callback_data="back_to_account_menu")])
    return InlineKeyboardMarkup(keyboard)

# --- (4) دوال المحادثة الرئيسية (البداية والقوائم) ---
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    """إرسال رسالة ترحيب شخصية مع صورة عند إرسال أمر /start."""
    user_name = update.effective_user.first_name
    context.user_data.clear()
    
    welcome_caption = (
        f"👋 أهلاً بك يا {user_name}!\n\n"
        "أنا مساعدك الذكي لأتمتة مهام انستغرام بكل سهولة وأمان.\n\n"
        "اختر ما تريد القيام به من القائمة بالأسفل. 👇"
    )
    
    if os.path.exists(WELCOME_IMAGE):
        try:
            await update.message.reply_photo(
                photo=open(WELCOME_IMAGE, 'rb'),
                caption=welcome_caption,
                reply_markup=main_menu_keyboard()
            )
        except Exception as e:
            logger.error(f"Could not send photo: {e}")
            await update.message.reply_text(welcome_caption, reply_markup=main_menu_keyboard())
    else:
        await update.message.reply_text(welcome_caption, reply_markup=main_menu_keyboard())
        
    return MAIN_MENU

async def main_menu_handler(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    """التعامل مع العودة إلى القائمة الرئيسية بشكل ديناميكي."""
    query = update.callback_query
    await query.answer()
    text = "🏠 القائمة الرئيسية:\nاختر الخدمة التي تريدها."
    try:
        await query.edit_message_text(text=text, reply_markup=main_menu_keyboard())
    except Exception as e: # In case the original message was a photo
        await query.message.reply_text(text=text, reply_markup=main_menu_keyboard())
        await query.message.delete()
    return MAIN_MENU

# --- (5) دوال إدارة الحسابات والتعليقات ---
async def account_menu_handler(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    query = update.callback_query; await query.answer(); await query.edit_message_text("⚙️ **إدارة الحسابات**\n\nتحكم في حسابات انستغرام التي يستخدمها البوت.", reply_markup=account_menu_keyboard(), parse_mode=ParseMode.MARKDOWN); return ACCOUNT_MENU

async def list_accounts_handler(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    query = update.callback_query; await query.answer()
    accounts = load_accounts()
    message = "<b>📋 قائمة الحسابات المضافة:</b>\n\n" + "\n".join(f"👤 <code>{acc['username']}</code>" for acc in accounts) if accounts else "⚠️ لا يوجد أي حسابات مضافة حالياً."
    await query.edit_message_text(message, parse_mode='HTML', reply_markup=account_menu_keyboard()); return ACCOUNT_MENU

async def add_account_prompt_handler(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    query = update.callback_query; await query.answer()
    text = ("✍️ **لإضافة حساب، أرسل المعلومات بالصيغة التالية:**\n\n`username password`\n\n**أو مع بروكسي (اختياري):**\n`username password http://user:pass@host:port`")
    await query.edit_message_text(text, parse_mode=ParseMode.MARKDOWN); return ADD_ACCOUNT_PROMPT

async def handle_add_account(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    parts = update.message.text.strip().split()
    if len(parts) not in [2, 3]:
        await update.message.reply_text("⚠️ صيغة خاطئة. يرجى المحاولة مرة أخرى.", reply_markup=account_menu_keyboard()); return ACCOUNT_MENU
    username, password, proxy = parts[0], parts[1], (parts[2] if len(parts) == 3 else None)
    accounts = load_accounts()
    if any(acc['username'] == username for acc in accounts):
        await update.message.reply_text(f"⚠️ الحساب {username} موجود بالفعل.", reply_markup=account_menu_keyboard())
    else:
        accounts.append({"username": username, "password": password, "proxy": proxy}); save_accounts(accounts)
        await update.message.reply_text(f"✅ تم إضافة الحساب `{username}` بنجاح!", reply_markup=account_menu_keyboard(), parse_mode=ParseMode.MARKDOWN)
    return ACCOUNT_MENU

async def remove_account_select_handler(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    query = update.callback_query; await query.answer()
    accounts = load_accounts()
    if not accounts: await query.edit_message_text("ℹ️ لا يوجد حسابات لحذفها.", reply_markup=account_menu_keyboard()); return ACCOUNT_MENU
    await query.edit_message_text("🗑️ اختر الحساب الذي تريد حذفه:", reply_markup=remove_account_keyboard(accounts)); return REMOVE_ACCOUNT_SELECT

async def handle_remove_account(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    query = update.callback_query; await query.answer()
    username_to_remove = query.data.replace("remove_acc_", "")
    accounts = [acc for acc in load_accounts() if acc['username'] != username_to_remove]
    save_accounts(accounts)
    await query.edit_message_text(f"🗑️ تم حذف الحساب `{username_to_remove}` بنجاح.", reply_markup=account_menu_keyboard(), parse_mode=ParseMode.MARKDOWN); return ACCOUNT_MENU

async def comment_menu_handler(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    query = update.callback_query; await query.answer(); await query.edit_message_text("📝 **إدارة التعليقات**\n\nأضف أو احذف من قائمة التعليقات العشوائية.", reply_markup=comment_menu_keyboard(), parse_mode=ParseMode.MARKDOWN); return COMMENT_MENU

async def add_comment_prompt_handler(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    query = update.callback_query; await query.answer(); text = ("✍️ **أرسل نص التعليق الجديد.**\n\n💡 **نصيحة:** يمكنك استخدام صيغة spintax لجعل التعليقات فريدة:\n`{رائع|جميل|مبدع}`")
    await query.edit_message_text(text, parse_mode=ParseMode.MARKDOWN); return ADD_COMMENT_PROMPT

async def handle_add_comment(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    comments = load_comments(); comments.append(update.message.text.strip()); save_comments(comments)
    await update.message.reply_text("✅ تم إضافة التعليق بنجاح!", reply_markup=comment_menu_keyboard()); return COMMENT_MENU

async def list_comments_handler(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    query = update.callback_query; await query.answer()
    comments = load_comments()
    if not comments: await query.edit_message_text("ℹ️ قائمة التعليقات فارغة.", reply_markup=comment_menu_keyboard()); return COMMENT_MENU
    await query.edit_message_text("🗑️ اختر تعليقاً لحذفه من القائمة:", reply_markup=list_comments_keyboard(comments)); return REMOVE_COMMENT_SELECT

async def handle_remove_comment(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    query = update.callback_query; await query.answer()
    comment_index = int(query.data.replace("remove_comment_", ""))
    comments = load_comments()
    if 0 <= comment_index < len(comments):
        removed_comment = comments.pop(comment_index)
        save_comments(comments)
        await query.answer(f"تم حذف: {removed_comment[:30]}...", show_alert=True)
    
    comments = load_comments() # Reload
    if not comments: await query.edit_message_text("✅ تم حذف آخر تعليق. القائمة فارغة الآن.", reply_markup=comment_menu_keyboard()); return COMMENT_MENU
    await query.edit_message_text("🗑️ اختر تعليقاً آخر لحذفه:", reply_markup=list_comments_keyboard(comments)); return REMOVE_COMMENT_SELECT


# --- (6) دوال سير عمل المهام ---
async def task_menu_handler(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    query = update.callback_query; await query.answer(); await query.edit_message_text("🚀 **بدء مهمة جديدة**\n\nاختر نوع المهمة التي تريد تنفيذها:", reply_markup=task_menu_keyboard(), parse_mode=ParseMode.MARKDOWN); return TASK_MENU

async def select_accounts_start_handler(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    query = update.callback_query; await query.answer()
    context.user_data.update({'task_type': query.data, 'selected_accounts': set()})
    all_accounts = load_accounts()
    if not all_accounts:
        await query.edit_message_text("⚠️ **خطأ:** لا يوجد حسابات مضافة!\nيرجى إضافة حساب أولاً من 'إدارة الحسابات'.", reply_markup=task_menu_keyboard(), parse_mode=ParseMode.MARKDOWN); return TASK_MENU
    await query.edit_message_text("**الخطوة 1 / 3:**\nاختر الحسابات التي ستنفذ المهمة:", reply_markup=select_accounts_keyboard(all_accounts, set())); return SELECT_ACCOUNTS

async def handle_account_selection(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    query = update.callback_query; await query.answer()
    action, all_accounts = query.data, load_accounts()
    selected = context.user_data.get('selected_accounts', set())

    if action.startswith("select_acc_"):
        username = action.replace("select_acc_", "")
        if username in selected: selected.remove(username)
        else: selected.add(username)
    elif action == "select_all": selected = {acc['username'] for acc in all_accounts}
    elif action == "deselect_all": selected.clear()

    context.user_data['selected_accounts'] = selected
    await query.edit_message_text("**الخطوة 1 / 3:**\nاختر الحسابات:", reply_markup=select_accounts_keyboard(all_accounts, selected), parse_mode=ParseMode.MARKDOWN); return SELECT_ACCOUNTS

async def get_link_prompt_handler(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    query = update.callback_query; await query.answer()
    if not context.user_data.get('selected_accounts'):
        await query.answer("⚠️ يرجى اختيار حساب واحد على الأقل!", show_alert=True); return SELECT_ACCOUNTS

    task_type = context.user_data['task_type']
    prompt = "🔗 **الخطوة 2 / 3:**\nالآن، أرسل رابط **حساب** انستغرام المستهدف." if 'follow' in task_type or 'interact' in task_type else "🔗 **الخطوة 2 / 3:**\nالآن، أرسل رابط **منشور** انستغرام المستهدف."
    await query.edit_message_text(prompt, parse_mode=ParseMode.MARKDOWN); return GET_LINK

async def handle_link(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    context.user_data['target_url'] = update.message.text
    task_type = context.user_data['task_type']

    if 'comment' in task_type:
        text = ("💬 **الخطوة 3 / 4:**\nأرسل نص التعليق.\n\n💡 لإدراج تعليق عشوائي من قائمتك، أرسل كلمة `random`.")
        await update.message.reply_text(text, parse_mode=ParseMode.MARKDOWN); return GET_COMMENT
    else:
        text = ("⏱️ **الخطوة 3 / 3 (الأخيرة):**\nأرسل الفاصل الزمني (بالثواني) بين كل عملية وأخرى.\n**مثال:** `15-30`")
        await update.message.reply_text(text, parse_mode=ParseMode.MARKDOWN); return GET_DELAYS

async def handle_comment(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    context.user_data['comment_text'] = update.message.text
    text = ("⏱️ **الخطوة 4 / 4 (الأخيرة):**\nأرسل الفاصل الزمني (بالثواني).\n**مثال:** `20-45`")
    await update.message.reply_text(text, parse_mode=ParseMode.MARKDOWN); return GET_DELAYS

async def handle_delays(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    delays = re.split(r'[,-]', update.message.text.strip())
    if len(delays) != 2 or not delays[0].isdigit() or not delays[1].isdigit():
        await update.message.reply_text("⚠️ صيغة غير صالحة. الرجاء إرسال رقمين مفصولين بـ `-` مثل `15-30`."); return GET_DELAYS
    
    context.user_data['delays'] = (int(delays[0]), int(delays[1]))
    user_data, task_map = context.user_data, {'task_super_interact': '🦸 تفاعل سوبر', 'task_comment': '💬 تعليق على منشور', 'task_like': '❤️ إعجاب بمنشور', 'task_follow': '👤 متابعة حساب'}
    task_name = task_map.get(user_data['task_type'], 'غير معروفة')
    
    summary_text = (f"<b>✨ مراجعة أخيرة للمهمة ✨</b>\n\n"
                    f"<b>- 🎯 نوع المهمة:</b> {task_name}\n"
                    f"<b>- 👥 عدد الحسابات:</b> {len(user_data['selected_accounts'])}\n"
                    f"<b>- 🔗 الرابط:</b> <code>{user_data['target_url']}</code>\n"
                    f"<b>- ⏱️ الفاصل الزمني:</b> بين {user_data['delays'][0]} و {user_data['delays'][1]} ثانية\n\n"
                    f"<b>هل أنت جاهز لبدء التنفيذ؟</b>")
    keyboard = InlineKeyboardMarkup([[InlineKeyboardButton("✅ نعم، ابدأ التنفيذ الآن!", callback_data="execute_task")]])
    await update.message.reply_text(summary_text, reply_markup=keyboard, parse_mode=ParseMode.HTML); return CONFIRM_TASK

# --- (7) محرك تنفيذ المهام (الجوهر) ---
async def run_task_logic(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    query = update.callback_query; await query.answer()
    
    user_data = context.user_data
    task, url, selected_usernames = user_data['task_type'], user_data['target_url'], user_data['selected_accounts']
    comment_input, (min_delay, max_delay) = user_data.get('comment_text'), user_data['delays']
    
    accounts_to_use = [acc for acc in load_accounts() if acc['username'] in selected_usernames]
    total = len(accounts_to_use)
    success, fail = 0, 0
    report_details = []

    await query.edit_message_text(f"🚀 **تم تأكيد المهمة!**\nجارٍ التحضير للبدء على {total} حساب...", parse_mode=ParseMode.MARKDOWN)
    progress_message = await query.message.reply_text("⏳", parse_mode=ParseMode.HTML)

    for i, acc_data in enumerate(accounts_to_use):
        username, password, proxy = acc_data['username'], acc_data['password'], acc_data['proxy']
        
        progress = (i + 1) / total
        progress_bar = "🟩" * int(progress * 12) + "⬜️" * (12 - int(progress * 12))
        status_text = (f"<b>({i+1}/{total}) 👤 العمل بالحساب:</b> <code>{username}</code>\n\n"
                       f"<b>التقدم:</b>\n<code>[{progress_bar}] {int(progress * 100)}%</code>\n\n"
                       f"✅ نجاح: <b>{success}</b> | ❌ فشل: <b>{fail}</b>")
        await context.bot.edit_message_text(chat_id=update.effective_chat.id, message_id=progress_message.message_id, text=status_text, parse_mode=ParseMode.HTML)
        
        cl = Client(); session_file = os.path.join(SESSIONS_DIR, f"{username}.json")
        try:
            if proxy: cl.set_proxy(proxy)
            if os.path.exists(session_file): cl.load_settings(session_file)
            cl.login(username, password)
            cl.dump_settings(session_file)
            
            if task == 'task_super_interact':
                user_id = cl.user_id_from_url(url)
                cl.user_follow(user_id); time.sleep(random.uniform(1.5, 3))
                medias = cl.user_medias(user_id, amount=3)
                if not medias: raise Exception("الحساب لا يحتوي على منشورات.")
                cl.media_like(medias[0].pk); time.sleep(random.uniform(2, 4))
                if random.random() < 0.5 and len(medias) > 1:
                    cl.media_like(medias[1].pk); time.sleep(random.uniform(2, 5))
                comments_list = load_comments()
                if not comments_list: raise Exception("لا توجد تعليقات للمهمة.")
                comment_to_post = process_spintax(random.choice(comments_list))
                cl.media_comment(medias[0].pk, comment_to_post)
            else:
                media_pk = cl.media_pk_from_url(url) if task in ['task_comment', 'task_like'] else None
                if task == 'task_comment':
                    comments_list = load_comments()
                    comment_text = (process_spintax(random.choice(comments_list)) if comment_input.lower() == 'random' and comments_list else comment_input)
                    if not comment_text: raise Exception("لا يوجد تعليق صالح.")
                    cl.media_comment(media_pk, comment_text)
                elif task == 'task_like': cl.media_like(media_pk)
                elif task == 'task_follow': cl.user_follow(cl.user_id_from_url(url))

            success += 1; report_details.append(f"✅ <code>{username}</code>: نجحت المهمة.")
        except ChallengeRequired: fail += 1; report_details.append(f"⚠️ <code>{username}</code>: يتطلب تحقق أمني!")
        except FeedbackRequired: fail += 1; report_details.append(f"🚫 <code>{username}</code>: تم تقييده من انستغرام!")
        except (PrivateError, UserNotFound): fail += 1; report_details.append(f"❌ <code>{username}</code>: حساب خاص/غير موجود.")
        except Exception as e: fail += 1; report_details.append(f"❌ <code>{username}</code>: خطأ - {str(e)[:40]}..."); logger.error(f"Error for {username}: {e}")
        
        finally:
            if i < total - 1:
                sleep_duration = random.uniform(min_delay, max_delay)
                await context.bot.edit_message_text(
                    chat_id=update.effective_chat.id, message_id=progress_message.message_id,
                    text=f"{status_text}\n\n<i>😴 في وضع الانتظار لمدة {int(sleep_duration)} ثانية...</i>",
                    parse_mode=ParseMode.HTML)
                time.sleep(sleep_duration)

    final_report = (f"<b>🏁 اكتملت المهمة! التقرير النهائي:</b>\n\n"
                    f"<b>- ✅ إجمالي النجاح:</b> {success}\n"
                    f"<b>- ❌ إجمالي الفشل:</b> {fail}\n\n"
                    f"<b>📜 تفاصيل العمليات:</b>\n" + "\n".join(report_details))
    await context.bot.edit_message_text(chat_id=update.effective_chat.id, message_id=progress_message.message_id, text=final_report, parse_mode=ParseMode.HTML)
    
    await context.bot.send_message(chat_id=update.effective_chat.id, text="ماذا تريد أن تفعل الآن؟", reply_markup=main_menu_keyboard())
    user_data.clear()
    return ConversationHandler.END

# --- (8) الدالة الرئيسية لتشغيل البوت ---
def main() -> None:
    """الدالة الرئيسية التي تقوم بإعداد وتشغيل البوت."""
    # تأكد من وجود مجلد الجلسات
    if not os.path.exists(SESSIONS_DIR):
        os.makedirs(SESSIONS_DIR)
        
    # بناء تطبيق البوت
    application = Application.builder().token(TELEGRAM_BOT_TOKEN).build()
    
    # إعداد محادثة متكاملة (ConversationHandler)
    conv_handler = ConversationHandler(
        entry_points=[CommandHandler("start", start)],
        states={
            MAIN_MENU: [CallbackQueryHandler(task_menu_handler, pattern="^start_task$"), CallbackQueryHandler(account_menu_handler, pattern="^manage_accounts$"), CallbackQueryHandler(comment_menu_handler, pattern="^manage_comments$")],
            ACCOUNT_MENU: [CallbackQueryHandler(add_account_prompt_handler, pattern="^add_account_prompt$"), CallbackQueryHandler(list_accounts_handler, pattern="^list_accounts$"), CallbackQueryHandler(remove_account_select_handler, pattern="^remove_account_select$"), CallbackQueryHandler(main_menu_handler, pattern="^back_to_main$")],
            ADD_ACCOUNT_PROMPT: [MessageHandler(filters.TEXT & ~filters.COMMAND, handle_add_account)],
            REMOVE_ACCOUNT_SELECT: [CallbackQueryHandler(handle_remove_account, pattern="^remove_acc_"), CallbackQueryHandler(account_menu_handler, pattern="^back_to_account_menu$")],
            COMMENT_MENU: [CallbackQueryHandler(add_comment_prompt_handler, pattern="^add_comment_prompt$"), CallbackQueryHandler(list_comments_handler, pattern="^list_comments$"), CallbackQueryHandler(main_menu_handler, pattern="^back_to_main$")],
            ADD_COMMENT_PROMPT: [MessageHandler(filters.TEXT & ~filters.COMMAND, handle_add_comment)],
            REMOVE_COMMENT_SELECT: [CallbackQueryHandler(handle_remove_comment, pattern="^remove_comment_"), CallbackQueryHandler(comment_menu_handler, pattern="^back_to_comment_menu$")],
            TASK_MENU: [CallbackQueryHandler(select_accounts_start_handler, pattern="^task_"), CallbackQueryHandler(main_menu_handler, pattern="^back_to_main$")],
            SELECT_ACCOUNTS: [CallbackQueryHandler(handle_account_selection, pattern="^select_"), CallbackQueryHandler(get_link_prompt_handler, pattern="^confirm_selection$"), CallbackQueryHandler(task_menu_handler, pattern="^back_to_task_menu$")],
            GET_LINK: [MessageHandler(filters.TEXT & filters.Entity("url"), handle_link)],
            GET_COMMENT: [MessageHandler(filters.TEXT & ~filters.COMMAND, handle_comment)],
            GET_DELAYS: [MessageHandler(filters.Regex(r'^\d{1,3}[,-]\d{1,3}$'), handle_delays)],
            CONFIRM_TASK: [CallbackQueryHandler(run_task_logic, pattern="^execute_task$")],
        },
        fallbacks=[CommandHandler("start", start), CallbackQueryHandler(main_menu_handler, pattern="^back_to_main$")],
        per_message=False # مهم لجعل الأزرار تعمل بشكل صحيح في نفس المحادثة
    )
    
    application.add_handler(conv_handler)
    
    logger.info("🚀 البوت المطور والجميل بدأ بالعمل...")
    application.run_polling()

if __name__ == "__main__":
    main()
