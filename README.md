# krx_backtest_core

코스피·코스닥 일봉 백테스트 엔진이다. 매매 규칙은 없다. 규칙은 알고리즘마다 따로 저장소를 만들어 그쪽에 둔다.
이 저장소는 여러 알고리즘이 같은 데이터와 같은 계산 방식을 쓰게 하려고 [bnf_backtest](https://github.com/destory1984/bnf_backtest) 에서 떼어 냈다.

| 모듈 | 하는 일 |
|---|---|
| `krxbt.fetch` | 종목 목록·지수·일봉 수집 (네이버 `siseJson`, 상장폐지 종목은 KRX KIND). 처음엔 3,243종목에 약 35분 |
| `krxbt.data` | 받은 데이터 읽기, 쓸 수 있는 종목 목록(`universe_tickers`) |
| `krxbt.frame` | 종목별 일봉 표. 거래정지·정리매매·수정 안 된 시세 끊김을 거르는 규칙이 여기 있다 |
| `krxbt.engine` | 신호와 익절 조건(불리언 배열 두 개)을 받아 거래 목록을 만든다. 다음 날 시가 진입, 손절, 최대 보유일, 비용 |
| `krxbt.costs` | 수수료·슬리피지·매도세(해마다 바뀐 세율) |
| `krxbt.portfolio` | 신호 거래 목록으로 동시 보유 N종목 포트폴리오를 돌린다. 폭락일에 자리 늘리기, 손절 뒤 재진입 제한 |

## 설치

알고리즘 저장소의 `requirements.txt` 에 버전을 박아서 설치한다. 엔진을 고쳐도 옛 알고리즘의 결과가 바뀌지 않게 하려는 것이다.

```
krxbt @ git+https://github.com/destory1984/krx_backtest_core@v0.1.0
```

엔진을 같이 고치는 중이면 로컬 폴더를 편집 가능 모드로 설치한다.

```
pip install -e ../krx_backtest_core
```

## 데이터 폴더

시세는 어느 저장소에도 넣지 않는다. 크기가 약 260MB 이고, 네이버 시세를 다시 배포하는 일이 되기 때문이다.
알고리즘 저장소 옆에 `krx_data` 폴더를 두고 모든 알고리즘이 그것을 같이 읽는다.

```
C:\_c\krx_data             시세 (prices/*.parquet, index_*.parquet, universe.parquet)
C:\_c\krx_backtest_core    이 저장소
C:\_c\bnf_backtest         알고리즘 1
C:\_c\다른_알고리즘        알고리즘 2
```

폴더 위치는 환경변수 `KRX_DATA_DIR` 가 있으면 그것을 쓰고, 없으면 config 의 `data.dir` 을 config 파일 기준 상대 경로로 읽는다(`../krx_data`).

수집·갱신은 알고리즘 저장소 하나에서 돌리면 된다. 다시 돌리면 새 날짜만 받는다.

```
python -m krxbt.fetch                 # config.yaml 의 data·universe 설정을 읽는다
python -m krxbt.fetch --config path/to/config.yaml
```

## 새 알고리즘 만들기

config.yaml 에 `data`, `universe`, `indicators`, `rules`, `costs` 가 있어야 한다. `bnf_backtest/config.yaml` 을 복사해서 시작하면 된다.
알고리즘이 할 일은 종목별로 "언제 사는가"(`cand`)와 "언제 익절하는가"(`target`)를 불리언 배열로 만드는 것뿐이다.

```python
from krxbt.config import load_config
from krxbt.data import load_calendar, universe_tickers
from krxbt.frame import index_features, ticker_frame
from krxbt import engine

cfg = load_config("config.yaml")
cal, idx = load_calendar(cfg), index_features(cfg)
tk = universe_tickers(cfg)
f = ticker_frame(cfg, "005930", "KOSPI", cal, idx, delisted=False)
a = engine.arrays(f)
cand = a["eligible"] & (a["disp"] <= 85) & engine.market_mask(a, "crash97")
target = a["close"] >= a["ma"]
trades = engine.run(a, cfg, cand, target, stop=-0.08, hold=10)
trades = trades[~trades["data_break"]]   # 시세 끊김을 끼고 들고 있던 거래는 뺀다
```

## 거르는 규칙 (`krxbt.frame`)

- 거래량 0 인 날은 거래정지로 보고 표에 남기되 사고팔지 않는다. 보유일 수에는 센다.
- 상장폐지 종목의 마지막 7거래일(정리매매)에는 신호를 막는다. 가격 제한이 없어 하루 -50% 가 나온다.
- 하루 하락이 가격 제한(2015-06-15 전 -16%, 이후 -30%)을 넘거나, 3거래일 넘는 거래정지가 풀리면 그날부터 이동평균 기간(25거래일) 동안 신호를 막는다.
- 그런 끊긴 날을 끼고 들고 있던 거래는 `data_break` 로 표시한다. 결과에서 빼는 것은 알고리즘 몫이다.
- 20일 평균 거래대금(종가 × 거래량 어림) 10억 원 미만, 상장 25거래일 미만 구간은 신호를 막는다(`eligible`).

## 버전

- v0.1.0 (2026-09-27): bnf_backtest 에서 떼어 냄. 익절 조건을 "종가 ≥ 25일선" 고정에서 알고리즘이 넘기는 배열로 바꿨다. 결과 숫자는 떼기 전과 같다.
