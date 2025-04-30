

===== FILE: /mnt/data/extracted_main/main/java/simplex/bn25/zhao102015/server/ServerApplication.java =====

package simplex.bn25.zhao102015.server;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class ServerApplication {
	public static void main(String[] args) {
		SpringApplication.run(ServerApplication.class, args);
	}
}


===== FILE: /mnt/data/extracted_main/main/java/simplex/bn25/zhao102015/server/annotation/MultipleOf100.java =====

package simplex.bn25.zhao102015.server.annotation;

import jakarta.validation.Constraint;
import jakarta.validation.Payload;

import java.lang.annotation.*;

@Documented
@Constraint(validatedBy = MultipleOf100Validator.class)
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
public @interface MultipleOf100 {
    String message() default "100の倍数を入力してください";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

===== FILE: /mnt/data/extracted_main/main/java/simplex/bn25/zhao102015/server/annotation/MultipleOf100Validator.java =====

package simplex.bn25.zhao102015.server.annotation;

import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;

public class MultipleOf100Validator implements ConstraintValidator<MultipleOf100, Long> {
    @Override
    public boolean isValid(Long value, ConstraintValidatorContext context) {
        if (value == null) {
            return true;
        }
        return value %100==0;
    }
}



===== FILE: /mnt/data/extracted_main/main/java/simplex/bn25/zhao102015/server/annotation/TradeDatetimeValidator.java =====

package simplex.bn25.zhao102015.server.annotation;

import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;
import java.time.DayOfWeek;
import java.time.LocalDateTime;
import java.time.LocalTime;

public class TradeDatetimeValidator
        implements ConstraintValidator<ValidTradedDatetime, LocalDateTime> {
    @Override
    public boolean isValid(LocalDateTime value, ConstraintValidatorContext context){
        if(value == null) return true;
        if(value.isAfter(LocalDateTime.now())) return false;
        DayOfWeek day = value.getDayOfWeek();
        if(day == DayOfWeek.SATURDAY || day == DayOfWeek.SUNDAY) return false;

        LocalTime time = value.toLocalTime();
        LocalTime start = LocalTime.of(9,0);
        LocalTime end = LocalTime.of(15,30);
        if(time.isBefore(start) || time.isAfter(end)) return false;

        return true;
    }


}



===== FILE: /mnt/data/extracted_main/main/java/simplex/bn25/zhao102015/server/annotation/ValidTradedDatetime.java =====

package simplex.bn25.zhao102015.server.annotation;


import jakarta.validation.Constraint;
import jakarta.validation.Payload;
import java.lang.annotation.*;

    @Documented
    @Constraint(validatedBy = TradeDatetimeValidator.class)
    @Target({ElementType.FIELD})
    @Retention(RetentionPolicy.RUNTIME)

    public @interface ValidTradedDatetime {
        String message() default "日時は平日9:00~15:30の過去の時間にしてください";
        Class<?>[] groups() default {};
        Class<? extends Payload>[] payload() default {};

    }


===== FILE: /mnt/data/extracted_main/main/java/simplex/bn25/zhao102015/server/annotation/ValidTradeInput.java =====

package simplex.bn25.zhao102015.server.annotation;

import jakarta.validation.Constraint;
import jakarta.validation.Payload;

import java.lang.annotation.*;

@Documented
@Constraint(validatedBy = ValidTradeInputValidator.class)
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface ValidTradeInput {
    String message() default "無効な取引入力です";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}


===== FILE: /mnt/data/extracted_main/main/java/simplex/bn25/zhao102015/server/annotation/ValidTradeInputValidator.java =====

package simplex.bn25.zhao102015.server.annotation;

import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;
import org.springframework.beans.factory.annotation.Autowired;
import simplex.bn25.zhao102015.server.model.Side;
import simplex.bn25.zhao102015.server.model.Trade;
import simplex.bn25.zhao102015.server.model.repository.TradeRepository;
import simplex.bn25.zhao102015.server.controller.TradeInputDto;

import java.sql.Timestamp;
import java.util.List;

public class ValidTradeInputValidator implements ConstraintValidator<ValidTradeInput, TradeInputDto> {

    @Autowired
    private TradeRepository tradeRepository;

    @Override
    public boolean isValid(TradeInputDto input, ConstraintValidatorContext context) {
        if (input.getStockId() == null || input.getTradedDatetime() == null
                || input.getQuantity() == null || input.getSide() == null) {
            return true; // 有空字段，跳过校验
        }

        List<Trade> trades = tradeRepository.findAllByStockIdOrderByTradedDatetimeAsc(input.getStockId());

        // 计算在当前交易时间之前的保有数
        long currentHoldings = 0L;
        for (Trade trade : trades) {
            if (trade.getTradedDatetime().after(Timestamp.valueOf(input.getTradedDatetime()))) {
                break;
            }
            currentHoldings += trade.getSide() == Side.BUY
                    ? trade.getQuantity()
                    : -trade.getQuantity();
        }

        long delta = input.getSide() == Side.BUY
                ? input.getQuantity()
                : -input.getQuantity();

        boolean validHoldings = (currentHoldings + delta) >= 0;

        boolean validDatetime = trades.isEmpty()
                || trades.get(trades.size() - 1).getTradedDatetime()
                .compareTo(Timestamp.valueOf(input.getTradedDatetime())) < 0;

        // 开始构建错误提示
        context.disableDefaultConstraintViolation();

        if (!validHoldings) {
            context.buildConstraintViolationWithTemplate("取引後の保有数がマイナスになります。")
                    .addPropertyNode("quantity")
                    .addConstraintViolation();
        }

        if (!validDatetime) {
            context.buildConstraintViolationWithTemplate("既存の取引より前の日時は指定できません。")
                    .addPropertyNode("tradedDatetime")
                    .addConstraintViolation();
        }

        return validHoldings && validDatetime;
    }
}


===== FILE: /mnt/data/extracted_main/main/java/simplex/bn25/zhao102015/server/annotation/WithinSharesIssued.java =====

package simplex.bn25.zhao102015.server.annotation;

import jakarta.validation.Constraint;
import jakarta.validation.Payload;

import java.lang.annotation.*;

@Documented
@Constraint(validatedBy = WithinSharesIssuedValidator.class)
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface WithinSharesIssued {
    String message() default "発行済株式数を超えています";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}


===== FILE: /mnt/data/extracted_main/main/java/simplex/bn25/zhao102015/server/annotation/WithinSharesIssuedValidator.java =====

package simplex.bn25.zhao102015.server.annotation;

import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import simplex.bn25.zhao102015.server.model.Stock;
import simplex.bn25.zhao102015.server.model.repository.StockRepository;
import simplex.bn25.zhao102015.server.controller.TradeInputDto;
import simplex.bn25.zhao102015.server.service.StockService;

@Component
public class WithinSharesIssuedValidator implements ConstraintValidator<WithinSharesIssued, TradeInputDto> {

    @Autowired
    private StockService stockService;

    @Override
    public boolean isValid(TradeInputDto dto, ConstraintValidatorContext context) {
        if (dto.getStockId() == null || dto.getQuantity() == null) {
            return true;
        }

        Stock stock = stockService.findById(dto.getStockId());
        if (stock == null) {
            return true;
        }

        boolean valid = dto.getQuantity() <= stock.getSharesIssued();
        if (!valid) {
            context.disableDefaultConstraintViolation();
            context.buildConstraintViolationWithTemplate(context.getDefaultConstraintMessageTemplate())
                    .addPropertyNode("quantity")
                    .addConstraintViolation();
        }

        return valid;
    }
}

===== FILE: /mnt/data/extracted_main/main/java/simplex/bn25/zhao102015/server/controller/HelloController.java =====

package simplex.bn25.zhao102015.server.controller;

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

===== FILE: /mnt/data/extracted_main/main/java/simplex/bn25/zhao102015/server/controller/HomeController.java =====

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


===== FILE: /mnt/data/extracted_main/main/java/simplex/bn25/zhao102015/server/controller/StockController.java =====

package simplex.bn25.zhao102015.server.controller;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.dao.DataIntegrityViolationException;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.ModelAttribute;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import simplex.bn25.zhao102015.server.service.StockService;
import simplex.bn25.zhao102015.server.model.Stock;
import org.springframework.validation.annotation.Validated;
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


===== FILE: /mnt/data/extracted_main/main/java/simplex/bn25/zhao102015/server/controller/StockInputDto.java =====

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


===== FILE: /mnt/data/extracted_main/main/java/simplex/bn25/zhao102015/server/controller/TradeController.java =====

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
    public String newTradeForm(@RequestParam(value = "ticker", required = false) String ticker, Model model) {
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
            model.addAttribute("input", input);
            return "trades/input";
        }

        tradeService.register(input);
        return "redirect:/trade";
    }

}


===== FILE: /mnt/data/extracted_main/main/java/simplex/bn25/zhao102015/server/controller/TradeInputDto.java =====

package simplex.bn25.zhao102015.server.controller;

import jakarta.validation.constraints.*;
import org.hibernate.validator.constraints.Range;
import org.springframework.format.annotation.DateTimeFormat;
import simplex.bn25.zhao102015.server.annotation.MultipleOf100;
import simplex.bn25.zhao102015.server.annotation.ValidTradeInput;
import simplex.bn25.zhao102015.server.annotation.ValidTradedDatetime;
import simplex.bn25.zhao102015.server.annotation.WithinSharesIssued;
import simplex.bn25.zhao102015.server.model.Side;
import simplex.bn25.zhao102015.server.model.Stock;

import java.math.BigDecimal;
import java.time.LocalDateTime;

@ValidTradeInput
@WithinSharesIssued
public class TradeInputDto {

    @NotNull
    private Integer stockId;

    private String ticker;
    private String name;

    @NotNull(message = "入力してください")
    @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME)
    @ValidTradedDatetime
    private LocalDateTime tradedDatetime;

    @NotNull(message = "入力してください")
    private Side side;

    @NotNull(message = "入力してください")
    @Positive(message = "0より大きい数値を入力してください")
    @MultipleOf100
    @Digits(integer = 12, fraction = 0, message = "12桁以下の整数を入力してください")
    private Long quantity;

    @NotNull(message = "入力してください")
    @Positive(message = "0より大きい数値を入力してください")
    @Digits(integer = 10, fraction = 2, message = "小数点以下2桁以内、999,999,999,999以下の数値を入力してください")
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

    public static TradeInputDto fromStock(Stock stock) {
        TradeInputDto dto = new TradeInputDto();
        dto.setStockId(stock.getId());
        dto.setTicker(stock.getTicker());
        dto.setName(stock.getName());
        return dto;
    }
}


===== FILE: /mnt/data/extracted_main/main/java/simplex/bn25/zhao102015/server/model/Market.java =====

package simplex.bn25.zhao102015.server.model;

public enum Market {
    PRIME,
    STANDARD,
    GROWTH
}


===== FILE: /mnt/data/extracted_main/main/java/simplex/bn25/zhao102015/server/model/Side.java =====

package simplex.bn25.zhao102015.server.model;

public enum Side {
    BUY,
    SELL
}


===== FILE: /mnt/data/extracted_main/main/java/simplex/bn25/zhao102015/server/model/Stock.java =====

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


===== FILE: /mnt/data/extracted_main/main/java/simplex/bn25/zhao102015/server/model/Trade.java =====

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


===== FILE: /mnt/data/extracted_main/main/java/simplex/bn25/zhao102015/server/model/repository/StockRepository.java =====

package simplex.bn25.zhao102015.server.model.repository;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.BeanPropertyRowMapper;
import org.springframework.jdbc.core.DataClassRowMapper;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.RowMapper;
import org.springframework.stereotype.Repository;
import simplex.bn25.zhao102015.server.model.Market;
import simplex.bn25.zhao102015.server.model.Stock;

import java.sql.ResultSet;
import java.sql.SQLException;
import java.util.List;
import java.util.Optional;

@Repository
public class StockRepository {

    private final JdbcTemplate jdbcTemplate;
    private final RowMapper<Stock> rowMapper = new DataClassRowMapper<>(Stock.class);

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

    public void register(Stock stock) {
        this.jdbcTemplate.query(
                "INSERT INTO stock (ticker, name, exchange_market, shares_issued) VALUES (?, ?, ?, ?) RETURNING *",
                rowMapper,
                stock.getTicker(),
                stock.getName(),
                stock.getExchangeMarket().name(),
                stock.getSharesIssued()
        );
    }

    public Optional<Stock> findById(Integer id) {
        String sql = "SELECT * FROM stock WHERE id = ?";
        return jdbcTemplate.query(sql, new StockRowMapper(), id).stream().findFirst();
    }

    private static class StockRowMapper implements RowMapper<Stock> {
        @Override
        public Stock mapRow(ResultSet rs, int rowNum) throws SQLException {
            Stock stock = new Stock();
            stock.setId(rs.getInt("id"));
            stock.setTicker(rs.getString("ticker"));
            stock.setName(rs.getString("name"));
            stock.setExchangeMarket(Market.valueOf(rs.getString("exchange_market")));
            stock.setSharesIssued(rs.getLong("shares_issued"));
            return stock;
        }
    }
}

===== FILE: /mnt/data/extracted_main/main/java/simplex/bn25/zhao102015/server/model/repository/TradeRepository.java =====

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

    public  List<Trade> findAllByStockIdOrderByTradedDatetimeAsc(int stockId){
        String sql = "SELECT t.*, s.ticker, s.name FROM trade t JOIN stock s ON t.stock_id = s.id " +
                "WHERE t.stock_id = ? ORDER BY t.traded_datetime ASC";
        return jdbcTemplate.query(sql, new TradeRowMapper(), stockId);
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


===== FILE: /mnt/data/extracted_main/main/java/simplex/bn25/zhao102015/server/service/StockService.java =====

package simplex.bn25.zhao102015.server.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Service;
import org.springframework.web.server.ResponseStatusException;
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
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND, "No Ticker Found"));
    }

    public Stock findById(Integer id) {
        return stockRepository.findById(id)
                .orElseThrow(() -> new IllegalArgumentException("銘柄が見つかりません"));
    }

    public void registerNewStock(Stock stock) { stockRepository.register(stock); }
}


===== FILE: /mnt/data/extracted_main/main/java/simplex/bn25/zhao102015/server/service/TradeService.java =====

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


===== FILE: /mnt/data/extracted_main/main/resources/templates/home.html =====

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


===== FILE: /mnt/data/extracted_main/main/resources/templates/layouts.html =====

<div th:fragment="header" xmlns:th="http://www.thymeleaf.org">
    <h1>trading-app-v2 by zhao102015</h1>
    <nav>
        <ul>
            <li><a th:href="@{/}">Home</a></li>
            <li><a th:href="@{/stocks}">Stocks</a></li>
            <li><a th:href="@{/stocks/new}">Create Stock</a></li>
            <li><a th:href="@{/trade}">Trade List</a></li>
        </ul>
    </nav>
</div>


===== FILE: /mnt/data/extracted_main/main/resources/templates/stocks/input.html =====

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


===== FILE: /mnt/data/extracted_main/main/resources/templates/stocks/list.html =====

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

<table border="1" width="700">
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

  <tr th:if="${#lists.isEmpty(stocks)}">
    <td colspan="6" style="text-align: center;">データがありません。</td>
  </tr>

  <tr th:each="stock : ${stocks}">
    <td style="text-align: center;" th:text="${stock.ticker}"></td>
    <td style="text-align: center;" th:text="${stock.name}"></td>
    <td style="text-align: center;" th:text="${stock.exchangeMarket}"></td>
    <td style="text-align: right;" th:text="${#numbers.formatInteger(stock.sharesIssued, 0, 'COMMA')}"></td>
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


===== FILE: /mnt/data/extracted_main/main/resources/templates/trades/input.html =====

<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
  <meta charset="UTF-8">
  <title>Input Trade</title>
</head>
<body>
<div>
  <h2>Input Trade</h2>
  <form novalidate method="post" th:object="${input}">
    <input type="hidden" th:field="*{stockId}"/>
    <input type="hidden" th:field="*{name}"/>

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
      <input type="number" id="quantity" th:field="*{quantity}" step="100" required/>
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

===== FILE: /mnt/data/extracted_main/main/resources/templates/trades/list.html =====

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
      <li><a href="/stocks">Stocks</a></li>
      <li><a href="/stocks/new">Create Stock</a></li>
      <li><a href="/trade">Trade List</a></li>
    </ul>
  </nav>
</div>
<div>

  <table border="1" width="1000">
    <thead>
    <tr>
      <th style="text-align: center;">Traded Datetime</th>
      <th style="text-align: center;">Ticker</th>
      <th style="text-align: center;">Name</th>
      <th style="text-align: center;">Side</th>
      <th style="text-align: center;">Quantity</th>
      <th style="text-align: center;">Traded Price</th>
    </tr>
    </thead>
    <tbody>

    <tr th:if="${#lists.isEmpty(trades)}">
      <td colspan="6" style="text-align: center;">データがありません。</td>
    </tr>


    <tr th:each="trade : ${trades}" th:unless="${#lists.isEmpty(trades)}">
      <td th:text="${#dates.format(trade.tradedDatetime, 'yyyy/MM/dd HH:mm')}" style="text-align: center;" ></td>
      <td th:text="${trade.ticker}" style="text-align: center;" ></td>
      <td th:text="${trade.name}" style="text-align: center;" ></td>
      <td th:text="${#strings.capitalize(trade.side.name().toLowerCase)}"  style="text-align: center;"></td>
      <td th:text="${#numbers.formatInteger(trade.quantity, 0, 'COMMA')}" style="text-align: right;"></td>
      <td th:text="${#numbers.formatDecimal(trade.tradedPrice, 1, 'COMMA', 2, 'POINT')}" style="text-align: right;"></td>
    </tr>
    </tbody>
  </table>
</div>
</body>
</html>

