# candlepat
candle patterns for stocks
from yahoo_fin import stock_info as si
from math import isclose
import csv

DELTA = 0.01
LAST_DAY='04/01/2022'
LAST_WEEK='03/21/2022'
LAST_MONTH='02/01/2022'
HAMMER_LONG_WICK_RATIO=3
HAMMER_SHORT_MIN_WICK_RATIO=0.01
HAMMER_SHORT_MAX_WICK_RATIO=0.50
FILENAME_DAILY = 'daily_dump.csv'
FILENAME_WEEKLY = 'weekly_dump.csv'
FILENAME_MONTHLY = 'monthly_dump.csv'

def get_tickers():
    # allow for move tickers in the future. For now, use s&p500
    tickers = si.tickers_sp500()

    return tickers

def find_pattern(o1, c1, h1, l1, o2, c2, h2, l2):
    # opening/closing price are the same
    if (isclose(o1, o2, abs_tol=DELTA) or isclose(c1, c2, abs_tol=DELTA)):
        return None

    # hammer - defined as a wick >= 3x the size of the body with a short wick
    # on the other side. arbitrarily, other wick isn't more than 0.5 body size
    body_size = abs(c2-o2)
        
    if (c2 > o2):
        top_wick = h2 - c2
        bottom_wick = o2 - l2
        #print(f'{top_wick >= HAMMER_LONG_WICK_RATIO*body_size}')
        #print(f'{bottom_wick <= HAMMER_SHORT_MAX_WICK_RATIO*body_size}')
        #print(f'{bottom_wick >= HAMMER_SHORT_MIN_WICK_RATIO*body_size}')
        if ((top_wick >= HAMMER_LONG_WICK_RATIO*body_size) and (bottom_wick <= HAMMER_SHORT_MAX_WICK_RATIO*body_size) and (bottom_wick >= HAMMER_SHORT_MIN_WICK_RATIO*body_size)):
            return "bullish inverted hammer"
        if ((bottom_wick >= HAMMER_LONG_WICK_RATIO*body_size) and (top_wick <= HAMMER_SHORT_MAX_WICK_RATIO*body_size) and (top_wick >= HAMMER_SHORT_MIN_WICK_RATIO*body_size)):
            return "bullish hammer"
    elif (o2 > c2):
        top_wick = h2 - o2
        bottom_wick = c2 - l2
        if ((top_wick >= HAMMER_LONG_WICK_RATIO*body_size) and (bottom_wick <= HAMMER_SHORT_MAX_WICK_RATIO*body_size) and (bottom_wick >= HAMMER_SHORT_MIN_WICK_RATIO*body_size)):
            return "bearish inverted hammer"
        if ((bottom_wick >= HAMMER_LONG_WICK_RATIO*body_size) and (top_wick <= HAMMER_SHORT_MAX_WICK_RATIO*body_size) and (top_wick >= HAMMER_SHORT_MIN_WICK_RATIO*body_size)):
            return "bearish hammer"

    # engulfing and harami 
    # bull cases
    if (o1 > c1 and c2 > o2):
        if (o1<c2 and c1>o2):
            return 'bullish engulfing'
        if (c1<o2 and o1>c2):
            return 'bullish harami'
    # bear cases
    if (c1 > o1 and o2 > c2):
        if (c1<o2 and o1>c2):
            return 'bearish engulfing'
        if (c1>o2 and o1<c2):
            return 'bearish harami'

    # inside day
    if(h1>h2 and l1<l2):
        return "inside day"

    return None

if __name__ == '__main__':
    output = {}

    is_weekly=False
    is_monthly=False
    interval = '1d'
    start_date = LAST_DAY
    output_file = FILENAME_DAILY

    if is_weekly:
        interval = '1wk'
        start_date = LAST_WEEK
        output_file = FILENAME_WEEKLY
    elif is_monthly:    
        interval = '1mo'
        start_date = LAST_MONTH
        output_file = FILENAME_MONTHLY

    for ticker in get_tickers():
        print(f'processing {ticker}')
        data = si.get_data(ticker, start_date=start_date, interval=interval)
        (o1, c1, h1, l1, v1) = data.iloc[0]['open'], data.iloc[0]['close'], data.iloc[0]['high'], data.iloc[0]['low'], data.iloc[0]['volume']
        (o2, c2, h2, l2, v2) = data.iloc[1]['open'], data.iloc[1]['close'], data.iloc[1]['high'], data.iloc[1]['low'], data.iloc[1]['volume']
        
        pattern = find_pattern(o1, c1, h1, l1, o2, c2, h2, l2)
        if pattern is not None:
            info = {}
            info['pattern'] = pattern
            info['volume_increasing'] = False
            if (v2>v1):
                info['volume_increasing'] = True
            output[ticker] = info

    print(f'finished')
    with open(output_file, 'w') as csvfile:
        print(f'writing output to csv file {output_file}')
        field_names = ['ticker', 'candle pattern', 'volume increasing']
        writer = csv.DictWriter(csvfile, fieldnames=field_names)
        writer.writeheader()
        for ticker, info in output.items():
            writer.writerow({'ticker': ticker, 'candle pattern': info['pattern'], 'volume increasing': info['volume_increasing']})

    print('completed writing')
