# Monitor and scan high performance stocks in Vietnam market
## Objective:
- Monitor and update VNINDEX, VN-30, HNX-30 daily
- Scan top high performance stocks base on P/E, ROE, EPS
- From top stocks, view detail price chart, valuation measures and financial health
---
- Author: Phạm Duy Tân
- Tools Used: Python/ Google Sheet/ Looker Studio
- Library: Numpy, Pandas, talib, vnstock, yfinance
---

# 📑 Workflow
1. Python: crawl stock indices, prices, and financial data using yfinance and vnstock lib
2. Google Sheet: Export and storage data on Google Sheet
3. Looker Studio: Visualize, refresh data daily
Furthermore, a function can be added to schedule specific hours for automatic execution by setting the condition current_time = hour

---
# 📊 Dashboard
- Looker: https://lookerstudio.google.com/reporting/7309b389-4c39-4054-a0c4-b18bbd7ce4ff
- Notice: on 3rd page, please tick any symbol to view

## Get price VNINDEX, VN-30, HNX-30
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
 ![image alt](https://github.com/tanpham0211/Monitor_and_Scan_stocks_Vietnam_market/blob/main/output%20index.png)

## Retrieve the stocks that recorded the largest market capitalization gains
- By using the prices from D-1 and D-2 and multiplying them by the outstanding shares, we can derive the market capitalization values and calculate the change accordingly
```
from datetime import datetime, timedelta
import pandas as pd
import yfinance as yf
from datetime import datetime, timedelta
vn30 = pd.read_csv('ticker_vn30.csv')           
vn30 = vn30['symbol'].astype(str) + ".VN"  
# thêm ".VN" vào mỗi mã để phù hợp với yfinance


# trong yfinance,lấy giá đóng cửa của ngày start và end
# từ đó tính ra thay đổi điểm và phần trăm thay đổi của cổ phiếu trong khoảng thời gian đó
# check ngày làm việc
# nếu thứ 7, chủ nhật, thứ 2: lấy giá thứ 5& 6
# nếu thứ 3: lấy giá thứ 6 và thứ 2

def get_date(d):
    if d.weekday() == 0:
        d1= d - timedelta(days=4)
        d2= d - timedelta(days=2)
    elif d.weekday() == 1:
        d1= d - timedelta(days=4)
        d2= d - timedelta(days=0)
    elif 1 < d.weekday() <= 5:
        d1= d - timedelta(days=2)
        d2= d - timedelta(days=0)
    elif d.weekday() == 6:
        d1= d - timedelta(days=3)
        d2= d - timedelta(days=1)
    return d1,d2

get_date(datetime.today())
d = get_date(datetime.today())
start = d[0].strftime("%Y-%m-%d")
end = d[1].strftime("%Y-%m-%d")

records = []
for ticker in vn30:
	data = yf.download(ticker, start=start, end=end)
	sharesOutstanding = yf.Ticker(ticker).info.get('sharesOutstanding', 'N/A')
	if data.empty:
		print(f"No data for {ticker} between {start} and {end}.")
		continue
	close_start = data['Close'].iloc[0]
	close_end = data['Close'].iloc[-1]
	sharesOutstanding = sharesOutstanding
	change_points = close_end - close_start
	change_percent = (change_points / close_start)*100
	market_cap_d_2ays_ago = (close_start * sharesOutstanding)/1e9
	maket_cap_now = (close_end * sharesOutstanding)/1e9
	change_market_cap_bil_vnd = maket_cap_now - market_cap_d_2ays_ago
	records.append({
		'Ticker': ticker.replace(".VN", "")
		,'Close End': close_end
		,'Change Points': change_points
		,'Change Percent': change_percent
		,'Market Cap 2 Days Ago (Bil VND)': market_cap_d_2ays_ago
		,'Market Cap Now (Bil VND)': maket_cap_now
		,'Change Market Cap (Bil VND)': change_market_cap_bil_vnd
		})

records= pd.DataFrame(records)
# Chuyển các cột về kiểu float để chỉ hiển thị số
records['Close End'] = records['Close End'].astype(float)
records['Change Points'] = records['Change Points'].astype(float)
records['Change Percent'] = records['Change Percent'].astype(float)
records['Market Cap 2 Days Ago (Bil VND)'] = records['Market Cap 2 Days Ago (Bil VND)'].astype(float)
records['Market Cap Now (Bil VND)'] = records['Market Cap Now (Bil VND)'].astype(float)
records['Change Market Cap (Bil VND)'] = records['Change Market Cap (Bil VND)'].astype(float)
records.sort_values(by='Change Market Cap (Bil VND)', ascending=False, inplace=True)
records.reset_index(drop=True, inplace=True)
records
```
![image alt](https://github.com/tanpham0211/Monitor_and_Scan_stocks_Vietnam_market/blob/main/output%20top%20high%20per%20stock.png)

## Use the price and the 20-day SMA to identify the price trend
```
import yfinance as yf
import pandas as pd
import talib
from datetime import datetime,timedelta

start = ((datetime.today() - timedelta(days=120)).replace(day=1).strftime("%Y-%m-%d"))
end = datetime.today().strftime("%Y-%m-%d")

def get_price(ticker):
    """Get price history for a single ticker, return DataFrame with Ticker and Date columns."""
    hist = yf.download(ticker, start=start, end=end, interval='1d')
    close = hist['Close'].reset_index()
    df = close.melt(var_name='Ticker', id_vars='Date', value_name='Close')
    df = df.sort_values(by=['Ticker', 'Date'])
    df['Ticker'] = df['Ticker'].str.replace('.VN', '')
    df['SMA20'] = talib.SMA(df['Close'], timeperiod=20)
    return df

def get_prices(tickers):
    all_hist = []
    for ticker in tickers:
        get_hist = get_price(ticker)
        all_hist.append(get_hist)
    result = pd.concat(all_hist).sort_values(by=['Ticker','Date']).reset_index(drop=True)
    return result

merged = get_prices(df)
merged
```
![image alt](
https://github.com/tanpham0211/Monitor_and_Scan_stocks_Vietnam_market/blob/main/price%20n%20sma20.png)

