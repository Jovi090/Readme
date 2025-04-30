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
