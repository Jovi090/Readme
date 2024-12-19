package simplex.bn25.zhao335952.trading.view;

import simplex.bn25.zhao335952.trading.model.Position;
import simplex.bn25.zhao335952.trading.repository.StockRepository;

import java.text.DecimalFormat;
import java.util.Map;

public class PositionView {

    private static final DecimalFormat DECIMAL_FORMAT = new DecimalFormat("#,##0.00");

    public void displayHoldings(Map<String, Position> positions, StockRepository stockRepository) {
        System.out.printf("%-10s %-15s %-10s %-15s %-15s %-15s %-15s\n",
                "Ticker", "Product Name", "Quantity", "Avg Price",
                "Valuation", "Unrealized PnL", "Realized PnL");

        for (Map.Entry<String, Position> entry : positions.entrySet()) {
            String ticker = entry.getKey();
            Position position = entry.getValue();
            String productName = stockRepository.getTickerNameByTicker(ticker);

            System.out.printf("%-10s %-15s %,10d %15s %15s %15s %15s\n",
                    ticker,
                    productName,
                    position.getQuantity(),
                    DECIMAL_FORMAT.format(position.getAverageUnitPrice()),
                    DECIMAL_FORMAT.format(position.getValuation()),
                    DECIMAL_FORMAT.format(position.getUnrealizedPnL()),
                    DECIMAL_FORMAT.format(position.getRealizedPnL()));
        }
    }
}

// 修改后的 Position 类
package simplex.bn25.zhao335952.trading.model;

import java.math.BigDecimal;
import java.time.LocalDateTime;

public class Position {
    private int quantity; // 持有数量
    private BigDecimal averageUnitPrice; // 平均取得单价
    private LocalDateTime latestTradeTime; // 最新交易时间
    private BigDecimal realizedPnL; // 实现损益
    private BigDecimal valuation; // 估值
    private BigDecimal unrealizedPnL; // 未实现损益

    public Position() {
        this.quantity = 0;
        this.averageUnitPrice = BigDecimal.ZERO;
        this.latestTradeTime = null;
        this.realizedPnL = BigDecimal.ZERO;
        this.valuation = BigDecimal.ZERO;
        this.unrealizedPnL = BigDecimal.ZERO;
    }

    public int getQuantity() {
        return quantity;
    }

    public void updateQuantity(int deltaQuantity, BigDecimal tradePrice, Side side) {
        if (side == Side.BUY) { // 买入更新平均取得单价
            BigDecimal totalCost = averageUnitPrice.multiply(BigDecimal.valueOf(quantity))
                    .add(tradePrice.multiply(BigDecimal.valueOf(deltaQuantity)));
            quantity += deltaQuantity;
            averageUnitPrice = totalCost.divide(BigDecimal.valueOf(quantity), 2, BigDecimal.ROUND_HALF_UP);
        } else if (side == Side.SELL) { // 卖出更新实现损益
            realizedPnL = realizedPnL.add(tradePrice.subtract(averageUnitPrice).multiply(BigDecimal.valueOf(deltaQuantity)));
            quantity -= deltaQuantity;
        }

        // 保有数量为0时，平均取得单价为0
        if (quantity == 0) {
            averageUnitPrice = BigDecimal.ZERO;
        }
    }

    public BigDecimal getAverageUnitPrice() {
        return averageUnitPrice;
    }

    public LocalDateTime getLatestTradeTime() {
        return latestTradeTime;
    }

    public void setLatestTradeTime(LocalDateTime latestTradeTime) {
        this.latestTradeTime = latestTradeTime;
    }

    public BigDecimal getRealizedPnL() {
        return realizedPnL;
    }

    public void calculateValuation(BigDecimal marketPrice) {
        // 估值用保有数量 × 时价
        this.valuation = marketPrice.multiply(BigDecimal.valueOf(quantity)).abs(); // 确保为正数
        this.unrealizedPnL = valuation.subtract(averageUnitPrice.multiply(BigDecimal.valueOf(quantity)));
    }

    public BigDecimal getValuation() {
        return valuation;
    }

    public BigDecimal getUnrealizedPnL() {
        return unrealizedPnL;
    }
}


private boolean isTradeTimeValid(String ticker, LocalDateTime tradedDatetime) {
    Position position = positions.get(ticker);
    if (position == null || position.getLatestTradeTime() == null) {
        // 如果没有持仓记录，交易时间有效
        return true;
    }
    return tradedDatetime.isAfter(position.getLatestTradeTime());
}

private boolean isQuantityValidAfterTrade(String ticker, int quantity, Side side) {
    if (side == Side.BUY) {
        // 买入交易不需要检查数量
        return true;
    }

    Position position = positions.get(ticker);
    if (position == null || position.getQuantity() < quantity) {
        // 如果没有持仓记录，或者卖出数量超过持仓数量
        return false;
    }

    return true;
}






// 新建 Position 类
package simplex.bn25.zhao335952.trading.model;

import java.math.BigDecimal;
import java.time.LocalDateTime;

public class Position {
    private int quantity; // 持有数量
    private BigDecimal averageUnitPrice; // 平均取得单价
    private LocalDateTime latestTradeTime; // 最新交易时间
    private BigDecimal realizedPnL; // 实现损益
    private BigDecimal valuation; // 估值
    private BigDecimal unrealizedPnL; // 未实现损益

    public Position() {
        this.quantity = 0;
        this.averageUnitPrice = BigDecimal.ZERO;
        this.latestTradeTime = null;
        this.realizedPnL = BigDecimal.ZERO;
        this.valuation = BigDecimal.ZERO;
        this.unrealizedPnL = BigDecimal.ZERO;
    }

    public int getQuantity() {
        return quantity;
    }

    public void updateQuantity(int deltaQuantity, BigDecimal tradePrice, Side side) {
        if (side == Side.BUY) { // 买入更新平均取得单价
            BigDecimal totalCost = averageUnitPrice.multiply(BigDecimal.valueOf(quantity))
                    .add(tradePrice.multiply(BigDecimal.valueOf(deltaQuantity)));
            quantity += deltaQuantity;
            averageUnitPrice = totalCost.divide(BigDecimal.valueOf(quantity), 2, BigDecimal.ROUND_HALF_UP);
        } else if (side == Side.SELL) { // 卖出更新实现损益
            realizedPnL = realizedPnL.add(tradePrice.subtract(averageUnitPrice).multiply(BigDecimal.valueOf(deltaQuantity)));
            quantity -= deltaQuantity;
        }
    }

    public BigDecimal getAverageUnitPrice() {
        return averageUnitPrice;
    }

    public LocalDateTime getLatestTradeTime() {
        return latestTradeTime;
    }

    public void setLatestTradeTime(LocalDateTime latestTradeTime) {
        this.latestTradeTime = latestTradeTime;
    }

    public BigDecimal getRealizedPnL() {
        return realizedPnL;
    }

    public void calculateValuation(BigDecimal marketPrice) {
        this.valuation = marketPrice.multiply(BigDecimal.valueOf(quantity));
        this.unrealizedPnL = valuation.subtract(averageUnitPrice.multiply(BigDecimal.valueOf(quantity)));
    }

    public BigDecimal getValuation() {
        return valuation;
    }

    public BigDecimal getUnrealizedPnL() {
        return unrealizedPnL;
    }
}


以下是完整的 TradeController 类实现，包含了原始功能和新增功能：
package simplex.bn25.zhao335952.trading.controller;

import simplex.bn25.zhao335952.trading.model.Position;
import simplex.bn25.zhao335952.trading.model.Side;
import simplex.bn25.zhao335952.trading.model.Trade;
import simplex.bn25.zhao335952.trading.repository.StockRepository;
import simplex.bn25.zhao335952.trading.repository.TradeRepository;
import simplex.bn25.zhao335952.trading.repository.MarketPriceRepository;
import simplex.bn25.zhao335952.trading.view.PositionView;
import simplex.bn25.zhao335952.trading.view.TradeView;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class TradeController {
    private final TradeRepository tradeRepository;
    private final StockRepository stockRepository;
    private final PositionView positionView;
    private final TradeView tradeView;
    private final MarketPriceRepository marketPriceRepository;
    private final Map<String, Position> positions = new TreeMap<>();

    public TradeController(TradeRepository tradeRepository, StockRepository stockRepository, MarketPriceRepository marketPriceRepository, TradeView tradeView, PositionView positionView) {
        this.tradeRepository = tradeRepository;
        this.stockRepository = stockRepository;
        this.tradeView = tradeView;
        this.positionView = positionView;
        this.marketPriceRepository = marketPriceRepository;
        loadPositions();
    }

    private void loadPositions() {
        List<Trade> trades = tradeRepository.getAllTrades();
        for (Trade trade : trades) {
            String ticker = trade.getTicker();
            Position position = positions.computeIfAbsent(ticker, k -> new Position());

            Side side = Side.valueOf(trade.getSide().toUpperCase());
            position.updateQuantity(trade.getQuantity(), trade.getTradedUnitPrice(), side);
            if (position.getLatestTradeTime() == null || trade.getTradedDatetime().isAfter(position.getLatestTradeTime())) {
                position.setLatestTradeTime(trade.getTradedDatetime());
            }
        }
    }

    public void displayHoldings() {
        if (positions.isEmpty()) {
            System.out.println("保有データが見つかりません。");
            return;
        }

        for (Map.Entry<String, Position> entry : positions.entrySet()) {
            String ticker = entry.getKey();
            Position position = entry.getValue();
            BigDecimal marketPrice = marketPriceRepository.getMarketPrice(ticker);
            position.calculateValuation(marketPrice);
        }

        positionView.displayHoldings(positions, stockRepository);
    }

    public void recordNewTrade() {
        System.out.print("銘柄コードを入力してください: ");
        String ticker = System.console().readLine().trim();

        System.out.print("取引区分（Buy/Sell）を入力してください: ");
        Side side = Side.fromString(System.console().readLine().trim());

        System.out.print("数量を入力してください: ");
        int quantity = Integer.parseInt(System.console().readLine().trim());

        System.out.print("単価を入力してください: ");
        BigDecimal tradedUnitPrice = new BigDecimal(System.console().readLine().trim());

        System.out.print("取引日時を入力してください（yyyy-MM-dd HH:mm）: ");
        LocalDateTime tradedDatetime = LocalDateTime.parse(System.console().readLine().trim());

        if (!isTradeValid(ticker, side, quantity, tradedDatetime)) {
            System.out.println("無効な取引です。");
            return;
        }

        Trade trade = new Trade(tradedDatetime, ticker, null, side.name(), quantity, tradedUnitPrice, LocalDateTime.now());
        tradeRepository.saveTrade(trade);

        Position position = positions.computeIfAbsent(ticker, k -> new Position());
        position.updateQuantity(quantity, tradedUnitPrice, side);
        position.setLatestTradeTime(tradedDatetime);

        System.out.println("取引が登録されました。");
    }

    public void displayAllTrades() {
        List<Trade> trades = tradeRepository.getAllTrades();
        tradeView.displayTradeList(trades);
    }

    private boolean isTradeValid(String ticker, Side side, int quantity, LocalDateTime tradedDatetime) {
        Position position = positions.get(ticker);
        if (position == null) return true;

        if (side == Side.SELL && position.getQuantity() < quantity) {
            System.out.println("売却数量が保有数量を超えています。");
            return false;
        }

        if (position.getLatestTradeTime() != null && tradedDatetime.isBefore(position.getLatestTradeTime())) {
            System.out.println("取引日時が最新取引日時より前です。");
            return false;
        }

        return true;
    }

    public void updateMarketPrice(String ticker, BigDecimal marketPrice) {
        marketPriceRepository.updateMarketPrice(ticker, marketPrice);
    }
}


新建MarketPriceRepository移除MarketPriceGenerator
负责管理市场价格。
// 新建 MarketPriceRepository 类
package simplex.bn25.zhao335952.trading.repository;

import java.io.BufferedReader;
import java.io.FileReader;
import java.io.IOException;
import java.math.BigDecimal;
import java.util.HashMap;
import java.util.Map;

public class MarketPriceRepository {
    private final Map<String, BigDecimal> marketPrices = new HashMap<>();
    private final String csvFilePath;

    public MarketPriceRepository(String csvFilePath) {
        this.csvFilePath = csvFilePath;
        loadMarketPrices();
    }

    public BigDecimal getMarketPrice(String ticker) {
        return marketPrices.getOrDefault(ticker, BigDecimal.ZERO);
    }

    public void updateMarketPrice(String ticker, BigDecimal marketPrice) {
        marketPrices.put(ticker, marketPrice);
    }

    private void loadMarketPrices() {
        try (BufferedReader br = new BufferedReader(new FileReader(csvFilePath))) {
            String line;
            br.readLine(); // Skip header
            while ((line = br.readLine()) != null) {
                String[] parts = line.split(",");
                if (parts.length == 2) {
                    String ticker = parts[0].trim();
                    BigDecimal price = new BigDecimal(parts[1].trim());
                    marketPrices.put(ticker, price);
                }
            }
            System.out.println("市場価格データが正常にロードされました。");
        } catch (IOException e) {
            System.out.println("市場価格データのロード中にエラーが発生しました: " + e.getMessage());
        }
    }
}




PositionView
扩展视图层显示持仓的所有字段。
package simplex.bn25.zhao335952.trading.view;

import simplex.bn25.zhao335952.trading.model.Position;
import simplex.bn25.zhao335952.trading.repository.StockRepository;

import java.util.Map;

public class PositionView {

    public void displayHoldings(Map<String, Position> positions, StockRepository stockRepository) {
        System.out.printf("%-10s %-15s %-10s %-10s %-10s %-10s %-10s\n", 
                          "Ticker", "Product Name", "Quantity", "Avg Price", 
                          "Valuation", "Unrealized PnL", "Realized PnL");

        for (Map.Entry<String, Position> entry : positions.entrySet()) {
            String ticker = entry.getKey();
            Position position = entry.getValue();
            String productName = stockRepository.getTickerNameByTicker(ticker);

            System.out.printf("%-10s %-15s %-10d %-10.2f %-10.2f %-10.2f %-10.2f\n",
                              ticker,
                              productName,
                              position.getQuantity(), //补充下千分位方法类似System.out.printf("%,10d", position.getQuantity());
                              position.getAverageUnitPrice(),
                              position.getValuation(),
                              position.getUnrealizedPnL(),
                              position.getRealizedPnL());
        }
    }
}

MenuView
新增“市场价格更新”和“显示所有交易”的选项。
package simplex.bn25.zhao335952.trading.view;

import java.util.Scanner;

public class MenuView {
    private final Scanner scanner;

    public MenuView() {
        scanner = new Scanner(System.in);
    }

    public String displayMainMenu() {
        System.out.println("操作するメニューを選んでください。");
        System.out.println("1. 銘柄マスタ一覧表示");
        System.out.println("2. 銘柄マスタ新規登録");
        System.out.println("3. 取引新規登録");
        System.out.println("4. 取引一覧表示");
        System.out.println("5. 保有ポジション表示");
        System.out.println("9. アプリケーションを終了する");
        System.out.print("入力してください: ");

        return scanner.nextLine().trim();
    }

    public void close() {
        scanner.close();
    }
}

Main 类
package simplex.bn25.zhao335952.trading;

import simplex.bn25.zhao335952.trading.controller.MainController;
import simplex.bn25.zhao335952.trading.controller.StockController;
import simplex.bn25.zhao335952.trading.controller.TradeController;
import simplex.bn25.zhao335952.trading.repository.MarketPriceRepository;
import simplex.bn25.zhao335952.trading.repository.StockRepository;
import simplex.bn25.zhao335952.trading.repository.TradeRepository;
import simplex.bn25.zhao335952.trading.view.MenuView;
import simplex.bn25.zhao335952.trading.view.PositionView;
import simplex.bn25.zhao335952.trading.view.StockView;
import simplex.bn25.zhao335952.trading.view.TradeView;

public class Main {
    public static void main(String[] args) {
        // 初始化视图
        MenuView menuView = new MenuView();
        StockView stockView = new StockView();
        TradeView tradeView = new TradeView();
        PositionView positionView = new PositionView();

        // 初始化文件路径
        String stockCsvFilePath = "C:/Users/qqq/IdeaProjects/untitled/New code/src/csvファイル/Stock.csv";
        String tradeCsvFilePath = "C:/Users/qqq/IdeaProjects/untitled/New code/src/csvファイル/Trade.csv";
        String marketPriceCsvFilePath = "C:/Users/qqq/IdeaProjects/untitled/New code/src/csvファイル/Market_price.csv";

        // 初始化仓库和相关模块
        StockRepository stockRepository = new StockRepository(stockCsvFilePath);
        TradeRepository tradeRepository = new TradeRepository(tradeCsvFilePath);
        MarketPriceRepository marketPriceRepository = new MarketPriceRepository(marketPriceCsvFilePath);

        // 初始化控制器
        StockController stockController = new StockController(stockRepository, stockView);
        TradeController tradeController = new TradeController(tradeRepository, stockRepository, marketPriceRepository, tradeView, positionView);

        // 创建主控制器并启动
        MainController mainController = new MainController(menuView, stockController, tradeController);
        mainController.start();
    }
}









