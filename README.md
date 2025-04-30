引入力バリデーション機能：保有株数チェック + 時系列チェック（注解方式）】

目的：
1. 買/売によって保有数がマイナスになる取引を禁止する
2. 既存取引より未来の日時での新規登録を禁止する

===========================================
1. 创建注解 @ValidTradeInput
===========================================
路径：src/main/java/simplex/bn25/zhao102015/server/annotation/ValidTradeInput.java

package simplex.bn25.zhao102015.server.annotation;

import jakarta.validation.Constraint;
import jakarta.validation.Payload;
import java.lang.annotation.*;

@Documented
@Constraint(validatedBy = simplex.bn25.zhao102015.server.annotation.ValidTradeInputValidator.class)
@Target({ ElementType.TYPE })
@Retention(RetentionPolicy.RUNTIME)
public @interface ValidTradeInput {
    String message() default "無効な取引入力です";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

===========================================
2. 创建校验器 ValidTradeInputValidator.java
===========================================
路径：src/main/java/simplex/bn25/zhao102015/server/annotation/ValidTradeInputValidator.java

package simplex.bn25.zhao102015.server.annotation;

import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import simplex.bn25.zhao102015.server.controller.TradeInputDto;
import simplex.bn25.zhao102015.server.model.Side;
import simplex.bn25.zhao102015.server.model.Trade;
import simplex.bn25.zhao102015.server.model.repository.TradeRepository;

import java.util.List;

@Component
public class ValidTradeInputValidator implements ConstraintValidator<ValidTradeInput, TradeInputDto> {

    @Autowired
    private TradeRepository tradeRepository;

    @Override
    public boolean isValid(TradeInputDto input, ConstraintValidatorContext context) {
        if (input.getStockId() == null || input.getTradedDatetime() == null || input.getQuantity() == null || input.getSide() == null) {
            return true;
        }

        List<Trade> trades = tradeRepository.findAllByStockIdOrderByTradedDatetimeAsc(input.getStockId());

        long currentHoldings = 0L;
        for (Trade trade : trades) {
            if (trade.getTradedDatetime().isAfter(input.getTradedDatetime())) break;
            currentHoldings += trade.getSide() == Side.BUY ? trade.getQuantity() : -trade.getQuantity();
        }

        long delta = input.getSide() == Side.BUY ? input.getQuantity() : -input.getQuantity();
        boolean validHoldings = (currentHoldings + delta) >= 0;

        boolean validDatetime = trades.stream()
            .noneMatch(t -> !t.getTradedDatetime().isBefore(input.getTradedDatetime()));

        if (!validHoldings) {
            context.disableDefaultConstraintViolation();
            context.buildConstraintViolationWithTemplate("取引により保有数がマイナスになります")
                   .addPropertyNode("quantity").addConstraintViolation();
        }

        if (!validDatetime) {
            context.disableDefaultConstraintViolation();
            context.buildConstraintViolationWithTemplate("既存取引より過去の日付を指定してください")
                   .addPropertyNode("tradedDatetime").addConstraintViolation();
        }

        return validHoldings && validDatetime;
    }
}

===========================================
3. DTO 添加注解

路径：src/main/java/simplex/bn25/zhao102015/server/controller/TradeInputDto.java

@ValidTradeInput
public class TradeInputDto {
    ...
}

===========================================
4. TradeRepository.java 追加方法

public List<Trade> findAllByStockIdOrderByTradedDatetimeAsc(int stockId) {
    String sql = "SELECT t.*, s.ticker, s.name FROM trade t JOIN stock s ON t.stock_id = s.id " +
                 "WHERE t.stock_id = ? ORDER BY t.traded_datetime ASC";
    return jdbcTemplate.query(sql, new TradeRowMapper(), stockId);
}

===========================================
错误提示将会在：







【交易录入功能增强：功能2（基于 Annotation 实现）】

目的：防止用户录入的交易数量超过股票的発行済株式数（shares_issued）

===========================================
1. 创建注解 @WithinSharesIssued
===========================================
路径：src/main/java/simplex/bn25/zhao102015/server/annotation/WithinSharesIssued.java

package simplex.bn25.zhao102015.server.annotation;

import jakarta.validation.Constraint;
import jakarta.validation.Payload;

import java.lang.annotation.*;

@Documented
@Constraint(validatedBy = simplex.bn25.zhao102015.server.annotation.WithinSharesIssuedValidator.class)
@Target({ ElementType.TYPE })
@Retention(RetentionPolicy.RUNTIME)
public @interface WithinSharesIssued {
    String message() default "発行済株式数を超えています";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

===========================================
2. 创建校验器 WithinSharesIssuedValidator
===========================================
路径：src/main/java/simplex/bn25/zhao102015/server/annotation/WithinSharesIssuedValidator.java

package simplex.bn25.zhao102015.server.annotation;

import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import simplex.bn25.zhao102015.server.model.Stock;
import simplex.bn25.zhao102015.server.service.StockService;
import simplex.bn25.zhao102015.server.controller.TradeInputDto;

@Component
public class WithinSharesIssuedValidator implements ConstraintValidator<WithinSharesIssued, TradeInputDto> {

    @Autowired
    private StockService stockService;

    @Override
    public boolean isValid(TradeInputDto dto, ConstraintValidatorContext context) {
        if (dto.getStockId() == null || dto.getQuantity() == null) {
            return true;  // 其他验证器会处理这些错误
        }

        Stock stock = stockService.findById(dto.getStockId());
        if (stock == null) {
            return true; // 避免 null 异常
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

===========================================
3. 修改 TradeInputDto.java
===========================================
路径：src/main/java/simplex/bn25/zhao102015/server/controller/TradeInputDto.java

在类上添加注解：

@WithinSharesIssued
public class TradeInputDto {
    ...
}

===========================================
4. 补充：StockService.java 添加 findById 方法
===========================================
路径：src/main/java/simplex/bn25/zhao102015/server/service/StockService.java

public Stock findById(Integer id) {
    return stockRepository.findById(id)
            .orElseThrow(() -> new IllegalArgumentException("銘柄が見つかりません: ID=" + id));
}

===========================================
5. 补充：StockRepository.java 添加 findById 方法与 RowMapper
===========================================
路径：src/main/java/simplex/bn25/zhao102015/server/model/repository/StockRepository.java

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
        stock.setExchangeMarket(rs.getString("exchange_market"));
        stock.setSharesIssued(rs.getLong("shares_issued"));
        stock.setCreatedDatetime(rs.getTimestamp("created_datetime"));
        return stock;
    }
}

===========================================
6. Controller 中无需再手动检查 shares_issued

TradeController.java 中只需保留标准的 BindingResult 检查即可：

if (bindingResult.hasErrors()) {
    model.addAttribute("input", input);
    return "trades/input";
}

tradeService.register(input);
return "redirect:/trade";

===========================================
校验失败时，将在数量字段下方以红字提示：“発行済株式数を超えています”




交易录入功能增强：功能2（基于 Annotation 实现）】

目的：防止用户录入的交易数量超过股票的発行済株式数（shares_issued）

===========================================
1. 创建注解 @WithinSharesIssued
===========================================
路径：src/main/java/simplex/bn25/zhao102015/server/annotation/WithinSharesIssued.java

package simplex.bn25.zhao102015.server.annotation;

import jakarta.validation.Constraint;
import jakarta.validation.Payload;

import java.lang.annotation.*;

@Documented
@Constraint(validatedBy = simplex.bn25.zhao102015.server.annotation.WithinSharesIssuedValidator.class)
@Target({ ElementType.TYPE })
@Retention(RetentionPolicy.RUNTIME)
public @interface WithinSharesIssued {
    String message() default "発行済株式数を超えています";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

===========================================
2. 创建校验器 WithinSharesIssuedValidator
===========================================
路径：src/main/java/simplex/bn25/zhao102015/server/annotation/WithinSharesIssuedValidator.java

package simplex.bn25.zhao102015.server.annotation;

import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import simplex.bn25.zhao102015.server.model.Stock;
import simplex.bn25.zhao102015.server.service.StockService;
import simplex.bn25.zhao102015.server.controller.TradeInputDto;

@Component
public class WithinSharesIssuedValidator implements ConstraintValidator<WithinSharesIssued, TradeInputDto> {

    @Autowired
    private StockService stockService;

    @Override
    public boolean isValid(TradeInputDto dto, ConstraintValidatorContext context) {
        if (dto.getStockId() == null || dto.getQuantity() == null) {
            return true;  // 其他验证器会处理这些错误
        }

        Stock stock = stockService.findById(dto.getStockId());
        if (stock == null) {
            return true; // 避免 null 异常
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

===========================================
3. 修改 TradeInputDto.java
===========================================
路径：src/main/java/simplex/bn25/zhao102015/server/controller/TradeInputDto.java

在类上添加注解：

@WithinSharesIssued
public class TradeInputDto {
    ...
}

===========================================
4. 依赖说明

- StockService 必须存在 findById 方法：
  public Stock findById(Integer id)

- shares_issued 类型建议为 Long

- Spring Boot 会自动注入 validator，无需额外配置

===========================================
5. 不再需要在 Controller 中手动检查数量与 sharesIssued 的关系

TradeController.java 中只需保留标准的 BindingResult 检查即可：

if (bindingResult.hasErrors()) {
    model.addAttribute("input", input);
    return "trades/input";
}

tradeService.register(input);
return "redirect:/trade";

===========================================
校验失败时，将在数量字段下方以红字提示：“発行済株式数を超えています”
