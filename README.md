# 📈 CryptoPulse — ML-Powered Crypto Trading Signal Bot

> A real-time cryptocurrency trading signal bot that uses **XGBoost** and a rich suite of **technical indicators** to generate LONG / SHORT / HOLD predictions, track their accuracy over time, and surface everything through a live **Flask web dashboard**.

---

## ✨ Features

- **Multi-coin support** — Simultaneously track BTC, ETH, BNB, SOL, XRP, ADA, DOGE, AVAX, TRX, and DOT via the Binance API
- **XGBoost ML model** — Trained on historical OHLCV data with 36 engineered features including RSI, MACD, Bollinger Bands, ATR, Stochastic Oscillator, Williams %R, OBV, ADX, and CCI
- **Multi-timeframe signals** — Predictions for 15-minute, 1-hour, 4-hour, and daily intervals with adaptive horizons
- **Incremental feature calculation** — Real-time feature updates without re-running full historical computation on every tick
- **Prediction tracker** — Persists every signal to CSV and automatically verifies accuracy once the prediction horizon elapses
- **Position tracker** — Monitors simulated open/closed positions, PnL, win rate, and profit factor
- **Auto-retraining** — Models retrain automatically every 7 days (configurable) or on demand via the dashboard API
- **Live Flask dashboard** — Single-page UI served from `templates/index.html` with a REST API powering all widgets

---

## 🏗️ Architecture

```
tradingBot/
├── main.py                  # Entry point — starts the Flask server
├── app.py                   # Flask app & REST API endpoints
├── predictor.py             # ProductionRealTimePredictor — orchestrates inference
├── features.py              # Technical indicator computation & IncrementalFeatureCalculator
├── models.py                # XGBoost training, evaluation, and persistence
├── binance_api.py           # Binance REST API wrapper (OHLCV, ticker, price)
├── tracker.py               # PredictionTracker — logs and verifies signals
├── positon_tracker.py       # PositionTracker — simulated trade P&L
├── config.py                # All constants, paths, and trading configuration
├── templates/
│   └── index.html           # Dashboard frontend (HTML/JS/CSS)
├── models/                  # Saved XGBoost models, scalers & metrics (per ticker)
├── prediction_history/      # Per-ticker prediction CSV logs
├── logs/                    # Application logs
└── history/                 # Trade history
```

---

## 🚀 Getting Started

### Prerequisites

- Python 3.9+
- A [Binance](https://www.binance.com) account with API access (read-only keys are sufficient for signal generation)

### 1. Clone the repository

```bash
git clone https://github.com/your-username/cryptopulse.git
cd cryptopulse
```

### 2. Install dependencies

```bash
pip install -r requirements.txt
```

### 3. Configure your API keys

Create a `.env` file in the project root:

```env
BINANCE_API_KEY=your_api_key_here
BINANCE_API_SECRET=your_api_secret_here
```

> **Never commit your `.env` file.** It is already listed in `.gitignore`.

### 4. Run the bot

```bash
python main.py
```

The dashboard will be available at `http://localhost:5000`.

---

## ⚙️ Configuration

All tunable parameters live in `config.py`:

| Parameter | Default | Description |
|---|---|---|
| `TOP_COINS` | 10 major coins | List of Binance trading pairs to support |
| `AUTO_REFRESH_INTERVAL` | `5` seconds | Dashboard polling interval |
| `MODEL_RETRAIN_DAYS` | `7` | Days before a model is automatically retrained |
| `TEST_SIZE_RATIO` | `0.2` | Fraction of data held out for model evaluation |
| `PREDICTION_HORIZONS` | `{15m:4, 1h:6, 4h:6, 1d:7}` | Prediction look-ahead periods per interval |
| `POSITION_CONFIG` | see below | Initial capital, position size, stop-loss, take-profit |

Default position config:

```python
POSITION_CONFIG = {
    'initial_capital': 10000,
    'position_size_pct': 0.10,   # 10% of capital per trade
    'stop_loss_pct': 0.02,        # 2% stop loss
    'take_profit_pct': 0.03,      # 3% take profit
    'trade_fee_pct': 0.001,       # 0.1% fee
}
```

---

## 🔌 REST API

The Flask server exposes the following endpoints:

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/` | Serves the live dashboard UI |
| `GET` | `/api/prediction?ticker=BTCUSDT&interval=1d` | Get current signal, confidence, price, and metrics |
| `GET` | `/api/position-info?ticker=BTCUSDT` | Get open position status and performance summary |
| `GET` | `/api/verify-now?ticker=BTCUSDT` | Manually trigger prediction verification |
| `GET` | `/api/stats` | List all active predictors and system status |
| `POST` | `/api/retrain` | Trigger an on-demand model retrain `{"ticker": "BTCUSDT"}` |

### Sample prediction response

```json
{
  "signal": "LONG",
  "confidence": 0.71,
  "price": 67420.50,
  "horizon_description": "7 days",
  "price_change": 2.34,
  "high_price": 68100.00,
  "low_price": 66800.00,
  "volume": 23400.0,
  "metrics": {
    "accuracy": 0.61,
    "f1_score": 0.58,
    "auc": 0.64
  },
  "tracker": {
    "accuracy": 0.63,
    "verified_total": 142,
    "long_accuracy": 0.67,
    "short_accuracy": 0.58
  }
}
```

---

## 📊 Technical Indicators

The model is trained on **36 features** across several indicator families:

| Category | Indicators |
|---|---|
| **Trend** | SMA (10/20/50/100/200), Price Momentum (5/10/20) |
| **Momentum** | RSI-14, Stochastic %K/%D, Williams %R, CCI, MACD / Signal / Histogram |
| **Volatility** | Bollinger Band Position & Width, ATR-14 & Ratio, Rolling Volatility (10/20/50) |
| **Volume** | Volume Ratio, OBV, OBV Signal |
| **Trend Strength** | ADX, DI+, DI− |
| **Lag Features** | Close Lag 1, 5, 10 |

---

## 🧠 Model Details

- **Algorithm:** XGBoost (`binary:logistic`)
- **Target:** Directional movement — `1` if price moves up beyond threshold within the horizon, `0` otherwise
- **Thresholds by interval:** 15m → 0.2%, 1h → 0.5%, 4h → 1%, 1d → 2%
- **Class balancing:** Sample weights applied to correct for class imbalance
- **Preprocessing:** `RobustScaler` (resistant to outliers in financial data)
- **Persistence:** Models and scalers saved as `.joblib` files; metrics as `.json`

Signal thresholds at inference time are **adaptive** — they widen with recent market volatility using a simple `LONG_THRESH = 0.55 + (volatility * 0.5)` formula.

---

## 📁 Supported Coins (Pre-trained Models Included)

Pre-trained XGBoost models are bundled for the following pairs:

- `BTCUSDT`
- `ETHUSDT`
- `BNBUSDT`
- `SOLUSDT`
- `XRPUSDT`

For any other coin in `TOP_COINS`, the model trains automatically on first run.

---

## ⚠️ Disclaimer

This project is for **educational and research purposes only**. Cryptocurrency trading carries significant financial risk. The signals generated by this bot are probabilistic and should never be interpreted as financial advice. Always conduct your own research and risk management before making any trading decisions.

---

## 🛠️ Tech Stack

- **Python 3.9+**
- **Flask** — Web server & REST API
- **XGBoost** — Core ML model
- **scikit-learn** — Preprocessing and evaluation
- **pandas / numpy** — Data manipulation
- **joblib** — Model persistence
- **Binance REST API** — Market data

---

## 📄 License

MIT License — see [LICENSE](LICENSE) for details.
