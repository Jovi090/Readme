以下是完整的 CalculatePosition 类，用来计算股票的 保有数量、平均取得单价、实现损益 和 未实现损益（评价損益）。
package simplex.bn25.zhao335952.trading.utils;

import simplex.bn25.zhao335952.trading.model.Trade;
import simplex.bn25.zhao335952.trading.model.repository.TradeRepository;
import simplex.bn25.zhao335952.trading.model.repository.StockRepository;

import java.math.BigDecimal;
import java.math.RoundingMode;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

/**
 * 计算股票的保有数量、平均取得单价、实现损益、市场价值和未实现损益的类。
 */
public class CalculatePosition {
    private final TradeRepository tradeRepository;  // 交易数据仓库
    private final StockRepository stockRepository; // 股票数据仓库
    private final Map<String, PositionData> positionDataMap; // 每只股票的计算结果

    public CalculatePosition(TradeRepository tradeRepository, StockRepository stockRepository) {
        this.tradeRepository = tradeRepository;
        this.stockRepository = stockRepository;
        this.positionDataMap = new HashMap<>();
    }

    /**
     * 计算股票的保有信息，包括：
     * - 保有数量（holdingQuantity）
     * - 平均取得单价（averageUnitPrice）
     * - 实现损益（realizedPnL）
     * - 市场价值（marketValue）
     * - 未实现损益（unrealizedPnL）
     */
    public void calculatePositions() {
        positionDataMap.clear(); // 清空之前的计算结果

        List<Trade> trades = tradeRepository.getAllTrades(); // 获取所有交易记录

        // 遍历每一条交易记录
        for (Trade trade : trades) {
            String ticker = trade.getTicker(); // 股票代码
            BigDecimal unitPrice = trade.getTradedUnitPrice(); // 交易单价
            int quantity = trade.getQuantity(); // 交易数量
            boolean isSell = "Sell".equalsIgnoreCase(trade.getSide()); // 判断交易类型

            // 如果不存在该股票的记录，则初始化
            if (!positionDataMap.containsKey(ticker)) {
                positionDataMap.put(ticker, new PositionData());
            }

            // 获取该股票的当前保有信息
            PositionData position = positionDataMap.get(ticker);

            if (isSell) {
                // 处理卖出交易
                int sellQuantity = quantity;

                // 计算实现损益： (卖出价格 - 平均取得单价) * 卖出数量
                BigDecimal realizedPnL = unitPrice.subtract(position.averageUnitPrice)
                        .multiply(BigDecimal.valueOf(sellQuantity));
                position.realizedPnL = position.realizedPnL.add(realizedPnL);

                // 更新保有数量
                position.holdingQuantity -= sellQuantity;
            } else {
                // 处理买入交易
                BigDecimal totalCost = position.averageUnitPrice
                        .multiply(BigDecimal.valueOf(position.holdingQuantity))
                        .add(unitPrice.multiply(BigDecimal.valueOf(quantity)));

                // 更新保有数量
                position.holdingQuantity += quantity;

                // 计算新的平均取得单价： 总成本 / 总数量
                position.averageUnitPrice = totalCost.divide(
                        BigDecimal.valueOf(position.holdingQuantity), 2, RoundingMode.HALF_UP);
            }
        }

        // 计算市场价值和未实现损益
        for (Map.Entry<String, PositionData> entry : positionDataMap.entrySet()) {
            String ticker = entry.getKey();
            PositionData position = entry.getValue();

            // 获取股票的市场价格（实时价格）
            BigDecimal marketPrice = stockRepository.getMarketPriceByTicker(ticker);

            // 市场价值 = 当前保有数量 * 市场价格
            position.marketValue = marketPrice.multiply(BigDecimal.valueOf(position.holdingQuantity));

            // 未实现损益 = 市场价值 - (平均取得单价 * 保有数量)
            BigDecimal totalCost = position.averageUnitPrice
                    .multiply(BigDecimal.valueOf(position.holdingQuantity));
            position.unrealizedPnL = position.marketValue.subtract(totalCost);
        }
    }

    /**
     * 获取所有股票的保有信息计算结果。
     */
    public Map<String, PositionData> getPositionData() {
        return positionDataMap;
    }

    /**
     * 内部类：用于存储每只股票的保有信息。
     */
    public static class PositionData {
        public int holdingQuantity = 0; // 保有数量
        public BigDecimal averageUnitPrice = BigDecimal.ZERO; // 平均取得单价
        public BigDecimal realizedPnL = BigDecimal.ZERO; // 实现损益
        public BigDecimal marketValue = BigDecimal.ZERO; // 市场价值
        public BigDecimal unrealizedPnL = BigDecimal.ZERO; // 未实现损益
    }
}


private boolean isQuantityValidAfterTrade(String ticker, String side, int quantity) {
    int currentQuantity = holdings.getOrDefault(ticker, 0);
    int delta; // 表示数量的增减变化

    if ("Sell".equalsIgnoreCase(side)) {
        delta = -quantity; // 如果是卖出，数量取负值
    } else {
        delta = quantity; // 如果是买入，数量为正值
    }

    int newQuantity = currentQuantity + delta; // 计算交易后的持仓量
    return newQuantity >= 0; // 判断持仓量是否为负
}


private void updateSharedData() {
    holdings.clear();
    latestTradeTimes.clear();

    List<Trade> trades = tradeRepository.getAllTrades();

    // 计算每只股票的持仓量和最晚交易时间
    for (Trade trade : trades) {
        // 更新持仓量
        int quantity = trade.getQuantity();
        if ("Sell".equalsIgnoreCase(trade.getSide())) {
            quantity = -quantity; // 卖出为负数
        }
        holdings.merge(trade.getTicker(), quantity, Integer::sum);

        // 更新最晚交易时间（简化逻辑）
        LocalDateTime currentLatestTime = latestTradeTimes.get(trade.getTicker());
        if (currentLatestTime == null || trade.getTradedDatetime().isAfter(currentLatestTime)) {
            latestTradeTimes.put(trade.getTicker(), trade.getTradedDatetime());
        }
    }
}
2. 更新 PositionView 类

PositionView 负责显示保有信息，包括保有数量、平均取得单价、实现損益和评价損益。

package simplex.bn25.zhao335952.trading.view;

import simplex.bn25.zhao335952.trading.utils.CalculatePosition.PositionData;
import simplex.bn25.zhao335952.trading.model.repository.StockRepository;

import java.util.List;

public class PositionView {

    public void displayPositions(List<PositionData> positionDataList, StockRepository stockRepository) {
        System.out.println("-------------------------------------------------------------------------------");
        System.out.printf("| %-6s | %-30s | %10s | %15s | %15s |\n",
                "Ticker", "Product Name", "Quantity", "Avg Price", "Realized PnL");
        System.out.println("-------------------------------------------------------------------------------");

        for (PositionData data : positionDataList) {
            String ticker = data.ticker;
            String productName = stockRepository.getTickerNameByTicker(ticker);
            System.out.printf("| %-6s | %-30s | %10d | %15.2f | %15.2f |\n",
                    ticker,
                    productName,
                    data.holdingQuantity,
                    data.averageUnitPrice,
                    data.realizedPnL);
        }

        System.out.println("-------------------------------------------------------------------------------");
    }
}


3. 在 TradeController 中调用

在 TradeController 中添加对 CalculatePosition 和 PositionView 的调用：

public void displayPositions() {
    CalculatePosition calculatePosition = new CalculatePosition(tradeRepository, stockRepository);
    calculatePosition.calculatePositions();
    List<CalculatePosition.PositionData> positionData = calculatePosition.getPositionData();
    positionView.displayPositions(positionData, stockRepository);
}


以下是实现一个新类来生成股票的实时价格 CSV 的完整方案，并满足两个要求：

程序启动时：根据 Trade.csv 自动生成实时价格文件。
运行程序功能3后：录入新的交易后，实时更新价格文件。

实现步骤
创建 MarketPriceGenerator 类：
读取 Trade.csv：获取每只股票的最新交易价格。
生成实时价格文件：输出为 market_price.csv，包含 Ticker 和 Market_price 两列。
修改 Main 类：
在程序启动时调用 MarketPriceGenerator 生成初始价格文件。
在 TradeController 中：
在功能3（recordNewTrade）执行完成后，调用 MarketPriceGenerator 更新价格文件。
package simplex.bn25.zhao335952.trading.utils;

import simplex.bn25.zhao335952.trading.model.Trade;
import simplex.bn25.zhao335952.trading.model.repository.TradeRepository;

import java.io.FileWriter;
import java.io.IOException;
import java.math.BigDecimal;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class MarketPriceGenerator {
    private final TradeRepository tradeRepository;
    private final String marketPriceCsvFilePath;

    public MarketPriceGenerator(TradeRepository tradeRepository, String marketPriceCsvFilePath) {
        this.tradeRepository = tradeRepository;
        this.marketPriceCsvFilePath = marketPriceCsvFilePath;
    }

    /**
     * 更新市场价格 CSV 文件
     */
    public void generateMarketPriceCsv() {
        List<Trade> trades = tradeRepository.getAllTrades();
        Map<String, BigDecimal> latestPrices = new HashMap<>();

        // 遍历交易记录，找到每只股票的最新价格
        for (Trade trade : trades) {
            String ticker = trade.getTicker();
            BigDecimal tradedPrice = trade.getTradedUnitPrice();
            latestPrices.put(ticker, tradedPrice); // 使用最后一次出现的交易价格
        }

        // 输出到 CSV 文件
        try (FileWriter writer = new FileWriter(marketPriceCsvFilePath)) {
            writer.write("Ticker,Market_price\n"); // 写入标题行
            for (Map.Entry<String, BigDecimal> entry : latestPrices.entrySet()) {
                writer.write(entry.getKey() + "," + entry.getValue() + "\n");
            }
            System.out.println("实时市场价格文件已更新: " + marketPriceCsvFilePath);
        } catch (IOException e) {
            System.out.println("实时市场价格文件生成失败: " + e.getMessage());
        }
    }
}
2. 修改 Main 类
在程序启动时调用 MarketPriceGenerator 生成初始价格文件。

package simplex.bn25.zhao335952.trading;

import simplex.bn25.zhao335952.trading.controller.MainController;
import simplex.bn25.zhao335952.trading.controller.StockController;
import simplex.bn25.zhao335952.trading.controller.TradeController;
import simplex.bn25.zhao335952.trading.model.repository.StockRepository;
import simplex.bn25.zhao335952.trading.model.repository.TradeRepository;
import simplex.bn25.zhao335952.trading.utils.MarketPriceGenerator;
import simplex.bn25.zhao335952.trading.view.MenuView;
import simplex.bn25.zhao335952.trading.view.PositionView;
import simplex.bn25.zhao335952.trading.view.StockView;
import simplex.bn25.zhao335952.trading.view.TradeView;

public class Main {
    public static void main(String[] args) {
        MenuView menuView = new MenuView();
        StockView stockView = new StockView();
        TradeView tradeView = new TradeView();
        PositionView positionView = new PositionView();

        String stockCsvFilePath = "C:/Users/qqq/IdeaProjects/untitled/New code/src/csvファイル/Stock.csv";
        String tradeCsvFilePath = "C:/Users/qqq/IdeaProjects/untitled/New code/src/csvファイル/Trade.csv";
        String marketPriceCsvFilePath = "C:/Users/qqq/IdeaProjects/untitled/New code/src/csvファイル/Market_price.csv";

        StockRepository stockRepository = new StockRepository(stockCsvFilePath);
        TradeRepository tradeRepository = new TradeRepository(tradeCsvFilePath);

        // 实时价格生成器
        MarketPriceGenerator marketPriceGenerator = new MarketPriceGenerator(tradeRepository, marketPriceCsvFilePath);
        marketPriceGenerator.generateMarketPriceCsv(); // 程序启动时生成实时价格文件

        StockController stockController = new StockController(stockRepository, stockView);
        TradeController tradeController = new TradeController(tradeRepository, stockRepository, tradeView, positionView);

        MainController mainController = new MainController(menuView, stockController, tradeController);
        mainController.start();
    }
}

以下是实现一个新类来生成股票的实时价格 CSV 的完整方案，并满足两个要求：

程序启动时：根据 Trade.csv 自动生成实时价格文件。
运行程序功能3后：录入新的交易后，实时更新价格文件。
实现步骤
创建 MarketPriceGenerator 类：
读取 Trade.csv：获取每只股票的最新交易价格。
生成实时价格文件：输出为 market_price.csv，包含 Ticker 和 Market_price 两列。
修改 Main 类：
在程序启动时调用 MarketPriceGenerator 生成初始价格文件。
在 TradeController 中：
在功能3（recordNewTrade）执行完成后，调用 MarketPriceGenerator 更新价格文件。
实现代码
1. MarketPriceGenerator 类

package simplex.bn25.zhao335952.trading.utils;

import simplex.bn25.zhao335952.trading.model.Trade;
import simplex.bn25.zhao335952.trading.model.repository.TradeRepository;

import java.io.FileWriter;
import java.io.IOException;
import java.math.BigDecimal;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class MarketPriceGenerator {
    private final TradeRepository tradeRepository;
    private final String marketPriceCsvFilePath;

    public MarketPriceGenerator(TradeRepository tradeRepository, String marketPriceCsvFilePath) {
        this.tradeRepository = tradeRepository;
        this.marketPriceCsvFilePath = marketPriceCsvFilePath;
    }

    /**
     * 更新市场价格 CSV 文件
     */
    public void generateMarketPriceCsv() {
        List<Trade> trades = tradeRepository.getAllTrades();
        Map<String, BigDecimal> latestPrices = new HashMap<>();

        // 遍历交易记录，找到每只股票的最新价格
        for (Trade trade : trades) {
            String ticker = trade.getTicker();
            BigDecimal tradedPrice = trade.getTradedUnitPrice();
            latestPrices.put(ticker, tradedPrice); // 使用最后一次出现的交易价格
        }

        // 输出到 CSV 文件
        try (FileWriter writer = new FileWriter(marketPriceCsvFilePath)) {
            writer.write("Ticker,Market_price\n"); // 写入标题行
            for (Map.Entry<String, BigDecimal> entry : latestPrices.entrySet()) {
                writer.write(entry.getKey() + "," + entry.getValue() + "\n");
            }
            System.out.println("实时市场价格文件已更新: " + marketPriceCsvFilePath);
        } catch (IOException e) {
            System.out.println("实时市场价格文件生成失败: " + e.getMessage());
        }
    }
}
2. 修改 Main 类
在程序启动时调用 MarketPriceGenerator 生成初始价格文件。

java
コードをコピーする
package simplex.bn25.zhao335952.trading;

import simplex.bn25.zhao335952.trading.controller.MainController;
import simplex.bn25.zhao335952.trading.controller.StockController;
import simplex.bn25.zhao335952.trading.controller.TradeController;
import simplex.bn25.zhao335952.trading.model.repository.StockRepository;
import simplex.bn25.zhao335952.trading.model.repository.TradeRepository;
import simplex.bn25.zhao335952.trading.utils.MarketPriceGenerator;
import simplex.bn25.zhao335952.trading.view.MenuView;
import simplex.bn25.zhao335952.trading.view.PositionView;
import simplex.bn25.zhao335952.trading.view.StockView;
import simplex.bn25.zhao335952.trading.view.TradeView;

public class Main {
    public static void main(String[] args) {
        MenuView menuView = new MenuView();
        StockView stockView = new StockView();
        TradeView tradeView = new TradeView();
        PositionView positionView = new PositionView();

        String stockCsvFilePath = "C:/Users/qqq/IdeaProjects/untitled/New code/src/csvファイル/Stock.csv";
        String tradeCsvFilePath = "C:/Users/qqq/IdeaProjects/untitled/New code/src/csvファイル/Trade.csv";
        String marketPriceCsvFilePath = "C:/Users/qqq/IdeaProjects/untitled/New code/src/csvファイル/Market_price.csv";

        StockRepository stockRepository = new StockRepository(stockCsvFilePath);
        TradeRepository tradeRepository = new TradeRepository(tradeCsvFilePath);

        // 实时价格生成器
        MarketPriceGenerator marketPriceGenerator = new MarketPriceGenerator(tradeRepository, marketPriceCsvFilePath);
        marketPriceGenerator.generateMarketPriceCsv(); // 程序启动时生成实时价格文件

        StockController stockController = new StockController(stockRepository, stockView);
        TradeController tradeController = new TradeController(tradeRepository, stockRepository, tradeView, positionView);

        MainController mainController = new MainController(menuView, stockController, tradeController);
        mainController.start();
    }
}
3. 修改 TradeController
在 recordNewTrade 方法中录入新交易后调用 MarketPriceGenerator 更新实时价格文件。
public void recordNewTrade() {
    // 原有交易录入逻辑...
    Trade trade = new Trade(tradedDatetime, ticker, null, side, quantity, tradedUnitPrice, inputDatetime);

    if (tradeRepository.saveTrade(trade)) {
        tradeView.showTradeAddedMessage(trade);

        // 更新实时价格文件
        MarketPriceGenerator marketPriceGenerator = new MarketPriceGenerator(tradeRepository, "C:/Users/qqq/IdeaProjects/untitled/New code/src/csvファイル/Market_price.csv");
        marketPriceGenerator.generateMarketPriceCsv();
    } else {
        System.out.print("データの書き込みにエラーが発生しました。");
    }
}




