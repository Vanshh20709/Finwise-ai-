
from fastapi import FastAPI
from fastapi.staticfiles import StaticFiles
from fastapi.responses import FileResponse
from sklearn.linear_model import LinearRegression

app = FastAPI()

stocks = {
    "RELIANCE": [1400, 1420, 1410, 1440, 1450],
    "TCS": [3900, 3920, 3950, 3980, 4000],
    "INFY": [1770, 1790, 1810, 1800, 1840]
}


@app.get("/")
def home():
    return FileResponse("static/index.html")


@app.get("/analyze/{asset}")
def analyze(asset: str):

    asset = asset.upper()

    if asset not in stocks:
        return {"error": "Stock not found"}

    prices = stocks[asset]

    days = [[1], [2], [3], [4], [5]]

    model = LinearRegression()
    model.fit(days, prices)

    prediction = model.predict([[6]])[0]

    if prediction > prices[-1]:
        trend = "UP"
        insight = (
            "The model predicts a possible price increase. "
            "Research the company and market before investing."
        )
    elif prediction < prices[-1]:
        trend = "DOWN"
        insight = (
            "The model predicts a possible price decrease. "
            "Be cautious and research before investing."
        )
    else:
        trend = "STABLE"
        insight = (
            "The model predicts a stable price. "
            "Study the market before making a decision."
        )

    return {
        "asset": asset,
        "current_price": prices[-1],
        "predicted_price": round(prediction, 2),
        "trend": trend,
        "insight": insight,
        "prices": prices
    }


app.mount(
    "/static",
    StaticFiles(directory="static"),
    name="static"
)
