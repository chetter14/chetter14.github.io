---
layout: post
title: Algorithms and Data Structures. Sqrt Decomposition.
---

So, in my Master's program, I took a course *"Advanced Algorithms and Data Structures"*. At first, you should set up an environment on your local machine the way it is set up in their [Yandex Contest](https://contest.yandex.com/) environment. The template project they provided for such a task can be found [here](https://github.com/vityaman-edu/algocont). I forked from their repository and had to deal with a few problems to make the environment work on my machine. Namely, I changed: *EOL symbols* (because I am on Windows but the code is compiled and run on Linux), *scripts* a bit (used 'bash' instead of 'sh'), and other little things.

After getting the environment ready I got to the task - *"Implement an efficient data structure that allows you to modify array elements and compute the index of the k-th zero from the left in a given segment of the array"*. So, the input is *the array you work with and the queries (update, get) you handle on this array*. The general solution to this problem can be achieved by **sqrt-decomposition**. I am going to show the crucial parts of my solution, all the code base is resided [here](https://github.com/chetter14/algocont/tree/labs).

To start, we have to calculate our block length and the amount of blocks that we are going to split our array into. The *block length is a square root of an array size* rounded to an integer. But I *round* the block length value to *the value of the power of two*. The point is the following: the `block_len` value is used frequently in calculations (namely, divisions) and if the `block_len` value is, say, $$333$$ (for a large input array), then calculations *would be slow* and optimal performance won't be achieved. But with a value like $$256$$ divisions *go faster* (because it's shift right operation).
```
uint block_len = GetClosestPowerTwo(static_cast<int>(std::sqrt(size)));
uint blocks_amount = (size + block_len - 1) / block_len;
```

I want `Update()` operation to be $$O(1)$$ and I made it this way:
```
void Update(int index, int value) {
	if (arr_[index] == value) {  // Nothing changes
	  return;
	}

	// Update the block:
	int& bck_zero_count = blocks_[index / block_len_];
	if (arr_[index] == 0 && value != 0) {
	  --bck_zero_count;
	} else if (arr_[index] != 0 && value == 0) {
	  ++bck_zero_count;
	}

	// Update the array:
	arr_[index] = value;
	}
```

The `Get()` operation is $$O(sqrt(n))$$ now. In few words, the algorithm is to process the leftmost partial block element by element, jump over full blocks in between (handling them somehow), and process the rightmost partial block:
```
int Get(uint left, uint right, uint key) {
    uint zero_count = 0;

    // Process the leftmost partial block
    while (left <= right && (left % block_len_ != 0) && left != 0) {
      if (arr_[left] == 0 && ++zero_count == key) {
        return static_cast<int>(left) + 1;
      }
      ++left;
    }

    // Process full blocks in the middle
    uint cur_block = left / block_len_;
    while (left + block_len_ - 1 <= right && zero_count + blocks_[cur_block] < key) {
      zero_count += blocks_[cur_block];
      left += block_len_;
      cur_block = left / block_len_;
    }

    // Process the rightmost partial block
    while (left <= right) {
      if (arr_[left] == 0 && ++zero_count == key) {
        return static_cast<int>(left) + 1;
      }
      ++left;
    }

    return -1;
  }
```

With such a code I could fit into time and memory limits. Other captivating things I encountered are *`.clang-tidy` and `.clang-format`*. These files describe the style you must write your code in. In the beginning, I was stunned, mainly because of anger. But now I understand that it's *a strong tool* that should be applied in companies and universities. I think, it leads to *better maintenance* of the whole codebase and **acts as rules, laws, and protocols** in other engineering fields. It provides a *foundation* from which you can work in a sense of purely engineering field. 
