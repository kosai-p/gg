# H4-glod
@ -0,0 +1,54 @@
import pandas as pd
import numpy as np
import requests
import talib

# ดึงข้อมูลราคาทองคำจาก API
api_url = 'https://api.example.com/gold-prices'
response = requests.get(api_url)
data = response.json()

# แปลงข้อมูลให้เป็น DataFrame
df = pd.DataFrame(data)
df['date'] = pd.to_datetime(df['date'])
df.set_index('date', inplace=True)

# คำนวณค่าเฉลี่ยเคลื่อนที่ (MA)
df['MA12'] = df['price'].rolling(window=12).mean()
df['MA26'] = df['price'].rolling(window=26).mean()

# คำนวณ RSI และ ATR
df['RSI'] = talib.RSI(df['price'], timeperiod=14)
df['ATR'] = talib.ATR(df['high'], df['low'], df['close'], timeperiod=14)

# ฟังก์ชั่นส่งการแจ้งเตือนผ่าน LINE
def send_line_notify(message):
    url = 'https://notify-api.line.me/api/notify'
    token = 'ZOV8t600O0p3eftqyA6DYKJA3FJKelEGLALngOpLoZE'
    headers = {'Authorization': 'Bearer ' + token}
    data = {'message': message}
    requests.post(url, headers=headers, data=data)

# ตรวจสอบเงื่อนไขการแจ้งเตือน
for i in range(26, len(df)-3):
    # ราคาย่อมาแตะเส้น MA12
    if df['price'][i] < df['MA12'][i] and df['price'][i-1] >= df['MA12'][i-1]:
        send_line_notify(f"ราคาย่อมาแตะเส้น MA12 ที่วันที่ {df.index[i]}")

        # ตรวจสอบว่าราคาปิดเหนือเส้น MA12 3 แท่งติดกันและ RSI > 50
        if all(df['price'][i+j] > df['MA12'][i+j] for j in range(1, 4)) and df['RSI'][i+3] > 50:
            current_price = df['price'][i+3]
            sl = df['low'][i] + df['ATR'][i]
            tp = current_price + 2 * (current_price - sl)
            send_line_notify(f"BUY NOW: ราคาปัจจุบัน {current_price}, SL {sl}, TP {tp}")

    # ราคาปิดต่ำกว่าเส้น MA12 และ RSI < 50
    if df['price'][i] > df['MA12'][i] and df['price'][i-1] <= df['MA12'][i-1]:
        if all(df['price'][i+j] < df['MA12'][i+j] for j in range(1, 4)) and df['RSI'][i+3] < 50:
            current_price = df['price'][i+3]
            sl = df['high'][i] + df['ATR'][i]
            tp = current_price - 2 * (sl - current_price)
            send_line_notify(f"SELL NOW: ราคาปัจจุบัน {current_price}, SL {sl}, TP {tp}")

# แสดงผล
print(df.tail())
