交易环境的 Web Interface：
![[Trade Panel.png]]
Submit Order:
- 可以指定订单类型为 Buy / Sell
- 对于不同的 Product，每笔订单的 Quantity 可能有不同的限制
- 可以指定订单类型为 Day / IOC。IoC = Immediate or Cancel，要求立即生效，未生效部分立即取消。Day 为日内有效。被撮合的两个成交订单类型不可能同时为 IOC。
- 每笔订单需支付 1 单位手续费，不管是否成交（IoC 订单如果被取消，也要支付手续费）

查询账户状态：
- Query Balance 会返回当前 Cash, 以及各个 Product 的 Position
- Query Order 会返回本账户下的 Open Order (day order)，这些 order 可以被 cancel
- Query PNL history 会返回过去一小时账户的 P
- nL 曲线


每 0.25s 刷新一次 Market Data：
![[Limit Order Book.png]]