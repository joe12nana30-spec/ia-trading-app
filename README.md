# ia-trading-app
es una aplicación inteligente de asistencia para trading diseñada para ayudar a traders (especialmente principiantes y estudiantes) a analizar el mercado, entender conceptos clave y tomar decisiones más informadas mediante herramientas automatizadas
import MetaTrader5 as mt5
import pandas as pd
import numpy as np
import sqlite3
import requests
import time
import logging
import streamlit as st
from datetime import datetime

# ===== CONFIG =====
SIMBOLO = "EURUSD"
TELEGRAM_TOKEN = "TU_TOKEN_AQUI"
CHAT_ID = "TU_CHAT_ID_AQUI"

logging.basicConfig(filename="bot.log", level=logging.INFO)

# ===== STREAMLIT UI =====
st.title("🤖 Mi IA de Trading Cuantitativo")
st.sidebar.header("Configuración de Riesgo")

activar_bot = st.sidebar.checkbox("Activar Vigilancia 24/5")

# ===== TELEGRAM =====
def enviar_telegram(mensaje):
    url = f"https://api.telegram.org/bot{TELEGRAM_TOKEN}/sendMessage"
    payload = {"chat_id": CHAT_ID, "text": mensaje, "parse_mode": "Markdown"}
    try:
        requests.post(url, json=payload, timeout=5)
    except Exception as e:
        logging.error(f"Telegram error: {e}")

# ===== DB =====
def init_db():
    conn = sqlite3.connect('ia_trading_memory.db')
    cursor = conn.cursor()
    cursor.execute("""
    CREATE TABLE IF NOT EXISTS trades (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        resultado REAL
    )
    """)
    conn.commit()
    conn.close()

def cargar_historial():
    conn = sqlite3.connect('ia_trading_memory.db')
    df = pd.read_sql_query("SELECT * FROM trades", conn)
    conn.close()
    return df

# ===== KELLY =====
def calcular_kelly_dinamico(df):
    if len(df) < 10:
        return 0.01
    win_rate = (df['resultado'] > 0).mean()
    avg_win = df[df['resultado'] > 0]['resultado'].mean()
    avg_loss = abs(df[df['resultado'] <= 0]['resultado'].mean())
    if avg_loss == 0 or np.isnan(avg_loss):
        return 0.01
    kelly = win_rate - (1 - win_rate) / (avg_win / avg_loss)
    return max(min(kelly, 0.02), 0.005)

# ===== STRESS TEST =====
def stress_test(equity):
    shocks = np.random.normal(-0.01, 0.02, 1000)
    equity_sim = equity * np.cumprod(1 + shocks)
    return equity_sim.min()

# ===== DATOS =====
def obtener_datos():
    rates = mt5.copy_rates_from_pos(SIMBOLO, mt5.TIMEFRAME_M5, 0, 200)
    df = pd.DataFrame(rates)
    df['EMA50'] = df['close'].ewm(span=50).mean()
    df['EMA200'] = df['close'].ewm(span=200).mean()
    df['Signal'] = np.where(df['EMA50'] > df['EMA200'], 1, -1)
    return df

# ===== ORDEN =====
def enviar_orden(signal, volumen):
    tick = mt5.symbol_info_tick(SIMBOLO)
    price = tick.ask if signal == 1 else tick.bid

    request = {
        "action": mt5.TRADE_ACTION_DEAL,
        "symbol": SIMBOLO,
        "volume": volumen,
        "type": mt5.ORDER_TYPE_BUY if signal == 1 else mt5.ORDER_TYPE_SELL,
        "price": price,
        "deviation": 20,
        "magic": 123456,
        "comment": "IA Trading",
        "type_time": mt5.ORDER_TIME_GTC,
        "type_filling": mt5.ORDER_FILLING_IOC,
    }
    return mt5.order_send(request)

# ===== MOTOR =====
def ejecutar_bot():
    df = obtener_datos()
    trade_df = cargar_historial()

    riesgo_frac = calcular_kelly_dinamico(trade_df)

    account = mt5.account_info()
    equity = account.equity

    equity_estres = stress_test(equity)

    if equity_estres < equity * 0.7:
        enviar_telegram("⚠️ Stress test bloqueó operación")
        return

    riesgo_usd = equity * riesgo_frac

    distancia = 0.0015
    volumen = riesgo_usd / distancia
    volumen = round(volumen / 100000, 2)

    signal = df['Signal'].iloc[-1]

    result = enviar_orden(signal, volumen)

    if result.retcode == mt5.TRADE_RETCODE_DONE:
        enviar_telegram(f"📈 Trade ejecutado | Volumen {volumen}")
        logging.info("Trade ejecutado")
    else:
        logging.error("Fallo en orden")

# ===== DASHBOARD =====
def mostrar_dashboard():
    df = cargar_historial()
    st.subheader("Curva de Equidad")

    if not df.empty:
        equity_curve = df['resultado'].cumsum()
        st.line_chart(equity_curve)
    else:
        st.write("Aún no hay trades registrados.")

# ===== INIT =====
init_db()

if not mt5.initialize():
    st.error("Error conectando MT5")
else:
    st.success("Conectado a MT5")

mostrar_dashboard()

# ===== BOT MANUAL =====
if st.button("▶ Ejecutar bot ahora"):
    ejecutar_bot()
    st.success("Bot ejecutado")

# ===== VIGILANCIA CONTINUA =====
placeholder = st.empty()

if activar_bot:
    placeholder.info("🟢 Vigilancia activa")

    ejecutar_bot()
    placeholder.write(f"Última revisión: {datetime.now().strftime('%H:%M:%S')}")

    time.sleep(60)
    st.experimental_rerun()
