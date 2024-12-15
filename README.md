以下是功能5（保有ポジション表示）相关的所有必须代码的整合，并附上每个部分的说明。
1. TradeController 中的 displayHoldings 方法
说明：
读取 Trade.csv 获取交易记录，计算每只股票的持仓量。
使用 StockRepository 从 Stock.csv 中检索股票名称。
调用 PositionView 的方法显示最终持仓数据。

package simplex.bn25.zhao335952.trading.controller;

import simplex.bn25.zhao335952.trading.model.Trade;
import simplex.bn25.zhao335952.trading.model.repository.StockRepository;
import simplex.bn25.zhao335952.trading.model.repository.TradeRepository;
import simplex.bn25.zhao335952.trading.view.PositionView;

import java.util.List;
import java.util.Map;
import java.util.TreeMap;

public class TradeController {
    private final TradeRepository tradeRepository;
    private final StockRepository stockRepository; // 用于检索股票名称
    private final PositionView positionView;

    public TradeController(TradeRepository tradeRepository, StockRepository stockRepository, PositionView positionView) {
        this.tradeRepository = tradeRepository;
        this.stockRepository = stockRepository; // 初始化
        this.positionView = positionView;
    }

    public void displayHoldings() {
        List<Trade> trades = tradeRepository.getAllTrades();
        Map<String, Integer> holdings = new TreeMap<>(); // 使用 TreeMap 保持按键排序

        // 计算每只股票的持仓量
        for (Trade trade : trades) {
            int quantity = trade.getQuantity();
            if ("Sell".equalsIgnoreCase(trade.getSide())) {
                quantity = -quantity; // 卖出为负数
            }
            holdings.merge(trade.getTicker(), quantity, Integer::sum); // 合并持仓数据
        }

        // 构建带股票名称的持仓数据
        Map<String, String> tickerToNameMap = new TreeMap<>();
        for (String ticker : holdings.keySet()) {
            String productName = stockRepository.getTickerNameByTicker(ticker); // 检索股票名称
            tickerToNameMap.put(ticker, productName);
        }

        // 调用视图层输出持仓数据
        positionView.displayHoldings(holdings, tickerToNameMap);
    }
}

2. PositionView 中的 displayHoldings 方法
说明：
接收股票代码到持仓数量的映射表 (holdings) 和股票代码到名称的映射表 (tickerToNameMap)。
格式化并显示持仓数据，股票名称会被截断以适应显示。

package simplex.bn25.zhao335952.trading.view;

import java.util.Map;

public class PositionView {

    public void displayHoldings(Map<String, Integer> holdings, Map<String, String> tickerToNameMap) {
        if (holdings.isEmpty()) {
            System.out.println("保有データが見つかりません。");
            return;
        }

        System.out.println("-------------------------------------------------------------------------------");
        System.out.printf("| %-6s | %-30s | %10s |%n", "Ticker", "Product Name", "Quantity");
        System.out.println("-------------------------------------------------------------------------------");

        for (Map.Entry<String, Integer> entry : holdings.entrySet()) {
            String ticker = entry.getKey();
            int quantity = entry.getValue();
            String productName = tickerToNameMap.getOrDefault(ticker, "Unknown");

            System.out.printf("| %-6s | %-30s | %15s |%n",
                    ticker.toUpperCase(),
                    truncateProductName(productName),
                    formatQuantity(quantity));
        }

        System.out.println("-------------------------------------------------------------------------------");
    }

    private String truncateProductName(String productName) {
        // 截断长于30个字符的股票名称
        if (productName.length() > 30) {
            return productName.substring(0, 27) + "...";
        }
        return productName;
    }

    private String formatQuantity(int quantity) {
        // 格式化数量为带逗号的形式
        return String.format("%,d", quantity);
    }
}


以下是功能5（保有ポジション表示）相关的所有必须代码的整合，并附上每个部分的说明。

1. TradeController 中的 displayHoldings 方法
说明：
读取 Trade.csv 获取交易记录，计算每只股票的持仓量。
使用 StockRepository 从 Stock.csv 中检索股票名称。
调用 PositionView 的方法显示最终持仓数据。
java
コードをコピーする
package simplex.bn25.zhao335952.trading.controller;

import simplex.bn25.zhao335952.trading.model.Trade;
import simplex.bn25.zhao335952.trading.model.repository.StockRepository;
import simplex.bn25.zhao335952.trading.model.repository.TradeRepository;
import simplex.bn25.zhao335952.trading.view.PositionView;

import java.util.List;
import java.util.Map;
import java.util.TreeMap;

public class TradeController {
    private final TradeRepository tradeRepository;
    private final StockRepository stockRepository; // 用于检索股票名称
    private final PositionView positionView;

    public TradeController(TradeRepository tradeRepository, StockRepository stockRepository, PositionView positionView) {
        this.tradeRepository = tradeRepository;
        this.stockRepository = stockRepository; // 初始化
        this.positionView = positionView;
    }

    public void displayHoldings() {
        List<Trade> trades = tradeRepository.getAllTrades();
        Map<String, Integer> holdings = new TreeMap<>(); // 使用 TreeMap 保持按键排序

        // 计算每只股票的持仓量
        for (Trade trade : trades) {
            int quantity = trade.getQuantity();
            if ("Sell".equalsIgnoreCase(trade.getSide())) {
                quantity = -quantity; // 卖出为负数
            }
            holdings.merge(trade.getTicker(), quantity, Integer::sum); // 合并持仓数据
        }

        // 构建带股票名称的持仓数据
        Map<String, String> tickerToNameMap = new TreeMap<>();
        for (String ticker : holdings.keySet()) {
            String productName = stockRepository.getTickerNameByTicker(ticker); // 检索股票名称
            tickerToNameMap.put(ticker, productName);
        }

        // 调用视图层输出持仓数据
        positionView.displayHoldings(holdings, tickerToNameMap);
    }
}
2. PositionView 中的 displayHoldings 方法
说明：
接收股票代码到持仓数量的映射表 (holdings) 和股票代码到名称的映射表 (tickerToNameMap)。
格式化并显示持仓数据，股票名称会被截断以适应显示。
java
コードをコピーする
package simplex.bn25.zhao335952.trading.view;

import java.util.Map;

public class PositionView {

    public void displayHoldings(Map<String, Integer> holdings, Map<String, String> tickerToNameMap) {
        if (holdings.isEmpty()) {
            System.out.println("保有データが見つかりません。");
            return;
        }

        System.out.println("-------------------------------------------------------------------------------");
        System.out.printf("| %-6s | %-30s | %10s |%n", "Ticker", "Product Name", "Quantity");
        System.out.println("-------------------------------------------------------------------------------");

        for (Map.Entry<String, Integer> entry : holdings.entrySet()) {
            String ticker = entry.getKey();
            int quantity = entry.getValue();
            String productName = tickerToNameMap.getOrDefault(ticker, "Unknown");

            System.out.printf("| %-6s | %-30s | %15s |%n",
                    ticker.toUpperCase(),
                    truncateProductName(productName),
                    formatQuantity(quantity));
        }

        System.out.println("-------------------------------------------------------------------------------");
    }

    private String truncateProductName(String productName) {
        // 截断长于30个字符的股票名称
        if (productName.length() > 30) {
            return productName.substring(0, 27) + "...";
        }
        return productName;
    }

    private String formatQuantity(int quantity) {
        // 格式化数量为带逗号的形式
        return String.format("%,d", quantity);
    }
}

3. StockRepository 中的 getTickerNameByTicker 方法
说明：
在 Stock.csv 中通过股票代码查找对应的股票名称。
如果未找到股票代码，返回 "Unknown"。

package simplex.bn25.zhao335952.trading.model.repository;

import java.io.BufferedReader;
import java.io.FileReader;
import java.io.IOException;

public class StockRepository {
    private final String csvFilePath;

    public StockRepository(String csvFilePath) {
        this.csvFilePath = csvFilePath;
    }

    public String getTickerNameByTicker(String ticker) {
        try (BufferedReader br = new BufferedReader(new FileReader(csvFilePath))) {
            String line;
            br.readLine(); // Skip header
            while ((line = br.readLine()) != null) {
                String[] fields = line.split(",");
                if (fields.length > 1 && fields[0].trim().equalsIgnoreCase(ticker)) {
                    return fields[1].trim(); // 返回股票名称
                }
            }
        } catch (IOException e) {
            System.out.println("データの読み込み中にエラーが発生しました。");
        }
        return "Unknown";
    }
}

4. TradeRepository 中的 getAllTrades 方法
说明：
从 Trade.csv 读取交易记录，解析新的数据格式。
不再读取股票名称，因为它会从 StockRepository 动态检索。
package simplex.bn25.zhao335952.trading.model.repository;

import simplex.bn25.zhao335952.trading.model.Trade;

import java.io.*;
import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.ArrayList;
import java.util.List;

public class TradeRepository {
    private final String csvFilePath;
    private static final DateTimeFormatter DATETIME_FORMATTER = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm");

    public TradeRepository(String csvFilePath) {
        this.csvFilePath = csvFilePath;
    }

    public List<Trade> getAllTrades() {
        List<Trade> trades = new ArrayList<>();

        try (BufferedReader br = new BufferedReader(new FileReader(csvFilePath))) {
            String line;
            br.readLine(); // Skip header
            while ((line = br.readLine()) != null) {
                String[] fields = line.split(",");
                if (fields.length == 6) { // 根据新格式调整解析逻辑
                    LocalDateTime tradedDatetime = LocalDateTime.parse(fields[0].trim(), DATETIME_FORMATTER);
                    String ticker = fields[1].trim();
                    String side = fields[2].trim();
                    int quantity = Integer.parseInt(fields[3].trim());
                    BigDecimal tradedUnitPrice = new BigDecimal(fields[4].trim());
                    LocalDateTime inputDatetime = LocalDateTime.parse(fields[5].trim(), DATETIME_FORMATTER);

                    // 创建交易对象，股票名称将在 TradeController 中补全
                    Trade trade = new Trade(tradedDatetime, ticker, null, side, quantity, tradedUnitPrice, inputDatetime);
                    trades.add(trade);
                }
            }
        } catch (IOException e) {
            System.out.println("データの読み込み中にエラーが発生しました。");
        }

        return trades;
    }
}

MainController 中的逻辑
说明：
负责调用 TradeController 的 displayHoldings 方法来执行功能5。
新增功能时通过菜单选项触发调用。
package simplex.bn25.zhao335952.trading.controller;

import simplex.bn25.zhao335952.trading.view.MenuView;

public class MainController {
    private final MenuView menuView;
    private final StockController stockController;
    private final TradeController tradeController;

    public MainController(MenuView menuView, StockController stockController, TradeController tradeController) {
        this.menuView = menuView;
        this.stockController = stockController;
        this.tradeController = tradeController;
    }

    public void start() {
        boolean running = true;
        while (running) {
            String choice = menuView.displayMainMenu();
            switch (choice) {
                case "1" -> {
                    System.out.println("「銘柄マスタ一覧表示」が選択されました。");
                    stockController.displayAllStocks();
                }
                case "2" -> {
                    System.out.println("「銘柄マスタ新規登録」が選択されました。");
                    stockController.addNewStock();
                }
                case "3" -> {
                    System.out.println("「取引新規登録」が選択されました。");
                    tradeController.recordNewTrade();
                }
                case "4" -> {
                    System.out.println("「取引一覧表示」が選択されました。");
                    tradeController.displayAllTrades();
                }
                case "5" -> {
                    System.out.println("「保有ポジション表示」が選択されました。");
                    tradeController.displayHoldings(); // 调用功能5
                }
                case "9" -> {
                    System.out.println("アプリケーションを終了します。");
                    running = false;
                }
                default -> System.out.println("対応するメニューは存在しません。");
            }
            System.out.println("---");
        }
        menuView.close();
    }
}

Main 类入口
说明：
创建 StockRepository、TradeRepository、StockController、TradeController、PositionView 等实例。
将所有组件连接到 MainController。

package simplex.bn25.zhao335952.trading;

import simplex.bn25.zhao335952.trading.controller.MainController;
import simplex.bn25.zhao335952.trading.controller.StockController;
import simplex.bn25.zhao335952.trading.controller.TradeController;
import simplex.bn25.zhao335952.trading.model.repository.StockRepository;
import simplex.bn25.zhao335952.trading.model.repository.TradeRepository;
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

        StockRepository stockRepository = new StockRepository(stockCsvFilePath);
        TradeRepository tradeRepository = new TradeRepository(tradeCsvFilePath);

        StockController stockController = new StockController(stockRepository, stockView);
        TradeController tradeController = new TradeController(tradeRepository, stockRepository, tradeView, positionView);

        MainController mainController = new MainController(menuView, stockController, tradeController);
        mainController.start();
    }
}

TradeController 的功能5逻辑
说明：
负责读取交易记录并计算持仓。
调用 StockRepository 的 getTickerNameByTicker 方法检索股票名称。
将持仓数据和名称传递给 PositionView。

package simplex.bn25.zhao335952.trading.controller;

import simplex.bn25.zhao335952.trading.model.Trade;
import simplex.bn25.zhao335952.trading.model.repository.StockRepository;
import simplex.bn25.zhao335952.trading.model.repository.TradeRepository;
import simplex.bn25.zhao335952.trading.view.PositionView;

import java.util.List;
import java.util.Map;
import java.util.TreeMap;

public class TradeController {
    private final TradeRepository tradeRepository;
    private final StockRepository stockRepository;
    private final PositionView positionView;

    public TradeController(TradeRepository tradeRepository, StockRepository stockRepository, PositionView positionView) {
        this.tradeRepository = tradeRepository;
        this.stockRepository = stockRepository;
        this.positionView = positionView;
    }

    public void displayHoldings() {
        List<Trade> trades = tradeRepository.getAllTrades();
        Map<String, Integer> holdings = new TreeMap<>();

        for (Trade trade : trades) {
            int quantity = trade.getQuantity();
            if ("Sell".equalsIgnoreCase(trade.getSide())) {
                quantity = -quantity; // 卖出为负数
            }
            holdings.merge(trade.getTicker(), quantity, Integer::sum);
        }

        Map<String, String> tickerToNameMap = new TreeMap<>();
        for (String ticker : holdings.keySet()) {
            String productName = stockRepository.getTickerNameByTicker(ticker);
            tickerToNameMap.put(ticker, productName);
        }

        positionView.displayHoldings(holdings, tickerToNameMap);
    }
}

PositionView 的显示逻辑
说明：
显示持仓数据，包含股票代码、名称和数量。
格式化名称和数量以便于阅读。
package simplex.bn25.zhao335952.trading.view;

import java.util.Map;

public class PositionView {

    public void displayHoldings(Map<String, Integer> holdings, Map<String, String> tickerToNameMap) {
        if (holdings.isEmpty()) {
            System.out.println("保有データが見つかりません。");
            return;
        }

        System.out.println("-------------------------------------------------------------------------------");
        System.out.printf("| %-6s | %-30s | %10s |%n", "Ticker", "Product Name", "Quantity");
        System.out.println("-------------------------------------------------------------------------------");

        for (Map.Entry<String, Integer> entry : holdings.entrySet()) {
            String ticker = entry.getKey();
            int quantity = entry.getValue();
            String productName = tickerToNameMap.getOrDefault(ticker, "Unknown");

            System.out.printf("| %-6s | %-30s | %15s |%n",
                    ticker.toUpperCase(),
                    truncateProductName(productName),
                    formatQuantity(quantity));
        }

        System.out.println("-------------------------------------------------------------------------------");
    }

    private String truncateProductName(String productName) {
        if (productName.length() > 30) {
            return productName.substring(0, 27) + "...";
        }
        return productName;
    }

    private String formatQuantity(int quantity) {
        return String.format("%,d", quantity);
    }
}

StockRepository 中的检索逻辑
说明：
从 Stock.csv 中检索股票名称，若找不到返回 "Unknown"。

package simplex.bn25.zhao335952.trading.model.repository;

import java.io.BufferedReader;
import java.io.FileReader;
import java.io.IOException;

public class StockRepository {
    private final String csvFilePath;

    public StockRepository(String csvFilePath) {
        this.csvFilePath = csvFilePath;
    }

    public String getTickerNameByTicker(String ticker) {
        try (BufferedReader br = new BufferedReader(new FileReader(csvFilePath))) {
            String line;
            br.readLine();
            while ((line = br.readLine()) != null) {
                String[] fields = line.split(",");
                if (fields.length > 1 && fields[0].trim().equalsIgnoreCase(ticker)) {
                    return fields[1].trim();
                }
            }
        } catch (IOException e) {
            System.out.println("データの読み込み中にエラーが発生しました。");
        }
        return "Unknown";
    }
}

TradeRepository 中的交易读取逻辑
说明：
从 Trade.csv 中读取交易记录，返回交易列表。
package simplex.bn25.zhao335952.trading.model.repository;

import simplex.bn25.zhao335952.trading.model.Trade;

import java.io.*;
import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.ArrayList;
import java.util.List;

public class TradeRepository {
    private final String csvFilePath;
    private static final DateTimeFormatter DATETIME_FORMATTER = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm");

    public TradeRepository(String csvFilePath) {
        this.csvFilePath = csvFilePath;
    }

    public List<Trade> getAllTrades() {
        List<Trade> trades = new ArrayList<>();

        try (BufferedReader br = new BufferedReader(new FileReader(csvFilePath))) {
            String line;
            br.readLine();
            while ((line = br.readLine()) != null) {
                String[] fields = line.split(",");
                if (fields.length == 6) {
                    LocalDateTime tradedDatetime = LocalDateTime.parse(fields[0].trim(), DATETIME_FORMATTER);
                    String ticker = fields[1].trim();
                    String side = fields[2].trim();
                    int quantity = Integer.parseInt(fields[3].trim());
                    BigDecimal tradedUnitPrice = new BigDecimal(fields[4].trim());
                    LocalDateTime inputDatetime = LocalDateTime.parse(fields[5].trim(), DATETIME_FORMATTER);

                    Trade trade = new Trade(tradedDatetime, ticker, null, side, quantity, tradedUnitPrice, inputDatetime);
                    trades.add(trade);
                }
            }
        } catch (IOException e) {
            System.out.println("データの読み込み中にエラーが発生しました。");
        }

        return trades;
    }
}


以下是对功能3（recordNewTrade）的要求进行实现的修改，包含两个追加条件的独立方法，并在 recordNewTrade 中引用它们。

修改方案
1. 新增方法：
isValidQuantityAfterTrade：检查交易后的持仓数量是否会变为负数。
isValidTradeDatetime：检查交易时间是否早于或等于该股票在 Trade.csv 中的最晚交易时间。
2. 在 recordNewTrade 中引用：
在用户输入数量后调用 isValidQuantityAfterTrade。
在用户输入交易时间后调用 isValidTradeDatetime。

修改 TradeController
package simplex.bn25.zhao335952.trading.controller;

import simplex.bn25.zhao335952.trading.model.Trade;
import simplex.bn25.zhao335952.trading.model.repository.StockRepository;
import simplex.bn25.zhao335952.trading.model.repository.TradeRepository;
import simplex.bn25.zhao335952.trading.view.TradeView;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.List;
import java.util.Map;
import java.util.TreeMap;

public class TradeController {
    private final TradeRepository tradeRepository;
    private final StockRepository stockRepository;
    private final TradeView tradeView;

    public TradeController(TradeRepository tradeRepository, StockRepository stockRepository, TradeView tradeView) {
        this.tradeRepository = tradeRepository;
        this.stockRepository = stockRepository;
        this.tradeView = tradeView;
    }

    public void recordNewTrade() {
        // 获取用户输入（简化为示例代码逻辑）
        System.out.print("取引日時を入力してください (yyyy-MM-dd HH:mm):");
        LocalDateTime tradedDatetime = LocalDateTime.parse(scanner.nextLine().trim(), DATETIME_FORMATTER);

        System.out.print("銘柄コードを入力してください (4桁の半角英数字): ");
        String ticker = scanner.nextLine().trim();

        if (!tradeRepository.isTickerRegistered(ticker)) {
            System.out.println("登録されているいない銘柄コードです。再度入力してください。");
            return;
        }

        if (!isValidTradeDatetime(ticker, tradedDatetime)) {
            System.out.println("取引日時は既存の最遅取引日時より後である必要があります。");
            return;
        }

        System.out.print("売買区分を入力してください (Buy/Sell): ");
        String side = scanner.nextLine().trim();

        System.out.print("数量を入力してください (100株単位):");
        int quantity = Integer.parseInt(scanner.nextLine().trim());

        if (!isValidQuantityAfterTrade(ticker, side, quantity)) {
            System.out.println("保有数量がマイナスになるため、この取引は無効です。");
            return;
        }

        System.out.print("取引単価を入力してください (小数点以下2桁まで):");
        BigDecimal tradedUnitPrice = new BigDecimal(scanner.nextLine().trim());

        LocalDateTime inputDatetime = LocalDateTime.now();

        Trade trade = new Trade(tradedDatetime, ticker, null, side, quantity, tradedUnitPrice, inputDatetime);

        if (tradeRepository.saveTrade(trade)) {
            tradeView.showTradeAddedMessage(trade);
        } else {
            System.out.print("データの書き込みにエラーが発生しました。");
        }
    }

    /**
     * 检查交易后的持仓数量是否会变为负数
     */
    private boolean isValidQuantityAfterTrade(String ticker, String side, int quantity) {
        List<Trade> trades = tradeRepository.getAllTrades();
        Map<String, Integer> holdings = new TreeMap<>();

        // 计算当前持仓
        for (Trade trade : trades) {
            int tradeQuantity = trade.getQuantity();
            if ("Sell".equalsIgnoreCase(trade.getSide())) {
                tradeQuantity = -tradeQuantity;
            }
            holdings.merge(trade.getTicker(), tradeQuantity, Integer::sum);
        }

        int currentQuantity = holdings.getOrDefault(ticker, 0);
        int newQuantity = currentQuantity + ("Sell".equalsIgnoreCase(side) ? -quantity : quantity);

        return newQuantity >= 0;
    }

    /**
     * 检查交易时间是否早于或等于该股票的最晚交易时间
     */
    private boolean isValidTradeDatetime(String ticker, LocalDateTime tradedDatetime) {
        List<Trade> trades = tradeRepository.getAllTrades();
        LocalDateTime latestDatetime = trades.stream()
                .filter(trade -> trade.getTicker().equalsIgnoreCase(ticker))
                .map(Trade::getTradedDatetime)
                .max(LocalDateTime::compareTo)
                .orElse(LocalDateTime.MIN);

        return tradedDatetime.isAfter(latestDatetime);
    }
}

以下是实现一个新类来生成股票的实时价格 CSV 的完整方案，并满足两个要求：

程序启动时：根据 Trade.csv 自动生成实时价格文件。
运行程序功能3后：录入新的交易后，实时更新价格文件。

创建 MarketPriceGenerator 类：
读取 Trade.csv：获取每只股票的最新交易价格。
生成实时价格文件：输出为 market_price.csv，包含 Ticker 和 Market_price 两列。
修改 Main 类：
在程序启动时调用 MarketPriceGenerator 生成初始价格文件。
在 TradeController 中：
在功能3（recordNewTrade）执行完成后，调用 MarketPriceGenerator 更新价格文件。

MarketPriceGenerator 类
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

修改 Main 类
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

修改 TradeController
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

