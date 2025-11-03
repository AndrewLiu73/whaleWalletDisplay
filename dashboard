import streamlit as st
import asyncio
import aiohttp
from motor.motor_asyncio import AsyncIOMotorClient
from collections import Counter
import os
from dotenv import load_dotenv
import datetime

load_dotenv()  # Loads variables from .env into environment

MONGO_URI = os.getenv("MONGO_URI")
API_URL = os.getenv("API_URL")

DB_NAME, COLL = "hyperliquid", "millionaires"
COINS = ["BTC", "ETH", "HYPE"]
RETRY, PARALLEL = 3, 10

def format_time():
    return datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S')

async def fetch_wallets():
    try:
        cli = AsyncIOMotorClient(MONGO_URI)
        docs = await cli[DB_NAME][COLL].find({}, {"_id": 0, "wallet": 1}).to_list(None)
        wallets = [d["wallet"] for d in docs if "wallet" in d]
        return wallets
    except Exception as e:
        st.error(f"MongoDB connection failed: {e}")
        return []

async def fetch_position(session, wallet):
    for attempt in range(RETRY):
        try:
            async with session.post(API_URL, json={"type": "clearinghouseState", "user": wallet}) as resp:
                if resp.status == 200:
                    js = await resp.json()
                    return wallet, js.get("assetPositions", [])
        except Exception as e:
            pass
        await asyncio.sleep(2**attempt)
    return wallet, []

async def process_wallets(wallets):
    wallet_bias = {}
    agg_val = {c: Counter() for c in COINS}
    async with aiohttp.ClientSession() as session:
        sema = asyncio.Semaphore(PARALLEL)
        async def worker(wallet):
            async with sema:
                w, positions = await fetch_position(session, wallet)
                per_val = {c: Counter() for c in COINS}
                for pos in positions:
                    coin = pos.get("position", {}).get("coin")
                    szi = float(pos.get("position", {}).get("szi", 0))
                    val = float(pos.get("position", {}).get("positionValue", 0))
                    if coin in COINS and szi and val:
                        side = "B" if szi > 0 else "A"
                        agg_val[coin][side] += val
                        per_val[coin][side] += val
                wallet_bias[wallet] = {}
                for coin in COINS:
                    long_val = per_val[coin].get("B", 0.0)
                    short_val = per_val[coin].get("A", 0.0)
                    total = long_val + short_val
                    l_pct = (long_val/total*100) if total else 0
                    s_pct = (short_val/total*100) if total else 0
                    direction = "Long" if long_val > short_val else "Short" if short_val > long_val else "Neutral"
                    wallet_bias[wallet][coin] = {
                        "long": long_val, "short": short_val,
                        "long_pct": l_pct, "short_pct": s_pct,
                        "direction": direction
                    }
        await asyncio.gather(*[worker(w) for w in wallets])
    # Aggregate stats
    agg_bias = {}
    for coin in COINS:
        long_val = agg_val[coin]["B"]
        short_val = agg_val[coin]["A"]
        total = long_val + short_val
        l_pct = (long_val/total*100) if total else 0
        s_pct = (short_val/total*100) if total else 0
        dirn = "Long" if long_val > short_val else "Short" if short_val > long_val else "Neutral"
        agg_bias[coin] = {
            "long": long_val, "short": short_val,
            "long_pct": l_pct, "short_pct": s_pct,
            "direction": dirn
        }
    return wallet_bias, agg_bias

# --- Streamlit UI ---
st.set_page_config(page_title='Millionaire Bias Tracker', layout='wide')
st.title("Millionaire Bias Tracker")

btn = st.button("Refresh Bias Now")
if btn:
    with st.spinner("Updating wallets and positions, please wait..."):
        wallets = asyncio.run(fetch_wallets())
        if wallets:
            wallet_bias, agg_bias = asyncio.run(process_wallets(wallets))
            st.success(f"Updated at {format_time()}! {len(wallets)} wallets loaded.")

            st.header("Aggregate Bias")
            for coin, stats in agg_bias.items():
                st.subheader(coin)
                st.write(f"{stats['direction']}: Long ${stats['long']:.2f} ({stats['long_pct']:.1f}%) | Short ${stats['short']:.2f} ({stats['short_pct']:.1f}%)")
                st.progress(stats['long_pct']/100)

            st.header("Individual Wallet Bias")
            subset = st.multiselect("Show specific wallets:", wallets)
            to_show = subset if subset else wallets[:min(10, len(wallets))]
            for w in to_show:
                coins = wallet_bias[w]
                st.markdown(f"**{w}**")
                for c, s in coins.items():
                    st.write(f"{c}: {s['direction']} | Long ${s['long']:.2f} ({s['long_pct']:.1f}%) | Short ${s['short']:.2f} ({s['short_pct']:.1f}%)")
                st.markdown('---')

        else:
            st.error("No wallets found or MongoDB error!")
else:
    st.info("Click 'Refresh Bias Now' to load stats.")
