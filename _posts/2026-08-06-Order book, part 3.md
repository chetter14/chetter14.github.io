---
layout: post
title: Order book. Part 3.
---

So, I began implementing two other modules for my program: the **order generator** and the **logger**. I'm going to start with the first one.

In the order generator, I think the most important part to understand is how the **price** values (of incoming orders) are generated. I made it in the form of a **normal distribution (a Gaussian bell curve)**. This way, buy/sell orders are mostly executed around one price point. In a case like that, only a few orders will sit in the order book waiting for a suitable bid/ask. Most of the time bids/asks are matched, and the order book is under high load.

The other values that have to be generated for the **ob::InputOrder** structure are:
```
struct InputOrder {
  UserId userId;
  Price price;
  Amount amount;
  OrderType type;
};
```

I made a class, **OrderGenerator**, with a constructor that expects a seed. The constructor initializes the private fields of the class. These fields are the **generator engine** and the distributions for all the values: **userId, price, amount, type**.
```
class OrderGenerator {
 public:
  explicit OrderGenerator(unsigned int seed)
      : generatorEngine_(seed),
        userIdGen_(MIN_USER_ID, MAX_USER_ID),
        priceGen_(ob::MIN_PRICE_VALUE +
                     (ob::MAX_PRICE_VALUE - ob::MIN_PRICE_VALUE) / 2,
                 PRICE_STDDEV),
        amountGen_(MIN_AMOUNT_VALUE, MAX_AMOUNT_VALUE),
        orderTypeGen_(0.5) {}

...

  /* Variables that are used for generating random values */
 private:
  std::mt19937 generatorEngine_;

  std::uniform_int_distribution<ob::UserId> userIdGen_;
  std::normal_distribution<double> priceGen_;
  std::uniform_int_distribution<ob::Amount> amountGen_;
  std::bernoulli_distribution orderTypeGen_;

};
```

There is one key function in this class for generating a random input order object: `generateOrder()`. I think the description of the function speaks for itself.
```
  /**
  * @brief Generates an input order with random values:
  * 
  * userId - uniform distribution in [MIN_USER_ID, MAX_USER_ID].
  * 
  * price - normal distribution in [MIN_PRICE_VALUE, MAX_PRICE_VALUE].
  * The range is fixed, values outside of it are truncated to MIN/MAX.
  * 
  * amount - uniform distribution in [MIN_AMOUNT_VALUE, MAX_AMOUNT_VALUE].
  * 
  * type - buy/sell is 50/50.
  * 
  * @return ob::InputOrder 
  */
  ob::InputOrder generateOrder();
```

To generate the random values, I defined the helper functions `nextUserId()`, `nextPrice()`, `nextAmount()`, and `nextOrderType()`, which are called by `generateOrder()`:
```
ob::InputOrder og::OrderGenerator::generateOrder() {
  return ob::InputOrder{.userId = nextUserId(),
                        .price = nextPrice(),
                        .amount = nextAmount(),
                        .type = nextOrderType()};
}

ob::UserId og::OrderGenerator::nextUserId() {
  return userIdGen_(generatorEngine_);
}

ob::Price og::OrderGenerator::nextPrice() {
  for (;;) {
    const auto v = std::lround(priceGen_(generatorEngine_));
    if (v >= static_cast<long>(ob::MIN_PRICE_VALUE) &&
        v <= static_cast<long>(ob::MAX_PRICE_VALUE)) {
      return static_cast<ob::Price>(v);
    }
  }
}

ob::Amount og::OrderGenerator::nextAmount() {
  return amountGen_(generatorEngine_);
}

ob::OrderType og::OrderGenerator::nextOrderType() {
  return orderTypeGen_(generatorEngine_) ? ob::OrderType::BUY
                                       : ob::OrderType::SELL;
}
```

The list of tests I added for the **order generator** module (I don't think I'll provide the source code for them here):
- The values of *userId*, *price*, *amount*, and *orderType* are always in range.
- The same seed produces the same sequence.
- After a large number of iterations, the average generated price is close to the one specified in the normal distribution.
- After a large number of iterations, both order types (*buy* and *sell*) are generated an almost equal number of times.

Now on to the **logger**. The first thing I did here was add an abstract class, `ExecutionSink`. It acts as a destination/logger/sink that does something when orders are executed:
```
namespace ob {
class ExecutionSink {
 public:
  virtual ~ExecutionSink() = default;
  virtual void onExecuted(const ExecutedOrder&) = 0;
};
}  // namespace ob
```

I modified the `OrderBook` class a bit so that it can take an execution sink and write to it when orders are executed:
```
class OrderBook {
 public:
  void setSink(ExecutionSink *sink) {
    m_sink = sink;
  }
};

void ob::OrderBook::executeOrdersAtPrice(...) {
  while (...) {
    ...
    if (m_sink) {
      if (logInfo.incomingOrderType == ob::OrderType::BUY) {
        m_sink->onExecuted({.buyer = logInfo.incomingOrderUserId,
                            .seller = resting.userId,
                            .type = ob::OrderType::BUY,
                            .price = logInfo.atPrice,
                            .amount = executed});
      } else {
        m_sink->onExecuted({.buyer = resting.userId,
                            .seller = logInfo.incomingOrderUserId,
                            .type = ob::OrderType::SELL,
                            .price = logInfo.atPrice,
                            .amount = executed});
      }
    }
  }
}
```

This way I can set any sink that implements `onExecuted(const ExecutedOrder&)`. It will help a lot with integration tests.

Now to the `Logger` class. It inherits from `ob::ExecutionSink` and overrides `onExecuted()` by simply redirecting to its internal function for writing a log entry.
```
class Logger : public ob::ExecutionSink {
  ...
 public:
  void onExecuted(const ob::ExecutedOrder& order) override {
    recordExecutedOrder(order);
  }

  void recordExecutedOrder(const ob::ExecutedOrder& order);
  ...
};
```

As for the creation of this logger, the `Logger` class defines a *static* method that creates an instance of the logger. If it fails (when opening the log file), an error is returned:
```
class Logger : public ob::ExecutionSink {
 private:
  Logger() = default;
 public:
  static std::expected<std::unique_ptr<Logger>, LoggerError> create(
      const std::filesystem::path& path);
  ...
};

std::expected<std::unique_ptr<ob_logger::Logger>, ob_logger::LoggerError>
ob_logger::Logger::create(const std::filesystem::path& path) {

  std::unique_ptr<ob_logger::Logger> logger{new Logger()};

  logger->m_out = std::ofstream{path, std::ios::app};
  if (!logger->m_out) {
    return std::unexpected(ob_logger::LoggerError::FAILED_TO_OPEN_FILE);
  }

  return logger;
};
```

All copy and move operations are *deleted* as well:
```
  Logger(const Logger&) = delete;
  Logger& operator=(const Logger&) = delete;

  Logger(Logger&&) = delete;
  Logger& operator=(Logger&&) = delete;
```

So, what `recordExecutedOrder()` does is lock a mutex, call a helper function to format the `const ob::ExecutedOrder& order` into a string, and write this string to the log file. The helper function is `formatExecutedOrder()`. The format it uses is specified in the body of the function below:
```
std::string ob_logger::formatExecutedOrder(
    const ob::ExecutedOrder& order,
    std::chrono::system_clock::time_point when) {
  std::ostringstream os;

  /* "[{} UTC] OPERATION={}, BUYER={}, SELLER={}, PRICE={}, AMOUNT={}" */
  os << "[" << std::chrono::floor<std::chrono::milliseconds>(when)
     << " UTC] OPERATION=" << ob::to_string(order.type)
     << ", BUYER=" << order.buyer << ", SELLER=" << order.seller
     << ", PRICE=" << order.price << ", AMOUNT=" << order.amount;

  return os.str();
}

void ob_logger::Logger::recordExecutedOrder(const ob::ExecutedOrder& order) {

  std::lock_guard lock{m_mtx};

  const auto now = std::chrono::system_clock::now();

  m_out << formatExecutedOrder(order, now) << "\n";
  m_out.flush();
}
```

For the logger I wrote these unit tests:
- `formatExecutedOrder()` produces the expected line.
- The behavior of `ob_logger::Logger::create()` with a valid and an invalid path.
- Write a number of log entries to a file, then read the file line by line, check that the lines match the regex, and that the total number of lines equals the number written.

That's it for the **order generator** and **logger** modules.