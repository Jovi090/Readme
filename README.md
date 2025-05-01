HTML 表格更新：positions/list.html】
<!-- templates/positions/list.html (更新后的 HTML 表格部分) -->
<div>
  <h2>Positions</h2>
  <table border="1">
    <thead>
    <tr>
      <th style="text-align: center;">Ticker</th>
      <th style="text-align: center;">Name</th>
      <th style="text-align: center;">Quantity</th>
      <th style="text-align: center;">Average Unit Price</th>
      <th style="text-align: center;">Realized Profit and Loss</th>
      <th style="text-align: center;">Market Price</th>
      <th style="text-align: center;">Valuation</th>
      <th style="text-align: center;">Unrealized Profit and Loss</th>
    </tr>
    </thead>
    <tbody>
    <tr th:each="position : ${positions}">
      <td style="text-align: left;" th:text="${position.ticker}"></td>
      <td style="text-align: left;" th:text="${position.name}"></td>
      <td style="text-align: right;" th:text="${position.quantity == 0 ? '0' : #numbers.formatInteger(position.quantity, 0, 'COMMA')}"></td>
      <td style="text-align: right;" th:text="${position.quantity == 0 ? 'N/A' : #numbers.formatDecimal(position.avgUnitPrice, 1, 2)}"></td>
      <td style="text-align: right;">
        <span th:if="${position.quantity == 0}" th:text="'N/A'"></span>
        <span th:if="${position.quantity != 0}" th:text="${#numbers.formatDecimal(position.realizedProfitLoss, 1, 2)}"
              th:style="${position.realizedProfitLoss > 0 ? 'color:red' : (position.realizedProfitLoss < 0 ? 'color:green' : '')}"></span>
      </td>
      <td style="text-align: right;" th:text="${position.marketPrice == null ? 'N/A' : #numbers.formatDecimal(position.marketPrice, 1, 2)}"></td>
      <td style="text-align: right;" th:text="${position.marketPrice == null ? 'N/A' : (position.quantity == 0 ? 'N/A' : #numbers.formatDecimal(position.valuation, 1, 2))}"
          th:style="${position.valuation > 0 ? 'color:red' : (position.valuation == 0 ? '' : (position.valuation < 0 ? 'color:green' : ''))}"></td>
      <td style="text-align: right;">
        <span th:if="${position.unrealizedProfitLoss == null}" th:text="'N/A'"></span>
        <span th:if="${position.unrealizedProfitLoss != null}" th:text="${#numbers.formatDecimal(position.unrealizedProfitLoss, 1, 2)}"
              th:style="${position.unrealizedProfitLoss > 0 ? 'color:red' : (position.unrealizedProfitLoss < 0 ? 'color:green' : '')}"></span>
      </td>
    </tr>
    </tbody>
  </table>
</div>

【Validation 修复：ValidTradeInputValidator.java】
// ValidTradeInputValidator.java (修复后的验证逻辑)
@Override
public boolean isValid(TradeInputDto input, ConstraintValidatorContext context) {
    if (input.getStockId() == null || input.getTradedDatetime() == null ||
        input.getQuantity() == null || input.getSide() == null) {
        return true;
    }

    List<Trade> trades = tradeRepository.findAllByStockIdOrderByTradedDatetimeAsc(input.getStockId());
    long currentHoldings = 0L;
    for (Trade trade : trades) {
        if (trade.getTradedDatetime().after(Timestamp.valueOf(input.getTradedDatetime()))) break;
        currentHoldings += trade.getSide() == Side.BUY ? trade.getQuantity() : -trade.getQuantity();
    }

    long delta = input.getSide() == Side.BUY ? input.getQuantity() : -input.getQuantity();
    boolean validHoldings = (currentHoldings + delta) >= 0;

    boolean validDatetime = trades.isEmpty() ||
            trades.getLast().getTradedDatetime().compareTo(Timestamp.valueOf(input.getTradedDatetime())) < 0;

    if (!validHoldings) {
        context.disableDefaultConstraintViolation();
        context.buildConstraintViolationWithTemplate("取引後の保有数がマイナスになります。")
                .addPropertyNode("quantity")
                .addConstraintViolation();
    }

    if (!validDatetime) {
        context.disableDefaultConstraintViolation();
        context.buildConstraintViolationWithTemplate("既存の取引より前の日時は指定できません。")
                .addPropertyNode("tradedDatetime")
                .addConstraintViolation();
    }

    return validHoldings && validDatetime;
}

【Repository 修复：防止同一 Ticker 重复行显示】
// PositionRepository.java - 修复重复 ticker 问题的 SQL

public List<Position> findAll() {
    String sql = """
        SELECT s.ticker, s.name,
               COALESCE(SUM(CASE WHEN t.side = 'BUY' THEN t.quantity
                                 WHEN t.side = 'SELL' THEN -t.quantity END), 0) AS quantity,
               CASE WHEN SUM(CASE WHEN t.side = 'BUY' THEN t.quantity ELSE 0 END) > 0
                    THEN ROUND(SUM(CASE WHEN t.side = 'BUY' THEN t.traded_price * t.quantity ELSE 0 END)
                             / NULLIF(SUM(CASE WHEN t.side = 'BUY' THEN t.quantity ELSE 0 END), 0), 2)
                    ELSE NULL END AS avg_unit_price,
               CASE WHEN SUM(CASE WHEN t.side = 'SELL' THEN t.quantity ELSE 0 END) > 0
                    THEN ROUND(SUM(CASE WHEN t.side = 'SELL' THEN (t.traded_price - (
                         SELECT ROUND(SUM(t2.traded_price * t2.quantity)::numeric / NULLIF(SUM(t2.quantity), 0), 2)
                         FROM trade t2 WHERE t2.stock_id = s.id AND t2.side = 'BUY')) * t.quantity), 2)
                    ELSE NULL END AS realized_pl,
               mp.market_price,
               CASE WHEN mp.market_price IS NOT NULL AND
                         SUM(CASE WHEN t.side = 'BUY' THEN t.quantity ELSE 0 END)
                       - SUM(CASE WHEN t.side = 'SELL' THEN t.quantity ELSE 0 END) > 0
                    THEN ROUND(mp.market_price *
                         (SUM(CASE WHEN t.side = 'BUY' THEN t.quantity ELSE 0 END)
                        - SUM(CASE WHEN t.side = 'SELL' THEN t.quantity ELSE 0 END)), 2)
                    ELSE NULL END AS valuation,
               CASE WHEN mp.market_price IS NOT NULL AND
                         SUM(CASE WHEN t.side = 'BUY' THEN t.quantity ELSE 0 END) > 0
                    THEN ROUND((mp.market_price -
                         (SUM(CASE WHEN t.side = 'BUY' THEN t.traded_price * t.quantity ELSE 0 END)
                          / NULLIF(SUM(CASE WHEN t.side = 'BUY' THEN t.quantity ELSE 0 END), 0)))
                          * (SUM(CASE WHEN t.side = 'BUY' THEN t.quantity ELSE 0 END)
                           - SUM(CASE WHEN t.side = 'SELL' THEN t.quantity ELSE 0 END)), 2)
                    ELSE NULL END AS unrealized_pl
        FROM stock s
        LEFT JOIN trade t ON t.stock_id = s.id
        LEFT JOIN (
            SELECT DISTINCT ON (stock_id) stock_id, market_price
            FROM market_price
            ORDER BY stock_id, created_datetime DESC
        ) mp ON mp.stock_id = s.id
        GROUP BY s.ticker, s.name, mp.market_price
        ORDER BY s.ticker
    """;

    return jdbcTemplate.query(sql, (rs, i) -> mapRow(rs));
}












目的】
追加「ポジション一覧画面」：路径 /positions
显示当前持仓状况，包括评价损益，风格统一，基于数据库 SQL 聚合计算。

============================================================
① PositionController.java
路径：main/java/simplex/bn25/zhao102015/server/controller/PositionController.java
============================================================

```java
package simplex.bn25.zhao102015.server.controller;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import simplex.bn25.zhao102015.server.service.PositionService;

@Controller
public class PositionController {

    @Autowired
    private PositionService positionService;

    @GetMapping("/positions")
    public String showPositions(Model model) {
        model.addAttribute("positions", positionService.getAllPositions());
        return "positions/list";
    }
}
```

============================================================
② PositionService.java
路径：main/java/simplex/bn25/zhao102015/server/service/PositionService.java
============================================================

```java
package simplex.bn25.zhao102015.server.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import simplex.bn25.zhao102015.server.model.Position;
import simplex.bn25.zhao102015.server.model.repository.PositionRepository;

import java.util.List;

@Service
public class PositionService {

    @Autowired
    private PositionRepository positionRepository;

    public List<Position> getAllPositions() {
        return positionRepository.findAll();
    }
}
```

============================================================
③ PositionRepository.java
路径：main/java/simplex/bn25/zhao102015/server/model/repository/PositionRepository.java
============================================================

```java
package simplex.bn25.zhao102015.server.model.repository;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Repository;
import simplex.bn25.zhao102015.server.model.Position;

import java.sql.ResultSet;
import java.sql.SQLException;
import java.util.List;

@Repository
public class PositionRepository {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    public List<Position> findAll() {
        String sql = """
            SELECT s.ticker, s.name,
                   COALESCE(SUM(CASE WHEN t.side = 'BUY' THEN t.quantity
                                     WHEN t.side = 'SELL' THEN -t.quantity END), 0) AS quantity,
                   CASE WHEN SUM(CASE WHEN t.side = 'BUY' THEN t.quantity ELSE 0 END) > 0
                        THEN ROUND(SUM(CASE WHEN t.side = 'BUY' THEN t.traded_price * t.quantity ELSE 0 END)
                                 / NULLIF(SUM(CASE WHEN t.side = 'BUY' THEN t.quantity ELSE 0 END), 0), 2)
                        ELSE NULL END AS avg_unit_price,
                   CASE WHEN SUM(CASE WHEN t.side = 'SELL' THEN t.quantity ELSE 0 END) > 0
                        THEN ROUND(SUM(CASE WHEN t.side = 'SELL' THEN (t.traded_price -
                             (SELECT ROUND(SUM(t2.traded_price * t2.quantity)::numeric / NULLIF(SUM(t2.quantity), 0), 2)
                              FROM trade t2 WHERE t2.stock_id = s.id AND t2.side = 'BUY')) * t.quantity
                            ELSE 0 END), 2)
                        ELSE NULL END AS realized_pl,
                   mp.market_price,
                   CASE WHEN mp.market_price IS NOT NULL AND
                             SUM(CASE WHEN t.side = 'BUY' THEN t.quantity ELSE 0 END)
                           - SUM(CASE WHEN t.side = 'SELL' THEN t.quantity ELSE 0 END) > 0
                        THEN ROUND(mp.market_price *
                             (SUM(CASE WHEN t.side = 'BUY' THEN t.quantity ELSE 0 END)
                            - SUM(CASE WHEN t.side = 'SELL' THEN t.quantity ELSE 0 END)), 2)
                        ELSE NULL END AS valuation,
                   CASE WHEN mp.market_price IS NOT NULL AND
                             SUM(CASE WHEN t.side = 'BUY' THEN t.quantity ELSE 0 END) > 0
                        THEN ROUND((mp.market_price -
                             (SUM(CASE WHEN t.side = 'BUY' THEN t.traded_price * t.quantity ELSE 0 END)
                              / NULLIF(SUM(CASE WHEN t.side = 'BUY' THEN t.quantity ELSE 0 END), 0)))
                              * (SUM(CASE WHEN t.side = 'BUY' THEN t.quantity ELSE 0 END)
                               - SUM(CASE WHEN t.side = 'SELL' THEN t.quantity ELSE 0 END)), 2)
                        ELSE NULL END AS unrealized_pl
            FROM stock s
            LEFT JOIN trade t ON t.stock_id = s.id
            LEFT JOIN market_price mp ON mp.stock_id = s.id
            GROUP BY s.ticker, s.name, mp.market_price
            ORDER BY s.ticker
        """;

        return jdbcTemplate.query(sql, (rs, i) -> mapRow(rs));
    }

    private Position mapRow(ResultSet rs) throws SQLException {
        Position p = new Position();
        p.setTicker(rs.getString("ticker"));
        p.setName(rs.getString("name"));
        p.setQuantity(rs.getLong("quantity"));
        p.setAverageUnitPrice(rs.getBigDecimal("avg_unit_price"));
        p.setRealizedPl(rs.getBigDecimal("realized_pl"));
        p.setMarketPrice(rs.getBigDecimal("market_price"));
        p.setValuation(rs.getBigDecimal("valuation"));
        p.setUnrealizedPl(rs.getBigDecimal("unrealized_pl"));
        return p;
    }
}
```

============================================================
④ Position.java
路径：main/java/simplex/bn25/zhao102015/server/model/Position.java
============================================================

```java
package simplex.bn25.zhao102015.server.model;

import java.math.BigDecimal;

public class Position {
    private String ticker;
    private String name;
    private long quantity;
    private BigDecimal averageUnitPrice;
    private BigDecimal realizedPl;
    private BigDecimal marketPrice;
    private BigDecimal valuation;
    private BigDecimal unrealizedPl;

    // Getters & Setters ...
}
```

============================================================
⑤ list.html
路径：main/resources/templates/positions/list.html
============================================================

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
  <meta charset="UTF-8">
  <title>Positions</title>
</head>
<body th:replace="~{layouts :: header}">
<div>
  <h2>Positions</h2>
  <table border="1">
    <thead>
    <tr>
      <th style="text-align:center;">Ticker</th>
      <th style="text-align:center;">Name</th>
      <th style="text-align:right;">Quantity</th>
      <th style="text-align:right;">Average Unit Price</th>
      <th style="text-align:right;">Realized P/L</th>
      <th style="text-align:right;">Market Price</th>
      <th style="text-align:right;">Valuation</th>
      <th style="text-align:right;">Unrealized P/L</th>
    </tr>
    </thead>
    <tbody>
    <tr th:each="pos : ${positions}">
      <td th:text="${pos.ticker}"></td>
      <td th:text="${pos.name}"></td>
      <td th:text="${pos.quantity}"></td>
      <td th:text="${pos.quantity == 0 ? 'N/A' : #numbers.formatDecimal(pos.averageUnitPrice, 1, 2)}"></td>
      <td th:if="${pos.realizedPl == null}" th:text="'N/A'"></td>
      <td th:if="${pos.realizedPl != null}" th:text="${#numbers.formatDecimal(pos.realizedPl, 1, 2)}"
          th:style="${pos.realizedPl > 0 ? 'color:red' : (pos.realizedPl < 0 ? 'color:green' : '')}"></td>
      <td th:text="${pos.marketPrice == null ? 'N/A' : #numbers.formatDecimal(pos.marketPrice, 1, 2)}"></td>
      <td th:text="${pos.valuation == null ? 'N/A' : #numbers.formatDecimal(pos.valuation, 1, 2)}"
          th:style="${pos.valuation > 0 ? 'color:red' : ''}"></td>
      <td th:text="${pos.unrealizedPl == null ? 'N/A' : #numbers.formatDecimal(pos.unrealizedPl, 1, 2)}"
          th:style="${pos.unrealizedPl > 0 ? 'color:red' : (pos.unrealizedPl < 0 ? 'color:green' : '')}"></td>
    </tr>
    </tbody>
  </table>
</div>
</body>
</html>
```

============================================================
⑥ layouts.html 菜单追加
============================================================
```html
<li><a href="/positions">Positions</a></li>
```

【完成】
所有显示数据由 SQL 聚合实现，输出格式与颜色由 Thymeleaf 条件控制。




目的】
完整复现 Trade 输入验证功能② + Market Price 登録功能③
适用于你当前已实装项目结构，内容按文件整理，确保可直接复现。

============================================================
① Trade 输入验证 - 追加 shares_issued 限制
============================================================

【1.1】修改类：ValidTradeInputValidator.java
路径：main/java/simplex/bn25/zhao102015/server/annotation/ValidTradeInputValidator.java

追加：
@Autowired
private StockRepository stockRepository;

在 `isValid()` 方法中，保持已有 validHoldings 判定的基础下，在其后加入：

```java
if (input.getSide() == Side.BUY) {
    Stock stock = stockRepository.findById(input.getStockId()).orElse(null);
    if (stock != null) {
        long totalBuy = trades.stream().filter(t -> t.getSide() == Side.BUY).mapToLong(Trade::getQuantity).sum();
        long totalSell = trades.stream().filter(t -> t.getSide() == Side.SELL).mapToLong(Trade::getQuantity).sum();
        long newTotal = totalBuy - totalSell + input.getQuantity();

        if (newTotal > stock.getSharesIssued()) {
            context.disableDefaultConstraintViolation();
            context.buildConstraintViolationWithTemplate("発行済株式数を超える数量は登録できません。")
                   .addPropertyNode("quantity")
                   .addConstraintViolation();
            return false;
        }
    }
}
```

注意：
- 不需创建新注解类；
- 不影响现有验证；
- 该逻辑仅对 BUY 侧有效。

============================================================
② Market Price Register 页面 - 批量 CSV 登录
============================================================

【2.1】创建 Controller：
路径：main/java/simplex/bn25/zhao102015/server/controller/MarketPriceController.java

```java
@Controller
@RequestMapping("/marketprice")
public class MarketPriceController {

    @Autowired
    private MarketPriceService marketPriceService;

    @GetMapping("/bulk")
    public String showBulkForm(Model model) {
        model.addAttribute("error", null);
        return "marketprice/bulk";
    }

    @PostMapping("/bulk")
    public String handleBulk(@RequestParam("csv") String csv, Model model) {
        String error = marketPriceService.processBulkCsv(csv);
        if (error != null) {
            model.addAttribute("error", error);
            model.addAttribute("csv", csv);
            return "marketprice/bulk";
        }
        return "redirect:/positions";
    }
}
```

【2.2】创建 Service：
路径：main/java/simplex/bn25/zhao102015/server/service/MarketPriceService.java

```java
@Service
public class MarketPriceService {

    @Autowired
    private MarketPriceRepository marketPriceRepository;

    @Autowired
    private StockRepository stockRepository;

    public String processBulkCsv(String csv) {
        if (csv == null || csv.isBlank()) return "入力は必須です。";

        String[] lines = csv.split("\r?\n");
        Set<String> seenTickers = new HashSet<>();
        List<Object[]> entries = new ArrayList<>();

        for (int i = 0; i < lines.length; i++) {
            String[] parts = lines[i].trim().split(",");
            if (parts.length != 2) return (i+1) + "行目：列数が不正です。";

            String ticker = parts[0].trim();
            String priceStr = parts[1].trim();

            if (ticker.isEmpty()) return (i+1) + "行目：tickerは必須です。";
            if (seenTickers.contains(ticker)) return (i+1) + "行目：tickerが重複しています。";
            seenTickers.add(ticker);

            Stock stock = stockRepository.findByTicker(ticker).orElse(null);
            if (stock == null) return (i+1) + "行目：ticker " + ticker + " は存在しません。";

            BigDecimal price;
            try {
                price = new BigDecimal(priceStr);
            } catch (NumberFormatException e) {
                return (i+1) + "行目：金額が不正です。";
            }

            if (price.compareTo(BigDecimal.ZERO) < 0) return (i+1) + "行目：負の金額は不可です。";
            if (price.scale() > 2) return (i+1) + "行目：小数点以下は2桁までです。";

            entries.add(new Object[]{stock.getId(), price});
        }

        for (Object[] e : entries) {
            marketPriceRepository.insert((Integer)e[0], (BigDecimal)e[1]);
        }

        return null;
    }
}
```

【2.3】创建 Repository：
路径：main/java/simplex/bn25/zhao102015/server/model/repository/MarketPriceRepository.java

```java
@Repository
public class MarketPriceRepository {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    public void insert(Integer stockId, BigDecimal price) {
        jdbcTemplate.update(
            "INSERT INTO market_price (stock_id, market_price) VALUES (?, ?)",
            stockId, price
        );
    }
}
```

【2.4】创建 HTML 页面：
路径：main/resources/templates/marketprice/bulk.html

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head><meta charset="UTF-8"><title>Market Price Register</title></head>
<body th:replace="~{layouts :: header}">
<div>
  <h2>Market Price 登録</h2>
  <form method="post" th:action="@{/marketprice/bulk}">
    <div>
      <label for="csv">Market Price</label><br>
      <textarea id="csv" name="csv" rows="10" cols="40" th:text="${csv}"></textarea>
    </div>
    <div>
      <button type="submit">Register</button>
    </div>
    <div th:if="${error}" style="color:red;" th:text="${error}"></div>
  </form>
</div>
</body>
</html>
```

【2.5】导航追加入口
修改 layouts.html：
```html
<li><a href="/marketprice/bulk">MarketPrice Register</a></li>
```

【完成】
两个功能实现将无缝对接你现有结构，验证和注册逻辑集中，结构清晰，输入错误有红字提示，跳转成功后进入 /positions。
