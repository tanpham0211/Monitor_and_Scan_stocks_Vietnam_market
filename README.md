### Monitor and scan high performance stocks in Vietnam market
## Objective:
_+ Monitor and update VNINDEX, VN-30, HNX-30 daily

_+ Scan top high performance stocks base on P/E, ROE, EPS
  
_+ From top stocks, view detail price chart, valuation measures and financial health

- Author: Phạm Duy Tân
- Tools Used: Python/ Google Sheet/ Looker Studio  
---

## 📑 Workflow
1. Python: crawl stock indices, prices, and financial data using yfinance and vnstock lib
2. Google Sheet: Export and storage data on Google Sheet
3. Looker Studio: Visualize, refresh data daily

---
### 📊 Dashboard
- Looker: https://lookerstudio.google.com/reporting/7309b389-4c39-4054-a0c4-b18bbd7ce4ff
- Notice: on 3rd page, please tick any symbol to view

# Get price VNINDEX, VN-30, HNX-30
- Use vnstock library to get indexes of Vietnam stock market
```
from vnstock import Quote
import pandas as pd
from datetime import timedelta,time

symbols = ['VNINDEX', 'VN30', 'HNX30']
data_frames = []

for sym in symbols:
    quote = Quote(symbol=sym, source='VCI')
    df = quote.history(length='1', interval='d').reset_index()  # reset index để có cột time thay vì index
    # chèn cột mới ở vị trí 0 (đầu DataFrame)
    df.insert(0, 'symbol', sym)
    # df = df.df[['symbol', 'time', 'close', 'volume']]
    # trong df, lấy ra các cột cần thiết: symbol, time, close, volume
    df = df[['symbol', 'time', 'close', 'volume']]
    data_frames.append(df)
    
all_data = pd.concat(data_frames, ignore_index=True)

# trong từng symsbol, lấy giá của ngày d-1, gán vào ngày d-2, tính chênh lệch điểm và phần trăm thay đổi
all_data['prev_close'] = all_data.groupby('symbol')['close'].shift(1)
all_data['change'] = all_data['close'] - all_data['prev_close']
all_data['change_pct'] = (all_data['change'] / all_data['prev_close'] * 100).round(2)	

today = pd.Timestamp.now().date().strftime('%Y-%m-%d')
all_data_today = all_data[all_data['time'] == today]

all_data_today.reset_index()
# all_data.to_csv('index_data.csv', index=False)
```
 ![image alt](https://github.com/tanpham0211/Monitor_and_Scan_stocks_Vietnam_market/blob/main/get%20index%20price%20.png)
