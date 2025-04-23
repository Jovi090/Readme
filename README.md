// FILE: TradeInputDto.java
package simplex.bn25.zhao102015.server.controller;

import org.springframework.format.annotation.DateTimeFormat;
import jakarta.validation.constraints.*;
import java.math.BigDecimal;
import java.time.LocalDate;

public class TradeInputDto {

    @NotNull(message = "入力してください")
    @DateTimeFormat(pattern = "yyyy-MM-dd")
    private LocalDate tradeDate;

    @NotBlank(message = "入力してください")
    private String ticker;

    @NotBlank(message = "入力してください")
    private String side;

    @NotNull(message = "入力してください")
    @Min(value = 100, message = "100単位で入力してください")
    private Integer quantity;

    @NotNull(message = "入力してください")
    @DecimalMin(value = "0.01", inclusive = true, message = "0より大きい金額を入力してください")
    @Digits(integer = 10, fraction = 2, message = "小数点以下は2桁までです")
    private BigDecimal price;

    // getters and setters ...


// END OF FILE

// FILE: StockRepository.java
package simplex.bn25.zhao102015.server.model.repository;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.BeanPropertyRowMapper;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Repository;
import simplex.bn25.zhao102015.server.model.Stock;

import java.util.List;
import java.util.Optional;

@Repository
public class StockRepository {

    private final JdbcTemplate jdbcTemplate;

    @Autowired
    public StockRepository(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    public List<Stock> findAll() {
        String sql = "SELECT * FROM stock ORDER BY id";
        return jdbcTemplate.query(sql, new BeanPropertyRowMapper<>(Stock.class));
    }

    public Optional<Stock> findByTicker(String ticker) {
        String sql = "SELECT * FROM stock WHERE ticker = ?";
        List<Stock> results = jdbcTemplate.query(sql, new BeanPropertyRowMapper<>(Stock.class), ticker);
        return results.stream().findFirst();
    }

    public Stock register(Stock stock) {
        String sql = "INSERT INTO stock (ticker, name, exchange_market, shares_issued, created_datetime) " +
                "VALUES (?, ?, ?, ?, current_timestamp)";
        jdbcTemplate.update(sql, stock.getTicker(), stock.getName(), stock.getExchangeMarket(), stock.getSharesIssued());
        return stock;
    }
}


import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

public interface StockRepository extends JpaRepository<Stock, String> {

    @Query("SELECT s.sharedIssued FROM Stock s WHERE s.ticker = :ticker")
    Integer findSharedIssuedByTicker(@Param("ticker") String ticker);
}


// END OF FILE

// FILE: TradeService.java
package simplex.bn25.zhao102015.server.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import simplex.bn25.zhao102015.server.controller.TradeInputDto;
import simplex.bn25.zhao102015.server.model.Trade;
import simplex.bn25.zhao102015.server.model.repository.TradeRepository;

import java.sql.Timestamp;
import java.util.List;

@Service
@Service
public class TradeService {

    @Autowired
    private StockRepository stockRepository;

    public boolean isValidQuantity(String ticker, int quantity) {
        Integer sharedIssued = stockRepository.findSharedIssuedByTicker(ticker);
        return sharedIssued != null && quantity > 0 && quantity % 100 == 0 && quantity <= sharedIssued;
    }


    private final TradeRepository tradeRepository;

    @Autowired
    public TradeService(TradeRepository tradeRepository) {
        this.tradeRepository = tradeRepository;
    }

    public List<Trade> findAll() {
        return tradeRepository.findAll();
    }

    public void register(TradeInputDto input) {
        Trade trade = new Trade();
        trade.setStockId(input.getStockId());
        trade.setTradedDatetime(Timestamp.valueOf(input.getTradedDatetime()));
        trade.setSide(input.getSide());
        trade.setQuantity(input.getQuantity());
        trade.setTradedPrice(input.getTradedPrice());
        tradeRepository.insert(trade);
    }
}


// END OF FILE

// FILE: TradeController.java
package simplex.bn25.zhao102015.server.controller;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.validation.BindingResult;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.*;
import simplex.bn25.zhao102015.server.controller.TradeInputDto;
import simplex.bn25.zhao102015.server.model.Stock;
import simplex.bn25.zhao102015.server.service.StockService;
import simplex.bn25.zhao102015.server.service.TradeService;

import java.util.List;

@Controller
@RequestMapping("/trade")
@Controller
public class TradeController {

    private boolean allowTradeInput = false;

    private boolean isWithinTradingHours() {
        java.time.LocalTime now = java.time.LocalTime.now();
        java.time.DayOfWeek day = java.time.LocalDate.now().getDayOfWeek();
        return !day.equals(java.time.DayOfWeek.SATURDAY)
            && !day.equals(java.time.DayOfWeek.SUNDAY)
            && now.isAfter(java.time.LocalTime.of(9, 0))
            && now.isBefore(java.time.LocalTime.of(15, 0));
    }


    private final TradeService tradeService;
    private final StockService stockService;

    @Autowired
    public TradeController(TradeService tradeService, StockService stockService) {
        this.tradeService = tradeService;
        this.stockService = stockService;
    }

    @GetMapping
    public String list(Model model) {
        List<?> trades = tradeService.findAll();
        model.addAttribute("trades", trades);
        model.addAttribute("noData", trades.isEmpty());
        return "trades/list";
    }

    @GetMapping("/new")
    public String newTradeForm(@RequestParam("ticker") String ticker, Model model) {
        Stock stock = stockService.findByTicker(ticker);
        TradeInputDto dto = new TradeInputDto();
        dto.setStockId(stock.getId());
        dto.setTicker(stock.getTicker());
        dto.setName(stock.getName());

        model.addAttribute("input", dto);
        return "trades/input";
    }

    @PostMapping("/new")
    public String registerTrade(@Validated @ModelAttribute("input") TradeInputDto input,
                                BindingResult bindingResult, Model model) {
        if (!isWithinTradingHours()) {
            bindingResult.reject("tradeTime", "取引は平日9時〜15時のみ可能です");
        }
        if (!tradeService.isValidQuantity(tradeInputDto.getTicker(), tradeInputDto.getQuantity())) {
            bindingResult.rejectValue("quantity", "invalid", "100単位で発行済株式数以内で入力してください");
        }
        if (bindingResult.hasErrors()) {
            model.addAttribute("ticker", tradeInputDto.getTicker());
            model.addAttribute("companyName", "会社名仮");
        
            return "trades/input";
        }

        tradeService.register(input);
        return "redirect:/trade";
    }
}


// END OF FILE

// FILE: input.html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
  <meta charset="UTF-8">
  <title>Input Trade</title>
</head>
<body>
<div>
  <h2>Input Trade</h2>
  <form th:object="${tradeInputDto}" novalidate method="post" th:action="@{/trade/new}" th:object="${input}">
    <input style="margin-bottom: 25px;" type="hidden" th:field="*{stockId}"/>

    <div>
      <label for="ticker">Ticker</label>
      <input style="margin-bottom: 25px;" type="text" id="ticker" th:value="${input.ticker}" disabled/>
    </div>

    <div>
      <span th:text="${input.name}"></span>
    </div>

    <div>
      <label for="tradedDatetime">Traded Datetime</label>
      <input style="margin-bottom: 25px;" type="datetime-local" id="tradedDatetime" th:field="*{tradedDatetime}" required/>
      <div th:if="${#fields.hasErrors('tradedDatetime')}" th:errors="*{tradedDatetime}" style="color:red"></div>
    </div>

    <div>
      <label>Side</label>
      <input style="margin-bottom: 25px;" type="radio" id="buy" th:field="*{side}" value="BUY"/> Buy
      <input style="margin-bottom: 25px;" type="radio" id="sell" th:field="*{side}" value="SELL"/> Sell
      <div th:if="${#fields.hasErrors('side')}" th:errors="*{side}" style="color:red"></div>
    </div>

    <div>
      <label for="quantity">Quantity</label>
      <input style="margin-bottom: 25px;" type="number" id="quantity" th:field="*{quantity}" min="1" step="100" required/>
      <div th:if="${#fields.hasErrors('quantity')}" th:errors="*{quantity}" style="color:red"></div>
    </div>

    <div>
      <label for="tradedPrice">Traded Price</label>
      <input style="margin-bottom: 25px;" type="number" id="tradedPrice" th:field="*{tradedPrice}" step="0.01" required/>
      <div th:if="${#fields.hasErrors('tradedPrice')}" th:errors="*{tradedPrice}" style="color:red"></div>
    </div>

    <div>
      <input style="margin-bottom: 25px;" type="submit" value="Register"/>
    </div>
  </form>
</div>
</body>
</html>
<div th:if="${#fields.hasErrors('tradeDate')}" th:errors="*{tradeDate}" style="color:red"></div>
<div th:if="${#fields.hasErrors('ticker')}" th:errors="*{ticker}" style="color:red"></div>
<div th:if="${#fields.hasErrors('side')}" th:errors="*{side}" style="color:red"></div>
<div th:if="${#fields.hasErrors('quantity')}" th:errors="*{quantity}" style="color:red"></div>
<div th:if="${#fields.hasErrors('price')}" th:errors="*{price}" style="color:red"></div>
<div th:if="${#fields.hasGlobalErrors()}" th:each="err : ${#fields.globalErrors()}" th:text="${err}" style="color:red"></div>


// END OF FILE

// FILE: list.html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
  <meta charset="UTF-8">
  <title>Trade List</title>
</head>
<body>
<div>
  <h1>trading-app-v2 by zhao102015</h1>
  <nav>
    <ul>
      <li><a href="/">Home</a></li>
      <li><a href="/hello">Hello</a></li>
      <li><a href="/stocks">Stocks</a></li>
      <li><a href="/stocks/new">Create Stock</a></li>
      <li><a href="/trade">Trade List</a></li>
    </ul>
  </nav>
</div>
<div>

  <table>
    <thead>
    <tr>
      <th style="text-align: left;">Traded Datetime</th>
      <th style="text-align: left;">Ticker</th>
      <th style="text-align: left;">Name</th>
      <th style="text-align: left;">Side</th>
      <th style="text-align: right;">Quantity</th>
      <th style="text-align: right;">Traded Price</th>
    </tr>
    </thead>
    <tbody>
    <!-- 无数据时显示提示 -->
    <tr th:if="${#lists.isEmpty(trades)}">
      <td colspan="6" style="text-align: center;">データがありません。</td>
    </tr>

    <!-- 有数据时逐行渲染 -->
    <tr th:each="trade : ${trades}" th:unless="${#lists.isEmpty(trades)}">
      <td th:text="${#dates.format(trade.tradedDatetime, 'yyyy/MM/dd HH:mm')}"></td>
      <td th:text="${trade.ticker}"></td>
      <td th:text="${trade.name}"></td>
      <td th:text="${trade.side == 'Buy' ? 'Buy' : 'Sell'}"></td>
      <td th:text="${#numbers.formatInteger(trade.quantity, 0, 'COMMA')}" style="text-align: right;"></td>
      <td th:text="${#numbers.formatDecimal(trade.tradedPrice, 1, 2)}" style="text-align: right;"></td>
    </tr>
    </tbody>
  </table>
</div>
</body>
</html>


// END OF FILE


111111111
// FILE: TradeInputDto.java
package simplex.bn25.zhao102015.server.controller;

import jakarta.validation.constraints.*;
import org.springframework.format.annotation.DateTimeFormat;
import simplex.bn25.zhao102015.server.model.Side;

import java.math.BigDecimal;
import java.time.LocalDateTime;

public class TradeInputDto {

    @javax.validation.constraints.NotNull(message = "入力してください")
    private java.time.LocalDate tradeDate;

    @javax.validation.constraints.NotNull(message = "入力してください")
    private String ticker;

    @javax.validation.constraints.NotNull(message = "入力してください")
    @javax.validation.constraints.Pattern(regexp = "Buy|Sell", message = "入力してください")
    private String side;

    @javax.validation.constraints.NotNull(message = "入力してください")
    @javax.validation.constraints.Min(value = 100, message = "100単位で入力してください")
    private Integer quantity;

    @javax.validation.constraints.NotNull(message = "入力してください")
    @javax.validation.constraints.DecimalMin(value = "0.01", message = "0より大きい金額を入力してください")
    @javax.validation.constraints.Digits(integer = 10, fraction = 2, message = "小数点以下は2桁までです")
    private java.math.BigDecimal price;

    @NotNull
    private Integer stockId;

    private String ticker;
    private String name;

    @NotNull
    @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME)
    private LocalDateTime tradedDatetime;

    @NotNull
    private Side side;

    @NotNull
    @Positive
    private Long quantity;

    @NotNull
    @Digits(integer = 10, fraction = 2)
    private BigDecimal tradedPrice;

    public Integer getStockId() {
        return stockId;
    }

    public void setStockId(Integer stockId) {
        this.stockId = stockId;
    }

    public String getTicker() {
        return ticker;
    }

    public void setTicker(String ticker) {
        this.ticker = ticker;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public LocalDateTime getTradedDatetime() {
        return tradedDatetime;
    }

    public void setTradedDatetime(LocalDateTime tradedDatetime) {
        this.tradedDatetime = tradedDatetime;
    }

    public Side getSide() {
        return side;
    }

    public void setSide(Side side) {
        this.side = side;
    }

    public Long getQuantity() {
        return quantity;
    }

    public void setQuantity(Long quantity) {
        this.quantity = quantity;
    }

    public BigDecimal getTradedPrice() {
        return tradedPrice;
    }

    public void setTradedPrice(BigDecimal tradedPrice) {
        this.tradedPrice = tradedPrice;
    }

    public static TradeInputDto fromStock(simplex.bn25.zhao102015.server.model.Stock stock) {
        TradeInputDto dto = new TradeInputDto();
        dto.setStockId(stock.getId());
        dto.setTicker(stock.getTicker());
        dto.setName(stock.getName());
        return dto;
    }
}


// END OF FILE

// FILE: TradeController.java
package simplex.bn25.zhao102015.server.controller;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.validation.BindingResult;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.*;
import simplex.bn25.zhao102015.server.controller.TradeInputDto;
import simplex.bn25.zhao102015.server.model.Stock;
import simplex.bn25.zhao102015.server.service.StockService;
import simplex.bn25.zhao102015.server.service.TradeService;

import java.util.List;

@Controller
@RequestMapping("/trade")
public class TradeController {
    private boolean validNavigation = false;

    private boolean isWithinTradingHours() {
        java.time.LocalTime now = java.time.LocalTime.now();
        java.time.DayOfWeek day = java.time.LocalDate.now().getDayOfWeek();
        return !day.equals(java.time.DayOfWeek.SATURDAY)
            && !day.equals(java.time.DayOfWeek.SUNDAY)
            && now.isAfter(java.time.LocalTime.of(9, 0))
            && now.isBefore(java.time.LocalTime.of(15, 0));
    }

    private final TradeService tradeService;
    private final StockService stockService;

    @Autowired
    public TradeController(TradeService tradeService, StockService stockService) {
        this.tradeService = tradeService;
        this.stockService = stockService;
    }

    @GetMapping
    public String list(Model model) {
        List<?> trades = tradeService.findAll();
        model.addAttribute("trades", trades);
        model.addAttribute("noData", trades.isEmpty());
        return "trades/list";
    }

    @GetMapping("/new")
    public String newTradeForm(@RequestParam("ticker") String ticker, Model model) {
        Stock stock = stockService.findByTicker(ticker);
        TradeInputDto dto = new TradeInputDto();
        dto.setStockId(stock.getId());
        dto.setTicker(stock.getTicker());
        dto.setName(stock.getName());

        model.addAttribute("input", dto);
        return "trades/input";
    }

    @PostMapping("/new")
    public String registerTrade(@Validated @ModelAttribute("input") TradeInputDto input,
                                BindingResult bindingResult, Model model) {
        if (bindingResult.hasErrors()) {
            return "trades/input";
        }

        tradeService.register(input);
        return "redirect:/trade";
    }
}


// END OF FILE

// FILE: TradeService.java
package simplex.bn25.zhao102015.server.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import simplex.bn25.zhao102015.server.controller.TradeInputDto;
import simplex.bn25.zhao102015.server.model.Trade;
import simplex.bn25.zhao102015.server.model.repository.TradeRepository;

import java.sql.Timestamp;
import java.util.List;

@Service
public class TradeService {
    @org.springframework.beans.factory.annotation.Autowired
    private simplex.bn25.zhao102015.server.model.repository.StockRepository stockRepository;

    public boolean validateTradeInput(String ticker, int quantity) {
        return stockRepository.findById(ticker)
                .map(stock -> stock.getSharedIssued() >= quantity && quantity % 100 == 0 && quantity > 0)
                .orElse(false);
    }

    private final TradeRepository tradeRepository;

    @Autowired
    public TradeService(TradeRepository tradeRepository) {
        this.tradeRepository = tradeRepository;
    }

    public List<Trade> findAll() {
        return tradeRepository.findAll();
    }

    public void register(TradeInputDto input) {
        Trade trade = new Trade();
        trade.setStockId(input.getStockId());
        trade.setTradedDatetime(Timestamp.valueOf(input.getTradedDatetime()));
        trade.setSide(input.getSide());
        trade.setQuantity(input.getQuantity());
        trade.setTradedPrice(input.getTradedPrice());
        tradeRepository.insert(trade);
    }
}


// END OF FILE

// FILE: input.html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
  <meta charset="UTF-8">
  <title>Input Trade</title>
</head>
<body>
<div>
  <h2>Input Trade</h2>
  <form novalidate method="post" th:action="@{/trade/new}" th:object="${input}">
    <input style="margin-bottom: 15px;" type="hidden" th:field="*{stockId}"/>

    <div>
      <label for="ticker">Ticker</label>
      <input style="margin-bottom: 15px;" type="text" id="ticker" th:value="${input.ticker}" disabled/>
    </div>

    <div>
      <span th:text="${input.name}"></span>
    </div>

    <div>
      <label for="tradedDatetime">Traded Datetime</label>
      <input style="margin-bottom: 15px;" type="datetime-local" id="tradedDatetime" th:field="*{tradedDatetime}" required/>
      <div th:if="${#fields.hasErrors('tradedDatetime')}" th:errors="*{tradedDatetime}" style="color:red"></div>
    </div>

    <div>
      <label>Side</label>
      <input style="margin-bottom: 15px;" type="radio" id="buy" th:field="*{side}" value="BUY"/> Buy
      <input style="margin-bottom: 15px;" type="radio" id="sell" th:field="*{side}" value="SELL"/> Sell
      <div th:if="${#fields.hasErrors('side')}" th:errors="*{side}" style="color:red"></div>
    </div>

    <div>
      <label for="quantity">Quantity</label>
      <input style="margin-bottom: 15px;" type="number" id="quantity" th:field="*{quantity}" min="1" step="100" required/>
      <div th:if="${#fields.hasErrors('quantity')}" th:errors="*{quantity}" style="color:red"></div>
    </div>

    <div>
      <label for="tradedPrice">Traded Price</label>
      <input style="margin-bottom: 15px;" type="number" id="tradedPrice" th:field="*{tradedPrice}" step="0.01" required/>
      <div th:if="${#fields.hasErrors('tradedPrice')}" th:errors="*{tradedPrice}" style="color:red"></div>
    </div>

    <div>
      <input style="margin-bottom: 15px;" type="submit" value="Register"/>
    </div>
  </form>
</div>
</body>
</html>

// END OF FILE

// FILE: list.html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
  <meta charset="UTF-8">
  <title>Trade List</title>
</head>
<body>
<div>
  <h1>trading-app-v2 by zhao102015</h1>
  <nav>
    <ul>
      <li><a href="/">Home</a></li>
      <li><a href="/hello">Hello</a></li>
      <li><a href="/stocks">Stocks</a></li>
      <li><a href="/stocks/new">Create Stock</a></li>
      <li><a href="/trade">Trade List</a></li>
    </ul>
  </nav>
</div>
<div>

  <table>
    <thead>
    <tr>
      <th style="text-align: left;">Traded Datetime</th>
      <th style="text-align: left;">Ticker</th>
      <th style="text-align: left;">Name</th>
      <th style="text-align: left;">Side</th>
      <th style="text-align: right;">Quantity</th>
      <th style="text-align: right;">Traded Price</th>
    </tr>
    </thead>
    <tbody>
    <!-- 无数据时显示提示 -->
    <tr th:if="${#lists.isEmpty(trades)}">
      <td colspan="6" style="text-align: center;">データがありません。</td>
    </tr>

    <!-- 有数据时逐行渲染 -->
    <tr th:each="trade : ${trades}" th:unless="${#lists.isEmpty(trades)}">
      <td th:text="${#dates.format(trade.tradedDatetime, 'yyyy/MM/dd HH:mm')}"></td>
      <td th:text="${trade.ticker}"></td>
      <td th:text="${trade.name}"></td>
      <td th:text="${trade.side == 'Buy' ? 'Buy' : 'Sell'}"></td>
      <td th:text="${#numbers.formatInteger(trade.quantity, 0, 'COMMA')}" style="text-align: right;"></td>
      <td th:text="${#numbers.formatDecimal(trade.tradedPrice, 1, 2)}" style="text-align: right;"></td>
    </tr>
    </tbody>
  </table>
</div>
</body>
</html>


// END OF FILE

==== FILE: /main/java/simplex/bn25/zhao102015/server/ServerApplication.java ====

package simplex.bn25.zhao102015.server;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class ServerApplication {
	public static void main(String[] args) {
		SpringApplication.run(ServerApplication.class, args);
	}
}



==== FILE: /main/java/simplex/bn25/zhao102015/server/controller/HelloController.java ====

package simplex.bn25.zhao102015.server.controller;

import com.fasterxml.jackson.databind.jsonschema.JsonSerializableSchema;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestParam;

@Controller
public class HelloController {

    @GetMapping("/hello/{name}")
    public String hello(@PathVariable String name, Model model) {
        model.addAttribute("message", String.format("Hello World, Hello %s.", name));
        return "hello";
    }

    @GetMapping("/hello")
    public String callNameByQueryParam(
            @RequestParam(name = "name", required = false, defaultValue = "World") String name,
            Model model
    ) {
        model.addAttribute("message", String.format("Hello World, Hello %s.", name));
        return "hello";
    }

}


==== FILE: /main/java/simplex/bn25/zhao102015/server/controller/HomeController.java ====

package simplex.bn25.zhao102015.server.controller;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;

@Controller
public class HomeController {

    @GetMapping("/")
    public String index() {
        return "home";
    }
}



==== FILE: /main/java/simplex/bn25/zhao102015/server/controller/StockController.java ====

package simplex.bn25.zhao102015.server.controller;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.dao.DataIntegrityViolationException;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.ModelAttribute;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import simplex.bn25.zhao102015.server.service.StockService;
import simplex.bn25.zhao102015.server.model.Stock;
import org.springframework.validation.annotation.Validated;
import org.springframework.ui.Model;
import org.springframework.validation.BindingResult;

import java.util.List;

@Controller
@RequestMapping("/stocks")
public class StockController {

    private final StockService stockService;

    @Autowired
    public StockController(StockService stockService) {
        this.stockService = stockService;
    }

    @GetMapping
    public String list(Model model) {
        List<Stock> stocks = stockService.listAll();
        model.addAttribute("stocks", stocks);
        model.addAttribute("noData", stocks.isEmpty());
        return "stocks/list";
    }

    @GetMapping("/new")
    public String inputForm(Model model) {
        model.addAttribute("input", new StockInputDto());
        return "stocks/input";
    }

    @PostMapping("/new") // POSTにマッピング
    public String create(@ModelAttribute("input") @Validated StockInputDto input,
                         BindingResult bindingResult, Model model) {
        if (bindingResult.hasErrors()) {
            return "stocks/input";
        }

        try {
            stockService.registerNewStock(input.toStock());
        } catch (DataIntegrityViolationException e) {
            bindingResult.rejectValue("ticker", "error.ticker", "銘柄コードはすでに利用されています。");
            return "stocks/input";
        }

        return "redirect:/stocks";
    }
}



==== FILE: /main/java/simplex/bn25/zhao102015/server/controller/StockInputDto.java ====

package simplex.bn25.zhao102015.server.controller;

import org.hibernate.validator.constraints.Range;
import simplex.bn25.zhao102015.server.model.Market;
import simplex.bn25.zhao102015.server.model.Stock;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Pattern;

public final class StockInputDto {

    @NotBlank
    @Pattern(regexp = "[1-9][ACDFGHJKLMNPRSTUWXY0-9][0-9][ACDFGHJKLMNPRSTUWXY0-9]")
    private String ticker;

    @NotBlank
    private String name;

    @NotNull
    private Market exchangeMarket;

    @Range(min = 1, max = 999_999_999_999L)
    private long sharesIssued;

    public String getTicker() {
        return ticker;
    }

    public void setTicker(String ticker) {
        this.ticker = ticker;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Market getExchangeMarket() {
        return exchangeMarket;
    }

    public void setExchangeMarket(Market exchangeMarket) {
        this.exchangeMarket = exchangeMarket;
    }

    public long getSharesIssued() {
        return sharesIssued;
    }

    public void setSharesIssued(long sharesIssued) {
        this.sharesIssued = sharesIssued;
    }

    public Stock toStock() {
        Stock stock = new Stock();
        stock.setTicker(ticker);
        stock.setName(name);
        stock.setExchangeMarket(exchangeMarket);
        stock.setSharesIssued(sharesIssued);
        return stock;
    }
}



==== FILE: /main/java/simplex/bn25/zhao102015/server/controller/TradeController.java ====

package simplex.bn25.zhao102015.server.controller;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.validation.BindingResult;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.*;
import simplex.bn25.zhao102015.server.controller.TradeInputDto;
import simplex.bn25.zhao102015.server.model.Stock;
import simplex.bn25.zhao102015.server.service.StockService;
import simplex.bn25.zhao102015.server.service.TradeService;

import java.util.List;

@Controller
@RequestMapping("/trade")
public class TradeController {

    private final TradeService tradeService;
    private final StockService stockService;

    @Autowired
    public TradeController(TradeService tradeService, StockService stockService) {
        this.tradeService = tradeService;
        this.stockService = stockService;
    }

    @GetMapping
    public String list(Model model) {
        List<?> trades = tradeService.findAll();
        model.addAttribute("trades", trades);
        model.addAttribute("noData", trades.isEmpty());
        return "trades/list";
    }

    @GetMapping("/new")
    public String newTradeForm(@RequestParam("ticker") String ticker, Model model) {
        Stock stock = stockService.findByTicker(ticker);
        TradeInputDto dto = new TradeInputDto();
        dto.setStockId(stock.getId());
        dto.setTicker(stock.getTicker());
        dto.setName(stock.getName());

        model.addAttribute("input", dto);
        return "trades/input";
    }

    @PostMapping("/new")
    public String registerTrade(@Validated @ModelAttribute("input") TradeInputDto input,
                                BindingResult bindingResult, Model model) {
        if (bindingResult.hasErrors()) {
            return "trades/input";
        }

        tradeService.register(input);
        return "redirect:/trade";
    }
}



==== FILE: /main/java/simplex/bn25/zhao102015/server/controller/TradeInputDto.java ====

package simplex.bn25.zhao102015.server.controller;

import jakarta.validation.constraints.*;
import org.springframework.format.annotation.DateTimeFormat;
import simplex.bn25.zhao102015.server.model.Side;

import java.math.BigDecimal;
import java.time.LocalDateTime;

public class TradeInputDto {

    @NotNull
    private Integer stockId;

    private String ticker;
    private String name;

    @NotNull
    @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME)
    private LocalDateTime tradedDatetime;

    @NotNull
    private Side side;

    @NotNull
    @Positive
    private Long quantity;

    @NotNull
    @Digits(integer = 10, fraction = 2)
    private BigDecimal tradedPrice;

    public Integer getStockId() {
        return stockId;
    }

    public void setStockId(Integer stockId) {
        this.stockId = stockId;
    }

    public String getTicker() {
        return ticker;
    }

    public void setTicker(String ticker) {
        this.ticker = ticker;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public LocalDateTime getTradedDatetime() {
        return tradedDatetime;
    }

    public void setTradedDatetime(LocalDateTime tradedDatetime) {
        this.tradedDatetime = tradedDatetime;
    }

    public Side getSide() {
        return side;
    }

    public void setSide(Side side) {
        this.side = side;
    }

    public Long getQuantity() {
        return quantity;
    }

    public void setQuantity(Long quantity) {
        this.quantity = quantity;
    }

    public BigDecimal getTradedPrice() {
        return tradedPrice;
    }

    public void setTradedPrice(BigDecimal tradedPrice) {
        this.tradedPrice = tradedPrice;
    }

    public static TradeInputDto fromStock(simplex.bn25.zhao102015.server.model.Stock stock) {
        TradeInputDto dto = new TradeInputDto();
        dto.setStockId(stock.getId());
        dto.setTicker(stock.getTicker());
        dto.setName(stock.getName());
        return dto;
    }
}



==== FILE: /main/java/simplex/bn25/zhao102015/server/model/Market.java ====

package simplex.bn25.zhao102015.server.model;

public enum Market {
    PRIME,
    STANDARD,
    GROWTH
}



==== FILE: /main/java/simplex/bn25/zhao102015/server/model/Side.java ====

package simplex.bn25.zhao102015.server.model;

public enum Side {
    BUY,
    SELL
}



==== FILE: /main/java/simplex/bn25/zhao102015/server/model/Stock.java ====

package simplex.bn25.zhao102015.server.model;

public class Stock {
    private int id;
    private String ticker;
    private String name;
    private Market exchangeMarket;
    private long sharesIssued;

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }

    public String getTicker() {
        return ticker;
    }

    public void setTicker(String ticker) {
        this.ticker = ticker;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Market getExchangeMarket() {
        return exchangeMarket;
    }

    public void setExchangeMarket(Market exchangeMarket) {
        this.exchangeMarket = exchangeMarket;
    }

    public long getSharesIssued() {
        return sharesIssued;
    }

    public void setSharesIssued(long sharesIssued) {
        this.sharesIssued = sharesIssued;
    }
}



==== FILE: /main/java/simplex/bn25/zhao102015/server/model/Trade.java ====

package simplex.bn25.zhao102015.server.model;

import java.math.BigDecimal;
import java.sql.Timestamp;

public class Trade {
    private int id;
    private Timestamp tradedDatetime;
    private int stockId;
    private Side side;
    private long quantity;
    private BigDecimal tradedPrice;
    private Timestamp createdDatetime;

    private String ticker; // from stock table
    private String name;   // from stock table

    public int getId() { return id; }
    public void setId(int id) { this.id = id; }

    public Timestamp getTradedDatetime() { return tradedDatetime; }
    public void setTradedDatetime(Timestamp tradedDatetime) { this.tradedDatetime = tradedDatetime; }

    public int getStockId() { return stockId; }
    public void setStockId(int stockId) { this.stockId = stockId; }

    public Side getSide() { return side; }
    public void setSide(Side side) { this.side = side; }

    public long getQuantity() { return quantity; }
    public void setQuantity(long quantity) { this.quantity = quantity; }

    public BigDecimal getTradedPrice() { return tradedPrice; }
    public void setTradedPrice(BigDecimal tradedPrice) { this.tradedPrice = tradedPrice; }

    public Timestamp getCreatedDatetime() { return createdDatetime; }
    public void setCreatedDatetime(Timestamp createdDatetime) { this.createdDatetime = createdDatetime; }

    public String getTicker() { return ticker; }
    public void setTicker(String ticker) { this.ticker = ticker; }

    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
}



==== FILE: /main/java/simplex/bn25/zhao102015/server/model/repository/StockRepository.java ====

package simplex.bn25.zhao102015.server.model.repository;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.BeanPropertyRowMapper;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Repository;
import simplex.bn25.zhao102015.server.model.Stock;

import java.util.List;
import java.util.Optional;

@Repository
public class StockRepository {

    private final JdbcTemplate jdbcTemplate;

    @Autowired
    public StockRepository(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    public List<Stock> findAll() {
        String sql = "SELECT * FROM stock ORDER BY id";
        return jdbcTemplate.query(sql, new BeanPropertyRowMapper<>(Stock.class));
    }

    public Optional<Stock> findByTicker(String ticker) {
        String sql = "SELECT * FROM stock WHERE ticker = ?";
        List<Stock> results = jdbcTemplate.query(sql, new BeanPropertyRowMapper<>(Stock.class), ticker);
        return results.stream().findFirst();
    }

    public Stock register(Stock stock) {
        String sql = "INSERT INTO stock (ticker, name, exchange_market, shares_issued, created_datetime) " +
                "VALUES (?, ?, ?, ?, current_timestamp)";
        jdbcTemplate.update(sql, stock.getTicker(), stock.getName(), stock.getExchangeMarket(), stock.getSharesIssued());
        return stock;
    }
}



==== FILE: /main/java/simplex/bn25/zhao102015/server/model/repository/TradeRepository.java ====

package simplex.bn25.zhao102015.server.model.repository;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.RowMapper;
import org.springframework.stereotype.Repository;
import simplex.bn25.zhao102015.server.model.Side;
import simplex.bn25.zhao102015.server.model.Trade;

import java.sql.ResultSet;
import java.sql.SQLException;
import java.util.List;

@Repository
public class TradeRepository {

    private final JdbcTemplate jdbcTemplate;

    @Autowired
    public TradeRepository(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    public List<Trade> findAll() {
        String sql = "SELECT t.*, s.ticker, s.name " +
                "FROM trade t " +
                "JOIN stock s ON t.stock_id = s.id " +
                "ORDER BY t.traded_datetime DESC";

        return jdbcTemplate.query(sql, new TradeRowMapper());
    }

    public int insert(Trade trade) {
        String sql = """
            INSERT INTO trade (traded_datetime, stock_id, side, quantity, traded_price, created_datetime)
            VALUES (?, ?, ?, ?, ?, NOW())
        """;

        return jdbcTemplate.update(sql,
                trade.getTradedDatetime(),
                trade.getStockId(),
                trade.getSide().name(),
                trade.getQuantity(),
                trade.getTradedPrice());
    }

    private static class TradeRowMapper implements RowMapper<Trade> {
        @Override
        public Trade mapRow(ResultSet rs, int rowNum) throws SQLException {
            Trade trade = new Trade();
            trade.setId(rs.getInt("id"));
            trade.setTradedDatetime(rs.getTimestamp("traded_datetime"));
            trade.setStockId(rs.getInt("stock_id"));
            trade.setSide(Side.valueOf(rs.getString("side").toUpperCase()));
            trade.setQuantity(rs.getLong("quantity"));
            trade.setTradedPrice(rs.getBigDecimal("traded_price"));
            trade.setCreatedDatetime(rs.getTimestamp("created_datetime"));
            trade.setTicker(rs.getString("ticker"));
            trade.setName(rs.getString("name"));
            return trade;
        }
    }
}



==== FILE: /main/java/simplex/bn25/zhao102015/server/service/StockService.java ====

package simplex.bn25.zhao102015.server.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import simplex.bn25.zhao102015.server.model.Stock;
import simplex.bn25.zhao102015.server.model.repository.StockRepository;

import java.util.List;

@Service
public class StockService {

    private final StockRepository stockRepository;

    @Autowired
    public StockService(StockRepository stockRepository) {
        this.stockRepository = stockRepository;
    }

    public List<Stock> listAll() {
        return stockRepository.findAll();
    }

    public Stock findByTicker(String ticker) {
        return stockRepository.findByTicker(ticker)
                .orElseThrow(() -> new IllegalArgumentException("銘柄が見つかりません: " + ticker));
    }

    public Stock registerNewStock(Stock stock) {
        return stockRepository.register(stock);
    }
}



==== FILE: /main/java/simplex/bn25/zhao102015/server/service/TradeService.java ====

package simplex.bn25.zhao102015.server.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import simplex.bn25.zhao102015.server.controller.TradeInputDto;
import simplex.bn25.zhao102015.server.model.Trade;
import simplex.bn25.zhao102015.server.model.repository.TradeRepository;

import java.sql.Timestamp;
import java.util.List;

@Service
public class TradeService {

    private final TradeRepository tradeRepository;

    @Autowired
    public TradeService(TradeRepository tradeRepository) {
        this.tradeRepository = tradeRepository;
    }

    public List<Trade> findAll() {
        return tradeRepository.findAll();
    }

    public void register(TradeInputDto input) {
        Trade trade = new Trade();
        trade.setStockId(input.getStockId());
        trade.setTradedDatetime(Timestamp.valueOf(input.getTradedDatetime()));
        trade.setSide(input.getSide());
        trade.setQuantity(input.getQuantity());
        trade.setTradedPrice(input.getTradedPrice());
        tradeRepository.insert(trade);
    }
}



==== FILE: /main/resources/application-local.properties ====

spring.devtools.restart.enabled=false


==== FILE: /main/resources/application.properties ====

spring.datasource.url=jdbc:postgresql://localhost:5432/trading_app_v2
spring.datasource.username=zhao102015
spring.datasource.password=password

spring.thymeleaf.prefix=classpath:/templates/
spring.thymeleaf.suffix=.html



==== FILE: /main/resources/templates/hello.html ====

<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
  <meta charset="UTF-8">
  <title>HelloWorld | trading-app</title>
</head>
<body>
<div th:replace="~{layouts :: header}"></div>

<span th:text="${message}">メッセージがここに展開されます。</span>

</body>
</html>



==== FILE: /main/resources/templates/home.html ====

<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>HelloWorld | trading-app</title>
</head>
<body>
<div th:replace="~{layouts :: header}"></div>

<span th:text="${message}">メッセージがここに展開されます。</span>

</body>
</html>



==== FILE: /main/resources/templates/layouts.html ====

<div th:fragment="header" xmlns:th="http://www.w3.org/1999/xhtml">
    <h1>trading-app-v2 by zhao102015</h1>
    <nav>
        <ul>
            <li><a th:href="@{/}">Home</a></li>
            <li><a th:href="@{/hello}">Hello</a></li>
            <li><a th:href="@{/stocks}">Stocks</a></li>
            <li><a th:href="@{/stocks/new}">Create Stock</a></li>
            <li><a th:href="@{/trade}">Trade List</a></li>
        </ul>
    </nav>
</div>



==== FILE: /main/resources/templates/stocks/input.html ====

<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
  <meta charset="UTF-8">
  <title>銘柄 登録</title>
</head>
<body>
<div th:replace="~{layouts :: header}"></div>

<form method="post" th:object="${input}">
  <div>
    <label for="ticker">Ticker</label>
    <input type="text" id="ticker" th:field="*{ticker}" placeholder="xxxx" />
    <div th:errors="*{ticker}"></div>
  </div>

  <div>
    <label for="name">Name</label>
    <input type="text" id="name" th:field="*{name}" placeholder="Xxx, Inc." />
    <div th:errors="*{name}"></div>
  </div>

  <div>
    <label for="exchangeMarket">Market</label>
    <select id="exchangeMarket" th:field="*{exchangeMarket}">
      <option th:each="market : ${T(simplex.bn25.zhao102015.server.model.Market).values()}"
              th:value="${market}" th:text="${market.name()}"></option>
    </select>
    <div th:errors="*{exchangeMarket}"></div>
  </div>

  <div>
    <label for="sharesIssued">Shares Issued</label>
    <input type="number" id="sharesIssued" th:field="*{sharesIssued}" placeholder="1000000" />
    <div th:errors="*{sharesIssued}"></div>
  </div>

  <div>
    <input type="submit" value="登録" />
  </div>
</form>
</body>
</html>



==== FILE: /main/resources/templates/stocks/list.html ====

<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
  <meta charset="UTF-8">
  <title>銘柄一覧 | trading-app</title>
  <style>
    .cell-right {
        text-align: right;
    }
  </style>
</head>
<body class="container">
<div th:replace="~{layouts :: header}"></div>

<table>
  <thead>
  <tr>
    <th style="text-align: center;">Ticker</th>
    <th style="text-align: center;">Name</th>
    <th style="text-align: center;">Market</th>
    <th style="text-align: center;">Shares Issued</th>
    <th style="text-align: center;">Input Trade</th>
  </tr>
  </thead>
  <tbody>
  <tr th:each="stock : ${stocks}">
    <td style="text-align: center;" th:text="${stock.ticker}"></td>
    <td style="text-align: center;" th:text="${stock.name}"></td>
    <td style="text-align: center;" th:text="${stock.exchangeMarket}"></td>
    <td style="text-align: center;" th:text="${#numbers.formatInteger(stock.sharesIssued, 0, 'COMMA')}"></td>
    <td style="text-align: center;">
      <a th:href="@{/trade/new(ticker=${stock.ticker})}" style="color: black; text-decoration: underline;">
        Link
      </a>
    </td>
  </tr>
  </tbody>
</table>


</body>
</html>



==== FILE: /main/resources/templates/trades/input.html ====

<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
  <meta charset="UTF-8">
  <title>Input Trade</title>
</head>
<body>
<div>
  <h2>Input Trade</h2>
  <form method="post" th:action="@{/trade/new}" th:object="${input}">
    <input type="hidden" th:field="*{stockId}"/>

    <div>
      <label for="ticker">Ticker</label>
      <input type="text" id="ticker" th:value="${input.ticker}" disabled/>
    </div>

    <div>
      <span th:text="${input.name}"></span>
    </div>

    <div>
      <label for="tradedDatetime">Traded Datetime</label>
      <input type="datetime-local" id="tradedDatetime" th:field="*{tradedDatetime}" required/>
      <div th:if="${#fields.hasErrors('tradedDatetime')}" th:errors="*{tradedDatetime}" style="color:red"></div>
    </div>

    <div>
      <label>Side</label>
      <input type="radio" id="buy" th:field="*{side}" value="BUY"/> Buy
      <input type="radio" id="sell" th:field="*{side}" value="SELL"/> Sell
      <div th:if="${#fields.hasErrors('side')}" th:errors="*{side}" style="color:red"></div>
    </div>

    <div>
      <label for="quantity">Quantity</label>
      <input type="number" id="quantity" th:field="*{quantity}" min="1" step="100" required/>
      <div th:if="${#fields.hasErrors('quantity')}" th:errors="*{quantity}" style="color:red"></div>
    </div>

    <div>
      <label for="tradedPrice">Traded Price</label>
      <input type="number" id="tradedPrice" th:field="*{tradedPrice}" step="0.01" required/>
      <div th:if="${#fields.hasErrors('tradedPrice')}" th:errors="*{tradedPrice}" style="color:red"></div>
    </div>

    <div>
      <input type="submit" value="Register"/>
    </div>
  </form>
</div>
</body>
</html>


==== FILE: /main/resources/templates/trades/list.html ====

<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
  <meta charset="UTF-8">
  <title>Trade List</title>
</head>
<body>
<div>
  <h1>trading-app-v2 by zhao102015</h1>
  <nav>
    <ul>
      <li><a href="/">Home</a></li>
      <li><a href="/hello">Hello</a></li>
      <li><a href="/stocks">Stocks</a></li>
      <li><a href="/stocks/new">Create Stock</a></li>
      <li><a href="/trade">Trade List</a></li>
    </ul>
  </nav>
</div>
<div>

  <table>
    <thead>
    <tr>
      <th style="text-align: left;">Traded Datetime</th>
      <th style="text-align: left;">Ticker</th>
      <th style="text-align: left;">Name</th>
      <th style="text-align: left;">Side</th>
      <th style="text-align: right;">Quantity</th>
      <th style="text-align: right;">Traded Price</th>
    </tr>
    </thead>
    <tbody>
    <!-- 无数据时显示提示 -->
    <tr th:if="${#lists.isEmpty(trades)}">
      <td colspan="6" style="text-align: center;">データがありません。</td>
    </tr>

    <!-- 有数据时逐行渲染 -->
    <tr th:each="trade : ${trades}" th:unless="${#lists.isEmpty(trades)}">
      <td th:text="${#dates.format(trade.tradedDatetime, 'yyyy/MM/dd HH:mm')}"></td>
      <td th:text="${trade.ticker}"></td>
      <td th:text="${trade.name}"></td>
      <td th:text="${trade.side}"></td>
      <td th:text="${#numbers.formatInteger(trade.quantity, 0, 'COMMA')}" style="text-align: right;"></td>
      <td th:text="${#numbers.formatDecimal(trade.tradedPrice, 1, 2)}" style="text-align: right;"></td>
    </tr>
    </tbody>
  </table>
</div>
</body>
</html>
