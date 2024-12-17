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




