from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import pandas as pd
import sqlite3
from sklearn.linear_model import LinearRegression

app = FastAPI(
    title="FinWise AI",
    description="Financial Market and Cryptocurrency Intelligence API",
    version="1.0"
)

DB = "finwise_new.db"


# -- STOCK DATA --

stocks = {
    "asset": ["RELIANCE"] * 5 + ["TCS"] * 5 + ["INFY"] * 5,
    "close": [
        1395, 1410, 1425, 1438, 1450,
        3880, 3910, 3945, 3970, 4005,
        1770, 1785, 1805, 1825, 1845
    ],
    "volume": [
        1200000, 1250000, 1300000, 1350000, 1400000,
        700000, 730000, 760000, 790000, 820000,
        900000, 930000, 960000, 990000, 1020000
    ]
}


# - CRYPTO DATA -

crypto = {
    "asset": ["BTC"] * 5 + ["ETH"] * 5 + ["SOL"] * 5,
    "close": [
        91000, 92000, 93200, 94500, 95800,
        3050, 3100, 3160, 3220, 3290,
        185, 190, 196, 202, 208
    ],
    "volume": [
        30000000000, 31000000000, 32000000000,
        33000000000, 34000000000,
        15000000000, 15500000000, 16000000000,
        16500000000, 17000000000,
        3000000000, 3100000000, 3200000000,
        3300000000, 3400000000
    ]
}


# - DATABASE -

def connect_db():
    return sqlite3.connect(DB)


db = connect_db()

db.execute("""
CREATE TABLE IF NOT EXISTS analyses (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    asset TEXT,
    market_type TEXT,
    current_price REAL,
    predicted_price REAL,
    trend TEXT,
    volume REAL,
    ai_analysis TEXT
)
""")

db.commit()
db.close()



class MarketData(BaseModel):
    asset: str
    market_type: str


# - GET DATA -

def get_data(market):

    if market == "stock":
        return pd.DataFrame(stocks)

    if market == "crypto":
        return pd.DataFrame(crypto)

    raise HTTPException(
        status_code=400,
        detail="Market type must be stock or crypto"
    )


# - PREDICTION -

def predict_price(prices):

    days = [[i] for i in range(1, len(prices) + 1)]

    model = LinearRegression()
    model.fit(days, prices)

    prediction = float(
        model.predict([[len(prices) + 1]])[0]
    )

    if prediction >= prices[-1]:
        trend = "Up"
    else:
        trend = "Down"

    return round(prediction, 2), trend


# - AI ANALYSIS -

def make_analysis(asset, current, predicted, volume, trend):

    change = round(predicted - current, 2)

    if trend == "Up":
        message = "The market may move upward."
    else:
        message = "The market may move downward."

    return (
        f"{asset}: {message} "
        f"Expected price change is {change}. "
        f"Latest volume is {volume}."
    )


# - HOME -

@app.get("/")
def home():

    return {
        "message": "Welcome to FinWise AI",
        "project": "Financial Market and Cryptocurrency Intelligence"
    }


# - ANALYZE -

@app.post("/analyze")
def analyze(data: MarketData):

    asset = data.asset.upper()
    market = data.market_type.lower()

    df = get_data(market)

    result = df[df["asset"] == asset]

    if result.empty:
        raise HTTPException(
            status_code=404,
            detail="Asset not found"
        )

    prices = [float(x) for x in result["close"].tolist()]

    current = float(prices[-1])
    volume = float(result["volume"].iloc[-1])

    predicted, trend = predict_price(prices)

    analysis = make_analysis(
        asset, current, predicted, volume, trend
    )

    db = connect_db()
    cursor = db.cursor()

    cursor.execute("""
        INSERT INTO analyses
        (asset, market_type, current_price,
         predicted_price, trend, volume, ai_analysis)
        VALUES (?, ?, ?, ?, ?, ?, ?)
    """, (
        asset,
        market,
        current,
        predicted,
        trend,
        volume,
        analysis
    ))

    db.commit()
    analysis_id = int(cursor.lastrowid)
    db.close()

    return {
        "analysis_id": analysis_id,
        "asset": asset,
        "market_type": market,
        "current_price": current,
        "predicted_price": predicted,
        "trend": trend,
        "volume": volume,
        "ai_market_analysis": analysis
    }


# -- ALL ANALYSES --

@app.get("/analyses")
def all_analyses():

    db = connect_db()
    rows = db.execute("SELECT * FROM analyses").fetchall()
    db.close()

    return [
        {
            "analysis_id": int(r[0]),
            "asset": r[1],
            "market_type": r[2],
            "current_price": float(r[3]),
            "predicted_price": float(r[4]),
            "trend": r[5],
            "volume": float(r[6]),
            "ai_market_analysis": r[7]
        }
        for r in rows
    ]

# -- ONE ANALYSIS --

@app.get("/analyses/{analysis_id}")
def one_analysis(analysis_id: int):

    db = connect_db()

    row = db.execute(
        "SELECT * FROM analyses WHERE id = ?",
        (analysis_id,)
    ).fetchone()

    db.close()

    if row is None:
        raise HTTPException(404, "Analysis not found")

    return {
        "analysis_id": int(row[0]),
        "asset": row[1],
        "market_type": row[2],
        "current_price": float(row[3]),
        "predicted_price": float(row[4]),
        "trend": row[5],
        "volume": float(row[6]),
        "ai_market_analysis": row[7]
    }


# -- DELETE --

@app.delete("/analyses/{analysis_id}")
def delete_analysis(analysis_id: int):

    db = connect_db()
    cursor = db.cursor()

    cursor.execute(
        "DELETE FROM analyses WHERE id = ?",
        (analysis_id,)
    )

    db.commit()

    if cursor.rowcount == 0:
        db.close()
        raise HTTPException(404, "Analysis not found")

    db.close()

    return {"message": "Analysis deleted successfully"
}
