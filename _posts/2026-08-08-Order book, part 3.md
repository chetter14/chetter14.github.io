---
layout: post
title: Order book. Part 3.
---

So, I began implementing two other modules for my program: **order generator** and **logger**. I'm going to start with the first one.

In order generator, I think the most important part to know is how **price** values (of incoming orders) are generated. I made it in a form of **normal distribution (a Gaussian bell curve)**. This way buy/sell orders are executed mostly in one place. In a case like that, just a few orders will sit in order book and wait for the suitable bid/ask. Most of the time bids/asks are satisfied and order book is under highload.

Other values that have to be generated for **ob::InputOrder** structure are:
```
struct InputOrder {
  UserId userId;
  Price price;
  Amount amount;
  OrderType type;
};
```

I made a class **OrderGenerator** with a constructor that expects a seed. The constructor initializes private fields of class. These fields are a **generator engine** used and distributions for all the values - **userId, price, amount, type**. 
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

There is one vital function in this class to generate a random input order object - `generateOrder()`. I guess, the description of function is self-contained.
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

For generating random values, I defined helper functions `nextUserId(), nextPrice(), nextAmount(), nextOrderType()`, which are called by `generateOrder()`:
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

The list of tests I added for the **order generator** module (think, I won't provide source code for it here):
- The values of *userId, price, amount, and orderType* are always in range.
- The same seed produces the same sequence.
- After a large number of iterations, an average generated price value is going to be close to the one specified in normal distribution.
- After a large number of iterations, both order types (*buy* and *sell*) are generated almost equal number of times.

Getting to the **logger**. The first thing I did here is added an abstract class `ExecutionSink`. It acts like a destination/logger/sink that is going to do something on execution of orders:
```
namespace ob {
class ExecutionSink {
 public:
  virtual ~ExecutionSink() = default;
  virtual void onExecuted(const ExecutedOrder&) = 0;
};
}  // namespace ob
```

I modified an `OrderBook` class a bit to set an execution sink and write to it at the execution of orders:
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

This way I can set a whatever sink that implements `onExecuted(const ExecutedOrder&)`. It will help a lot at integration tests.

Now to `Logger` class. It inherits from `ob::ExecutionSink` and overrides `onExecuted()` by just redirecting to its internal function for writing a log. 
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

As for creation of this logger, this `Logger` class defines a static method that creates an instance of the logger. If it fails (at opening the log file), an error is returned:
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

Also, all the operations of copy and move are restricted:
```
  Logger(const Logger&) = delete;
  Logger& operator=(const Logger&) = delete;

  Logger(Logger&&) = delete;
  Logger& operator=(Logger&&) = delete;
```

So, what `recordExecutedOrder()` does is take a mutex, call a helper function for formatting a `const ob::ExecutedOrder& order` into a string, and print this string out to the log file. A helper function is `formatExecutedOrder()`. The format it uses is specified below in body of the function:
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

For logger I wrote these unit-tests:
- `formatExecutedOrder()` produces an expected line.
- Behavior of `ob_logger::Logger::create()` with correct and wrong path.
- Record a number of logs to file, then read the file string by string, check whether strings match the *regex*, and total number of strings is equal to the number of strings written.
