import requests
from bs4 import BeautifulSoup
import telebot
from telebot.apihelper import ApiTelegramException
import time
import re
from datetime import datetime

# --- CONFIG ---
BOT_TOKEN = "6050076233:AAHTfZpIWvzuQfZWA20uBSqYQpzvI1ozoyU"
# Added your new Chat ID below
CHAT_IDS = ["-1001919117846", "-1001539965485"] 
CHANNEL_LINK = "https://t.me/SattaMatkaResultslive"

bot = telebot.TeleBot(BOT_TOKEN)
hindi_pattern = re.compile(r"[ऀ-ॿ]+")

# --- TYPOGRAPHY ENGINE ---
def bs(text):
    """Bold Sans for Numbers"""
    r = ""
    for c in text:
        if "A" <= c <= "Z": r += chr(ord(c)-65+0x1D5D4)
        elif "a" <= c <= "z": r += chr(ord(c)-97+0x1D5EE)
        elif "0" <= c <= "9": r += chr(ord(c)-48+0x1D7EC)
        else: r += c
    return r

def sc(text):
    """Small Caps for Names"""
    m = {'a':'ᴀ','b':'ʙ','c':'ᴄ','d':'ᴅ','e':'ᴇ','f':'ꜰ','g':'ɢ','h':'ʜ','i':'ɪ','j':'ᴊ','k':'ᴋ','l':'ʟ','m':'ᴍ','n':'ɴ','o':'ᴏ','p':'ᴘ','q':'ǫ','r':'ʀ','s':'s','t':'ᴛ','u':'ᴜ','v':'ᴠ','w':'ᴡ','x':'x','y':'ʏ','z':'ᴢ'}
    return "".join(m.get(c.lower(), c) for c in text)

def tiny(text):
    """Superscript for Tags"""
    m = {'a':'ᴬ','b':'ᴮ','c':'ᶜ','d':'ᴰ','e':'ᴱ','f':'ᶠ','g':'ᴳ','h':'ᴴ','i':'ᴵ','j':'ᴶ','k':'ᴷ','l':'ᴸ','m':'ᴹ','n':'ᴺ','o':'ᴼ','p':'ᴾ','q':'ᵠ','r':'ᴿ','s':'ˢ','t':'ᵀ','u':'ᵁ','v':'ⱽ','w':'ᵂ','x':'ˣ','y':'ʸ','z':'ᴢ'}
    return "".join(m.get(c.lower(), c) for c in text)

# --- MARKET TIMES ---
MARKET_TIMES = {
    "SRIDEVI": "11:30", "TIME BAZAR": "13:00", "MADHUR DAY": "13:15",
    "MILAN DAY": "15:00", "RAJDHANI DAY": "15:00", "KALYAN": "15:45",
    "SUPREME DAY": "15:35", "SRIDEVI NIGHT": "19:00", "MADHUR NIGHT": "20:30",
    "SUPREME NIGHT": "20:45", "MILAN NIGHT": "21:00", "RAJDHANI NIGHT": "21:00",
    "KALYAN NIGHT": "21:25", "MAIN BAZAR": "21:40"
}

baseline_results = {}
baseline_loaded = False
last_message_ids = {}

def clean_res(raw):
    r = hindi_pattern.sub("", raw).strip()
    match = re.search(r"(d{3}-d{1,2}-d{3})|(d{3}-d{1,2})", r)
    return match.group(0) if match else ""

def scrape():
    res = {}
    try:
        resp = requests.get("https://sattamatkano1.me/", timeout=10)
        soup = BeautifulSoup(resp.text, "html.parser")
        for frame in soup.select("div.frame"):
            try:
                name_tag = frame.find("p", class_="fontframe")
                res_tag = frame.find("p", class_="black")
                if name_tag and res_tag:
                    name = name_tag.text.strip()
                    if "SUPRRME" in name: name = "SUPREME DAY" # Fix Site Typo
                    if name in MARKET_TIMES:
                        res[name] = clean_res(res_tag.text)
            except: continue
    except: pass
    return res

def build_dashboard(data):
    now = datetime.now()
    cur_time = now.strftime("%H:%M")
    
    msg = f"<b>┏━━━━━━━━━━━━━━━━━━┓</b>
"
    msg += f"<b>   💎 {sc('satta live monitor')} 💎</b>
"
    msg += f"<b>┗━━━━━━━━━━━━━━━━━━┛</b>
"
    msg += f"📅 <b>{sc(now.strftime('%A %d %b'))}</b>
"
    msg += f"⏰ <b>{sc('last update')}: {now.strftime('%I:%M:%S %p')}</b>
"
    msg += f"<b>──────────────────</b>
"

    for mn, start_t in MARKET_TIMES.items():
        val = data.get(mn, "")
        base = baseline_results.get(mn, "")
        
        parts = val.split('-')
        is_partial = (len(parts) == 2 and parts[1] != "")
        is_new_full = (len(parts) == 3 and val != base)
        started = cur_time >= start_t
        m_name = bs(mn)

        if is_partial:
            msg += f"<b>┏━ {m_name} {tiny('live')} ━┓</b>
"
            msg += f"<b>┃ 🟡 {bs(val)} ⏳</b>
"
            msg += f"<b>┃ {tiny('waiting for close')}...</b>
"
            msg += f"<b>┗━━━━━━━━━━━━━━━━━━┛</b>
"

        elif is_new_full:
            msg += f"<b>┏━ {m_name} {tiny('full')} ━┓</b>
"
            msg += f"<b>┃ 🟢 {bs(val)} 🔥</b>
"
            msg += f"<b>┗━━━━━━━━━━━━━━━━━━┛</b>
"

        elif started and not val:
            msg += f"<b>┏━ {m_name} ━┓</b>
"
            msg += f"<b>┃ ⏳ {sc('today soon')}...</b>
"
            msg += f"<b>┃ <i>{tiny('checking live')}</i></b>
"
            msg += f"<b>┗━━━━━━━━━━━━━━━━━━┛</b>
"

        else:
            d_val = bs(val) if val else "ᴡᴀɪᴛɪɴɢ..."
            # Anti-Wrapping Alignment for previous results
            msg += f"⚪ <b>{sc(mn)}</b>
"
            msg += f"   ➬ <code>{d_val}</code> {tiny('prev')}
"
            msg += f"<b>┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈</b>
"

    msg += f"
<b>🚀 {sc('powered by professional engine')}</b>
"
    msg += f"<b>🔗 <a href='{CHANNEL_LINK}'>{sc('join official channel')}</a></b>"
    return msg

def push_update(text):
    for cid in CHAT_IDS:
        try:
            mid = last_message_ids.get(cid)
            if mid:
                bot.edit_message_text(text, cid, mid, parse_mode="HTML", disable_web_page_preview=True)
            else:
                m = bot.send_message(cid, text, parse_mode="HTML", disable_web_page_preview=True)
                last_message_ids[cid] = m.message_id
        except ApiTelegramException as e:
            if "message is not modified" in str(e): continue
            try:
                m = bot.send_message(cid, text, parse_mode="HTML")
                last_message_ids[cid] = m.message_id
            except Exception as e_inner:
                print(f"Error sending to {cid}: {e_inner}")
        time.sleep(0.2)

# --- EXECUTION ---
print("💎 Dual-Channel Dashboard Running...")
while True:
    try:
        current_data = scrape()
        if not baseline_loaded and current_data:
            for k, v in current_data.items():
                if len(v.split('-')) == 3: baseline_results[k] = v
            baseline_loaded = True
        
        dashboard = build_dashboard(current_data)
        push_update(dashboard)
    except Exception as e:
        print(f"Global Error: {e}")
    time.sleep(2)