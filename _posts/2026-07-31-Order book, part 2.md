---
layout: post
title: Order book. Part 2.
---

Now let's get to the implementation details of the order book - the main module, where the matching logic resides. There are *five important functions* that require an explanation. I'll start with the smallest ones.

1) `advanceAsksBoundary()`. When a sell order is executed (a suitable buyer appears), we want to *move the index of the lowest ask up if the lowest ask is completely fulfilled*. So we *increment* the asks start index *until an ask with a higher price is found, or until MAX_PRICE_VALUE is reached*.

2) `retreatBidsBoundary()`. The opposite of `advanceAsksBoundary()`. When a buy order is executed (a suitable seller appears), we want to *move the index of the highest bid down if the highest bid is completely fulfilled*. So we *decrement* the bids start index *until a bid with a lower price is found, or until MIN_PRICE_VALUE is reached*.

3) `executeBid(const Order&, Price)`. We get here when the bid price is higher than or equal to the lowest ask price (the asks start index). *The bid is executed sequentially, starting from the asks start index. If, after executing the sell orders, the bid is still not fully executed, it is stored in the order book* for future orders. The bids start index is now at this price, and the asks start index is somewhere above it.

4) `executeAsk(const Order&, Price)`. The opposite of `executeBid(...)`. We get here when the ask price is lower than or equal to the highest bid price (the bids start index). *The ask is executed sequentially, starting from the bids start index. If, after executing the buy orders, the ask is still not fully executed, it is stored in the order book* for future orders. The asks start index is now at this price, and the bids start index is somewhere below it.

5) `applyOrder(const InputOrder&)`. Checks the order type, compares the input order price with the bids/asks start index and, depending on the case, handles the input order:

  - executes the order, or
  - does not execute the order and just adds it to the order book (possibly updating the bids/asks start index as well)

After implementing it, I wrote unit tests for the module. I won't dive into the details, but basically I tried out every scenario I could think of: an exact buy (sell) order empties the level of sell (buy) orders; when the bids and asks start indices cross, everything is processed correctly (nothing is superfluous or missing); and so on.

Another cool thing I did was modifying CMake to handle **sanitizers (ASan and TSan) and coverage**. I used an `INTERFACE` target that I later link to the target modules (i.e., to the order book library):
```cmake
# Set of flags used by all the targets
add_library(target_flags INTERFACE)

...

if (USE_ASAN AND USE_TSAN)
    message(FATAL_ERROR "Can't use ASAN and TSAN together!")
endif()

if (NOT MSVC)
    if (USE_ASAN)
        target_compile_options(target_flags INTERFACE -fsanitize=address,undefined -g)
        target_link_options(target_flags INTERFACE -fsanitize=address,undefined)
    endif()
    if (USE_TSAN)
        target_compile_options(target_flags INTERFACE -fsanitize=thread -g)
        target_link_options(target_flags INTERFACE -fsanitize=thread)
    endif()
    if (USE_COVERAGE)
        target_compile_options(target_flags INTERFACE --coverage -O0 -g)
        target_link_options(target_flags INTERFACE --coverage)
    endif()
endif()
```

Also, I added CMake *presets* for compiling the project with *g++ or clang++* (in debug/release mode, with sanitizers, with coverage).

I even wrote a *bash script for running code coverage*. It takes a bunch of steps, so I decided to automate it - if coverage is needed, you can just run the script.

```bash
./run_coverage.sh
```
