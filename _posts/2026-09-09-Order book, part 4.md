---
layout: post
title: Order book. Part 4.
---

Let's *connect all the modules together* in a **sample** program. The sample performs the following straightforward sequence of steps:

1) It initializes a logger, an order book, and an order generator.
2) In an infinite loop, it generates an order, applies it, and sleeps for 5 seconds. The sleep is there for presentation purposes only and should be removed in a real program.

The implementation is small enough that I can put the whole `main()` here:
```
int main() {
  std::cout << "Launched an order book app" << std::endl;

  auto logCreateResult = ob_logger::Logger::create("logs.txt");
  if (!logCreateResult.has_value()) {
    std::cout << ob_logger::to_string(logCreateResult.error()) << "\n";
    exit(1);
  }

  auto logger = std::move(logCreateResult.value());

  auto orderBook = std::make_unique<ob::OrderBook>();
  orderBook->setSink(logger.get());

  og::OrderGenerator orderGen{1};

  while (true) {
    auto inputOrder = orderGen.generateOrder();
    auto applyResult = orderBook->applyOrder(inputOrder);
    if (applyResult.has_value()) {
      std::cout << inputOrder << " was applied\n";
    }

    using namespace std::chrono_literals;
    std::this_thread::sleep_for(5s);
  }

  return 0;
}
```

Now, on to **integration tests**. Here I want to check that if I feed X into my program, I get the expected output Y. This is easy to achieve with the `ob::ExecutionSink` interface.

For further tests I wrote an implementation of `ob::ExecutionSink` - a `RecordingSink` that simply stores all executed orders:
```
class RecordingSink : public ob::ExecutionSink {
 public:
  void onExecuted(const ob::ExecutedOrder& eo) override {
    executions.push_back(eo);
  }
  std::vector<ob::ExecutedOrder> executions;
};
```

First, I want to ensure that the number of shares submitted to the order book equals the number of executed and resting shares - that is: `APPLIED_SHARES = EXECUTED_SHARES * 2 + RESTING_SHARES`. Each execution consumes shares from two orders, hence the factor of two.

I set up a `RecordingSink` and count the total number of shares in the submitted orders:
```
RecordingSink sink;
orderBook.setSink(&sink);

std::size_t submitted = 0U;
for (auto i = 0U; i < 1000000; ++i) {
  auto inputOrder = orderGen.generateOrder();
  if (orderBook.applyOrder(inputOrder).has_value()) {
    submitted += inputOrder.amount;
  }
}
```

After applying all the orders, I count the resting shares directly from the order book and the executed ones from the recording sink, and then compare the number of shares sent with the number accounted for:
```
std::size_t resting = 0U;
for (auto price = ob::MIN_PRICE_VALUE; price <= ob::MAX_PRICE_VALUE;
    ++price) {
  for (const auto& order : orderBook.getOrdersAtPrice(price).value()) {
    resting += order.amount;
  }
}

std::size_t executed = 0U;
for (const auto& executedOrder : sink.executions) {
  executed += executedOrder.amount;
}

EXPECT_EQ(submitted, resting + 2 * executed);
```

Another small integration test *checks the exact values of the executed orders*, to make sure the IDs and prices are correct:
```
RecordingSink sink;
orderBook.setSink(&sink);

orderBook.applyOrder(ob::sell(1, 100, 20));
orderBook.applyOrder(ob::buy(2, 120, 20));

ASSERT_EQ(sink.executions.size(), 1);
EXPECT_EQ(sink.executions[0].price, 100);
EXPECT_EQ(sink.executions[0].buyer, 2);
EXPECT_EQ(sink.executions[0].seller, 1);
```

The last test I came up with compares a properly configured `ob_logger::Logger` with a `RecordingSink`. I want to verify that *the file output contains the same values as a plain in-memory vector*.

For this test I defined a sink that wraps both an `ob_logger::Logger` and a `RecordingSink`:
```
class TeeSink : public ob::ExecutionSink {
 public:
  TeeSink(ob::ExecutionSink* sink_a, ob::ExecutionSink* sink_b)
      : m_sink_a(sink_a), m_sink_b(sink_b) {}

  void onExecuted(const ob::ExecutedOrder& eo) override {
    m_sink_a->onExecuted(eo);
    m_sink_b->onExecuted(eo);
  }

 private:
  ob::ExecutionSink *m_sink_a, *m_sink_b;
};
```

As before, I initialize the `TeeSink`, attach it to the order book, and start applying orders:
```
/* Sink A */
auto logger = ...

/* Sink B */
RecordingSink recordingSink;

/* Sinks A and B */
TeeSink sink{logger.get(), &recordingSink};

... 

ob::OrderBook orderBook{};
orderBook.setSink(&sink);

for (auto i = 0U; i < 100000; ++i) {
    orderBook.applyOrder(orderGen.generateOrder());
}
```

The comparison itself starts here. First, I compare the number of lines in the log file (which equals the number of executed orders) with the number of elements in the vector inside `recordingSink`:
```
{
  std::ifstream input{m_path};
  std::string line;
  std::size_t lineCount = 0U;
  while (std::getline(input, line)) {
    ++lineCount;
  }

  ASSERT_EQ(lineCount, recordingSink.executions.size());
}
```

If that check passes, the exact values can be compared. I used a regex to parse each individual line from the log file:
```
{
  const std::regex re{
    R"(^\[[\d-]{10} [\d:]{8}\.\d{3} UTC\] OPERATION=(BUY|SELL), BUYER=(\d+), SELLER=(\d+), PRICE=(\d+), AMOUNT=(\d+)$)"};

  auto to_int = [](const std::ssub_match& sm) {
    int v{};
    std::from_chars(&*sm.first, &*sm.second, v);
    return v;
  };

  std::ifstream input{m_path};
  std::string line;
  std::size_t i = 0U;

  while (std::getline(input, line)) {
    std::smatch m;
    ASSERT_TRUE(std::regex_match(line, m, re));

    EXPECT_EQ(m[1].str(), ob::to_string(recordingSink.executions[i].type));
    EXPECT_EQ(to_int(m[2]), recordingSink.executions[i].buyer);
    EXPECT_EQ(to_int(m[3]), recordingSink.executions[i].seller);
    EXPECT_EQ(to_int(m[4]), recordingSink.executions[i].price);
    EXPECT_EQ(to_int(m[5]), recordingSink.executions[i].amount);

    ++i;
  }
}
```

So that's how I put the whole program together, along with the integration tests that ensure the input (`InputOrder`s) produces the expected output (`ExecutedOrder`s).
