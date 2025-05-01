目的】
为交易一览页面（/trade）追加筛选功能，不破坏已有功能与结构。
功能包括按 Ticker 文字、交易日（全部 or 今日）筛选，并支持空白时显示全部。

---

【修改 1】TradeController.java
（位置：main/java/simplex/bn25/zhao102015/server/controller/TradeController.java）

在类中追加以下 import（放在已有 import 之后）：

import jakarta.servlet.http.HttpServletRequest;
import java.sql.Timestamp;
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.time.LocalTime;

---

在 @GetMapping 下追加以下方法：

@GetMapping
public String listTrades(HttpServletRequest request, Model model) {
    String ticker = request.getParameter("ticker");
    String date = request.getParameter("date");

    List<simplex.bn25.zhao102015.server.model.Trade> trades;

    if ((ticker == null || ticker.isEmpty()) && (date == null || date.equals("all"))) {
        trades = tradeService.findAll();
    } else {
        trades = tradeService.findByFilter(ticker, date);
    }

    model.addAttribute("trades", trades);
    model.addAttribute("ticker", ticker == null ? "" : ticker);
    model.addAttribute("date", date == null ? "all" : date);
    return "trades/list";
}

---

【修改 2】TradeService.java
（位置：main/java/simplex/bn25/zhao102015/server/service/TradeService.java）

追加方法：

public List<Trade> findByFilter(String ticker, String date) {
    List<Trade> all = tradeRepository.findAll();
    List<Trade> filtered = new java.util.ArrayList<>();

    LocalDate today = LocalDate.now();
    Timestamp todayStart = Timestamp.valueOf(LocalDateTime.of(today, LocalTime.MIDNIGHT));
    Timestamp todayEnd = Timestamp.valueOf(LocalDateTime.of(today, LocalTime.MAX));

    for (Trade t : all) {
        boolean match = true;

        if (ticker != null && !ticker.isEmpty()) {
            if (!t.getTicker().toLowerCase().contains(ticker.toLowerCase())) {
                match = false;
            }
        }

        if ("today".equalsIgnoreCase(date)) {
            Timestamp traded = t.getTradedDatetime();
            if (traded.before(todayStart) || traded.after(todayEnd)) {
                match = false;
            }
        }

        if (match) {
            filtered.add(t);
        }
    }

    return filtered;
}

---

【修改 3】trades/list.html
（位置：main/resources/templates/trades/list.html）

在 <table> 标签之前，追加以下 HTML 表单：

<form method="get" th:action="@{/trade}" style="margin-bottom: 20px;">
  <div>
    <label for="ticker">Ticker:</label>
    <input type="text" id="ticker" name="ticker" th:value="${ticker}" placeholder="Ticker" />
  </div>
  <div>
    <label>Date:</label>
    <input type="radio" id="all" name="date" value="all"
           th:checked="${date == 'all'}" />
    <label for="all">ALL</label>

    <input type="radio" id="today" name="date" value="today"
           th:checked="${date == 'today'}" />
    <label for="today">Today</label>
  </div>
  <div>
    <button type="submit" style="background-color: lightgreen;">フィルター</button>
  </div>
</form>

---

【完成】
- 无匹配数据时自动显示“データがありません。” 无需改动；
- 保持原有数据展示功能；
- 新筛选功能以参数 ?ticker=xxx&date=today/all 形式运行；
- 所有代码只做追加，未动原有结构与功能。




【目的】
为取引入力追加一个验证逻辑：
→ 保有数量 + 输入数量 不得超过 stock 表中的 shares_issued。
即：累积买入数量 - 累积卖出数量 + 此次交易数量 ≤ shares_issued。

---

【修改 1】新增 annotation：@WithinSharesIssued
（位置建议：main/java/simplex/bn25/zhao102015/server/annotation/WithinSharesIssued.java）

```java
package simplex.bn25.zhao102015.server.annotation;

import jakarta.validation.Constraint;
import jakarta.validation.Payload;

import java.lang.annotation.*;

@Documented
@Constraint(validatedBy = WithinSharesIssuedValidator.class)
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface WithinSharesIssued {
    String message() default "発行済株式数を超える数量は登録できません。";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

---

【修改 2】新增验证器类
（位置建议：main/java/simplex/bn25/zhao102015/server/annotation/WithinSharesIssuedValidator.java）

```java
package simplex.bn25.zhao102015.server.annotation;

import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;
import org.springframework.beans.factory.annotation.Autowired;
import simplex.bn25.zhao102015.server.controller.TradeInputDto;
import simplex.bn25.zhao102015.server.model.Side;
import simplex.bn25.zhao102015.server.model.repository.TradeRepository;
import simplex.bn25.zhao102015.server.model.repository.StockRepository;
import simplex.bn25.zhao102015.server.model.Trade;
import simplex.bn25.zhao102015.server.model.Stock;

import java.util.List;

public class WithinSharesIssuedValidator implements ConstraintValidator<WithinSharesIssued, TradeInputDto> {

    @Autowired
    private TradeRepository tradeRepository;

    @Autowired
    private StockRepository stockRepository;

    @Override
    public boolean isValid(TradeInputDto input, ConstraintValidatorContext context) {
        if (input.getStockId() == null || input.getQuantity() == null || input.getSide() != Side.BUY) {
            return true; // 卖出或不完整数据不验证
        }

        List<Trade> trades = tradeRepository.findAllByStockId(input.getStockId());
        long currentTotalBuy = trades.stream()
                .filter(t -> t.getSide() == Side.BUY)
                .mapToLong(Trade::getQuantity)
                .sum();

        long currentTotalSell = trades.stream()
                .filter(t -> t.getSide() == Side.SELL)
                .mapToLong(Trade::getQuantity)
                .sum();

        long afterBuy = currentTotalBuy - currentTotalSell + input.getQuantity();

        Stock stock = stockRepository.findById(input.getStockId()).orElse(null);
        if (stock == null) return true;

        return afterBuy <= stock.getSharesIssued();
    }
}
```

---

【修改 3】TradeInputDto.java 添加注解
（文件位置：main/java/simplex/bn25/zhao102015/server/controller/TradeInputDto.java）

在 class 上添加注解：

```java
@WithinSharesIssued
public class TradeInputDto {
    ...
```

---

【修改 4】trades/input.html（若使用）
（路径：templates/trades/input.html）

可选：在数量栏下添加一段提示错误信息：

```html
<div th:if="${#fields.hasErrors('quantity')}" th:errors="*{quantity}" style="color:red"></div>
```

---

【完成】
- 只影响买入交易；
- 自动从 trade 表累积已有买卖数量；
- 调用 stock 表判断 shares_issued；
- 如超过，提示“発行済株式数を超える数量は登録できません。”

以上逻辑完全通过 annotation + validator 实现，不影响控制器代码。

