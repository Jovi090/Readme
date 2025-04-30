交易录入功能增强：功能2 数量不能超过股票 shares_issued】

此实现确保：用户录入交易时输入的数量不能超过该股票的「発行済株式数 shares_issued」

==================================================
1. StockService.java
路径：src/main/java/simplex/bn25/zhao102015/server/service/StockService.java

// 导入
import simplex.bn25.zhao102015.server.model.Stock;
import java.util.Optional;

public class StockService {

    private final StockRepository stockRepository;

    public StockService(StockRepository stockRepository) {
        this.stockRepository = stockRepository;
    }

    public Stock findById(Integer id) {
        return stockRepository.findById(id)
            .orElseThrow(() -> new IllegalArgumentException("指定された銘柄が見つかりません"));
    }
}

==================================================
2. StockRepository.java
路径：src/main/java/simplex/bn25/zhao102015/server/model/repository/StockRepository.java

// 导入
import simplex.bn25.zhao102015.server.model.Stock;
import org.springframework.jdbc.core.RowMapper;
import java.util.Optional;
import java.sql.ResultSet;
import java.sql.SQLException;

public class StockRepository {

    public Optional<Stock> findById(Integer id) {
        String sql = "SELECT * FROM stock WHERE id = ?";
        return jdbcTemplate.query(sql, new StockRowMapper(), id)
                .stream().findFirst();
    }

    // 新增 RowMapper
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
}

==================================================
3. TradeController.java
路径：src/main/java/simplex/bn25/zhao102015/server/controller/TradeController.java

@PostMapping("/new")
public String create(@ModelAttribute("input") @Validated TradeInputDto input,
                     BindingResult bindingResult,
                     Model model) {

    if (!bindingResult.hasErrors()) {
        Stock stock = stockService.findById(input.getStockId());

        if (input.getQuantity() > stock.getSharesIssued()) {
            bindingResult.rejectValue("quantity", "quantity.tooLarge", "発行済株式数を超えています");
        }
    }

    if (bindingResult.hasErrors()) {
        return "trades/input";
    }

    tradeService.register(input);
    return "redirect:/trade";
}

==================================================
4. trades/input.html 表单页面

确保包含隐藏字段提交 stockId：
<input type="hidden" th:field="*{stockId}" />

==================================================
【小贴士】
- Stock 实体中 sharesIssued 应为 Long 或 Integer 类型。
- TradeInputDto 中 stockId 必须保留在表单中以便查询。
- 异常处理可按需要补充 ResponseStatusException 做 404 显示。






【小贴士】
- Stock 实体中 sharesIssued 应为 Long 或 Integer 类型。
- TradeInputDto 中 stockId 必须保留在表单中以便查询。
- 异常处理可按需要补充 ResponseStatusException 做 404 显示。




交易录入功能增强：功能2 数量不能超过股票 shares_issued】

本说明文档涵盖如何实现以下功能：
▶ 在交易录入页面中，限制“数量”字段不能超过该股票的「発行済株式数 shares_issued」

==================================================
【实现方式】
在 TradeController 中通过 stockId 查询该股票对象，
再与输入的 quantity 进行比较。若超出则加入 BindingResult 错误提示。

==================================================
【步骤一】StockService 中添加 findById 方法
路径：src/main/java/simplex/bn25/zhao102015/server/service/StockService.java

public Stock findById(Integer id) {
    return stockRepository.findById(id)
        .orElseThrow(() -> new IllegalArgumentException("指定された銘柄が見つかりません"));
}

==================================================
【步骤二】StockRepository 中实现 findById 方法
路径：src/main/java/simplex/bn25/zhao102015/server/model/repository/StockRepository.java

public Optional<Stock> findById(Integer id) {
    String sql = "SELECT * FROM stock WHERE id = ?";
    return jdbcTemplate.query(sql, new StockRowMapper(), id)
            .stream().findFirst();
}

※ 注意：你需要已有 StockRowMapper 能正确映射 shares_issued 字段

==================================================
【步骤三】在 TradeController 的 POST 方法中添加判断
路径：src/main/java/simplex/bn25/zhao102015/server/controller/TradeController.java

@PostMapping("/new")
public String create(@ModelAttribute("input") @Validated TradeInputDto input,
                     BindingResult bindingResult, Model model) {

    if (!bindingResult.hasErrors()) {
        Stock stock = stockService.findById(input.getStockId());

        if (input.getQuantity() > stock.getSharesIssued()) {
            bindingResult.rejectValue("quantity", "quantity.tooLarge", "発行済株式数を超えています");
        }
    }

    if (bindingResult.hasErrors()) {
        return "trades/input";
    }

    tradeService.register(input);
    return "redirect:/trade";
}

==================================================
【注意事项】
- TradeInputDto 中 stockId 字段必须从页面传入，建议通过隐藏字段：
  <input type="hidden" th:field="*{stockId}"/>

- sharesIssued 字段类型建议为 Long 或 Integer

- 若股票不存在，StockService 将抛出 IllegalArgumentException，
  可考虑用 ResponseStatusException 替代以显示 404 页面

==================================================
功能完成后，用户在输入超过股票发行数量的交易时将收到红色错误提示。
