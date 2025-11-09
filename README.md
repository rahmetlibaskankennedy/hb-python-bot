import json
import time
import re
import traceback
import requests
from bs4 import BeautifulSoup

from telegram import (
    Update, InlineKeyboardMarkup,
    InlineKeyboardButton
)
from telegram.ext import (
    ApplicationBuilder,
    CommandHandler,
    CallbackQueryHandler,
    ContextTypes
)

TOKEN = "8231327026:AAHWANk7CjwMu7mfo3o4F8gTTkdmVKaHhDE"
DB = "products.json"
LOGFILE = "lastlog.txt"

ITEMS_PER_PAGE = 10



def log(msg):
    print(msg)
    with open(LOGFILE, "a", encoding="utf-8") as f:
        f.write(msg + "\n")



def load_db():
    try:
        with open(DB, "r", encoding="utf-8") as f:
            return json.load(f)
    except:
        return {"products": []}


def save_db(data):
    with open(DB, "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, indent=2)



def extract_redux_data(html):
    """HTML'den Redux store verisini çıkart"""
    try:
        match = re.search(r'<script type="mime/invalid" id="reduxStore">\s*(\{.+?\})\s*</script>', html, re.DOTALL)
        if match:
            return json.loads(match.group(1))
        return None
    except Exception as e:
        log(f"[REDUX ERROR] {e}")
        return None



def get_campaign_price(url, redux_data):
    """API'den kampanyalı fiyatı çek"""
    try:
        product = redux_data["productState"]["product"]
        
        
        tag_list = []
        for tag in product.get("tagList", []):
            if isinstance(tag, dict):
                tag_list.append(tag.get("tagId", ""))
            else:
                tag_list.append(str(tag))
        
        
        prices = product.get("prices", [])
        if not prices:
            return None, None
        
        
        normal_price = prices[0].get("value")
        
       
        payload = {
            "userId": "",
            "product": {
                "productTags": tag_list,
                "finalPrice": normal_price,
                "sku": product.get("sku", ""),
                "listingId": product.get("listingId", ""),
                "productId": product.get("productId", ""),
                "brand": product.get("brand", ""),
                "merchantId": product.get("merchantId", ""),
                "rootCategoryList": [int(cat["categoryId"]) for cat in product.get("categories", [])],
                "rootBuyingCategoryList": [None],
                "campaignIds": product.get("campaignIds", []),
                "definitionName": product.get("definitionName", ""),
                "definitionId": str(product.get("definitionId", "")),
                "finalPriceOnSale": prices[-1].get("value") if len(prices) > 1 else normal_price,
                "taxVatRate": product.get("taxVatRate", 20)
            }
        }
        
        
        response = requests.post(
            "https://www.hepsiburada.com/api/v1/withoutAffordability",
            json=payload,
            headers={
                "user-agent": "Mozilla/5.0 (Linux; Android 6.0; Nexus 5) AppleWebKit/537.36",
                "content-type": "application/json",
                "origin": "https://www.hepsiburada.com",
                "referer": url
            },
            timeout=10
        )
        
        if response.status_code == 200:
            data = response.json()
            result = data.get("data", {}).get("result", {}).get("product", {})
            
            
            discount_price = None
            
            
            price_data = result.get("priceData", {})
            if price_data.get("status") != "NODATA":
                discount_price = price_data.get("discountedPrice")
            
            
            if not discount_price:
                promo_data = result.get("promoData", {}).get("data", {})
                campaign_result = promo_data.get("campaignEvaluateResult", {})
                
                
                eval_result = campaign_result.get("evaluateResult", {})
                if eval_result and eval_result.get("discountedPrice"):
                    discount_price = eval_result.get("discountedPrice")
                
               
                if not discount_price:
                    premium_result = campaign_result.get("evaluateAsPremiumResult", {})
                    if premium_result and premium_result.get("discountedPrice"):
                        discount_price = premium_result.get("discountedPrice")
            
            log(f"[DISCOUNT FOUND] {discount_price}")
            
            return normal_price, discount_price
        else:
            log(f"[API ERROR] Status: {response.status_code}")
        
        return normal_price, None
        
    except Exception as e:
        log(f"[API ERROR] {e}\n{traceback.format_exc()}")
        return None, None


def scrape_hepsiburada(url):
    log(f"[SCRAPER] URL: {url}")

    headers = {
        "user-agent": "Mozilla/5.0 (Linux; Android 6.0; Nexus 5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/142.0.0.0 Mobile Safari/537.36",
        "accept-language": "tr-TR,tr;q=0.7",
        "referer": "https://www.hepsiburada.com/",
    }

    try:
        r = requests.get(url, headers=headers, timeout=15)
        html = r.text
        soup = BeautifulSoup(html, "html.parser")

        
        title_tag = soup.find("h1", {"data-test-id": "title"})
        title = title_tag.get_text(strip=True) if title_tag else None

        if not title:
            return {"title": None}

        
        redux_data = extract_redux_data(html)
        
        if not redux_data:
            log("[ERROR] Redux data bulunamadı!")
            return {"title": None}
        
        
        normal_price, discount_price = get_campaign_price(url, redux_data)
        
        log(f"[PRICE] Normal: {normal_price}, Discount: {discount_price}")
        
        return {
            "title": title,
            "url": url,
            "normal_price": str(normal_price) if normal_price else None,
            "discount_price": str(discount_price) if discount_price else None,
        }

    except Exception as e:
        log(f"[SCRAPER ERROR] {e}\n{traceback.format_exc()}")
        return {"title": None}



async def ekle(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not context.args:
        return await update.message.reply_text("❗ Kullanım: /ekle <link>")

    url = context.args[0]
    await update.message.reply_text(" Ürün alınıyor...")

    info = scrape_hepsiburada(url)

    if not info.get("title"):
        return await update.message.reply_text("❌ Ürün alınamadı. /debug ile log at.")

    db = load_db()

    if not db["products"]:
        next_id = 1
    else:
        next_id = db["products"][-1]["id"] + 1

    info["id"] = next_id
    db["products"].append(info)
    save_db(db)

   
    price_info = ""
    if info.get("discount_price") and info.get("normal_price"):
        if info["discount_price"] != info["normal_price"]:
            price_info = f"\n Normal: {info['normal_price']} TL\n İndirimli: {info['discount_price']} TL"
        else:
            price_info = f"\n Fiyat: {info['normal_price']} TL"
    elif info.get("normal_price"):
        price_info = f"\n Fiyat: {info['normal_price']} TL"

    await update.message.reply_text(f" #{next_id} eklendi\n📌 {info['title']}{price_info}")



def build_list_page(page):
    db = load_db()
    items = db["products"]
    total = len(items)

    start = page * ITEMS_PER_PAGE
    end = start + ITEMS_PER_PAGE
    page_items = items[start:end]

    text = f" *Takip Edilen Ürünler* (Sayfa {page+1})\n\n"

    for p in page_items:
        price_text = ""
        if p.get("discount_price") and p.get("normal_price"):
            if p["discount_price"] != p["normal_price"]:
                price_text = f"\n {p['normal_price']} TL →  {p['discount_price']} TL"
            else:
                price_text = f"\n {p['normal_price']} TL"
        
        text += f" *{p['id']}*\n {p['title']}{price_text}\n➡️ {p['url']}\n\n"

    buttons = []

    if page > 0:
        buttons.append(InlineKeyboardButton("◀️ Geri", callback_data=f"liste:{page-1}"))

    if end < total:
        buttons.append(InlineKeyboardButton("İleri ▶️", callback_data=f"liste:{page+1}"))

    keyboard = InlineKeyboardMarkup([buttons] if buttons else [])

    return text, keyboard



async def liste(update: Update, context: ContextTypes.DEFAULT_TYPE):
    db = load_db()
    if not db["products"]:
        return await update.message.reply_text(" Ürün yok.")

    text, kb = build_list_page(0)
    await update.message.reply_markdown(text, reply_markup=kb)



async def callback_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()

    data = query.data

    if data.startswith("liste:"):
        page = int(data.split(":")[1])
        text, kb = build_list_page(page)
        await query.edit_message_text(text, parse_mode="Markdown", reply_markup=kb)



async def sil(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not context.args:
        return await update.message.reply_text("❗ Kullanım: /sil <id>")

    try:
        pid = int(context.args[0])
    except:
        return await update.message.reply_text("❗ ID sayı olmalı!")

    db = load_db()
    new_list = [p for p in db["products"] if p["id"] != pid]

    if len(new_list) == len(db["products"]):
        return await update.message.reply_text("❌ Bu ID yok.")

    db["products"] = new_list
    save_db(db)

    await update.message.reply_text(" Silindi.")



async def debug(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_document(open(LOGFILE, "rb"))



def main():
    open(LOGFILE, "w").close()

    import pytz
    from apscheduler.schedulers.asyncio import AsyncIOScheduler

    # Türkiye saat dilimi
    istanbul_tz = pytz.timezone("Europe/Istanbul")

    app = ApplicationBuilder().token(TOKEN).build()

    app.add_handler(CommandHandler("ekle", ekle))
    app.add_handler(CommandHandler("liste", liste))
    app.add_handler(CommandHandler("sil", sil))
    app.add_handler(CommandHandler("debug", debug))

    app.add_handler(CallbackQueryHandler(callback_handler))

    log("BOT BAŞLADI")
    app.run_polling()


if __name__ == "__main__":
    main()# hb-python-bot
