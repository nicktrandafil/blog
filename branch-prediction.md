# Faster algorithms with predictable data

Programming languages abstract you away from the hardware, so that you can write programs without knowing the hardware details. However, having some insight into hardware techniques can help you to write more efficient programs.

One of these techniques is branch prediction. Simply put, hardware starts executing one of the branches of an `if` statement before knowing the result of the condition. If the prediction is correct, the program runs faster; else the CPU has to discard the work and start over, which is a waste. The algorithm runs faster if the the hardware predicts the branches correctly most of the time. The perfect program wouldn't have branches at all, but when it isn't possible, simply organizing the data so that each branch is executed in a sequence will improve the performance. This idea is demonstrated in the following examples.

Types used do emulate the problem:

```
enum class MovingStyle {
	hop,
	crawl,
	walk,
};

struct Animal {
	int id;
	MovingStyle style;
	int position;
	int energy_tank;
};
```

The input data is generated randomly:

```
inline void generate_data(std::vector<Animal>& animals) {
	std::mt19937 gen(0);

	std::uniform_int_distribution<int> style(1, 3);	
	std::uniform_int_distribution<int> energy_tank(0, 100);

	animals.resize(100'000);
	std::generate(begin(animals), end(animals), [&, id = 0]() mutable {
			return Animal {
				.id = ++id,
				.style = [&] {
					switch (style(gen)) {
					case 1: return MovingStyle::hop;
					case 2: return MovingStyle::crawl;
					case 3: return MovingStyle::walk;
					}
					std::abort();
				}(),
				.energy_tank = energy_tank(gen),
			};
	});
}
```

The natural solution is to iterate through the elements using a `for` loop and use a `switch` statement to select the appropriate function to dispatch to.

```
inline void for_each_if(std::vector<Animal>& animals) noexcept {
	for (auto& x : animals) {
		switch (x.style) {
			case MovingStyle::hop:
				hop(x);
				break;
			case MovingStyle::crawl:
				crawl(x);
				break;
			case MovingStyle::walk:
				walk(x);
				break;
		}
	}
}
```

But, partitioning the data so that the elements of the same type are contiguous allows the appropriate function to run on each range without `if`s.

```
inline void partition_then_for_each_range(std::vector<Animal>& animals) noexcept {
	auto e = std::partition(begin(animals), end(animals), [](auto const& x) { return x.style == MovingStyle::hop; });
	for (auto& x: std::ranges::subrange(begin(animals), e)) {
		hop(x);
	}

	auto e2 = std::partition(e, end(animals), [](auto const& x) { return x.style == MovingStyle::crawl; });
	for (auto& x: std::ranges::subrange(e, e2)) {
		crawl(x);
	}

	for (auto& x: std::ranges::subrange(e2, end(animals))) {
		walk(x);
	}
}
```

Benchmarks:

```console
---------------------------------------------------------------------------
Benchmark                                 Time             CPU   Iterations
---------------------------------------------------------------------------
BM_for_each_if                       459378 ns       458366 ns         1551
BM_partition_then_for_each_range     109564 ns       109562 ns         6495
```

This algorithm is faster by 4-5 times. The results might feel counter intuitive. For sure, there are branches inside `std::partition`, so how it is that `partition_then_for_each_range` traverses the data two times more than `for_each_if` but performs faster?

I have some intuitive explanation, but I'm not sure if it is valid. Consider how partition works. Depending on the condition, two elements may get swapped or not. This means, that in case of misprediction, no additional work is required apart from discarding the done work, whereas in `for_each_if`, the other branch still needs to be calculated. This means, that both algorithms have the same amount of branches and mispredictions, but the cost of misprediction is lower in `partition_then_for_each_range`.

Indeed, see what happens if we pass already partitioned data to the algorithms. We'll change the data generation function to produce predictable data.

```
#ifdef P
					if (id < 30'000) {
						return MovingStyle::hop;
					} else if (id < 60'000) {
						return MovingStyle::crawl;
					} else {
						return MovingStyle::walk;
					}
#else
					switch (style(gen)) {
					case 1: return MovingStyle::hop;
					case 2: return MovingStyle::crawl;
					case 3: return MovingStyle::walk;
					}
					std::abort();
#endif
```

Benchmarks:

```
---------------------------------------------------------------------------
Benchmark                                 Time             CPU   Iterations
---------------------------------------------------------------------------
BM_for_each_if                        47139 ns        47138 ns        14470
BM_partition_then_for_each_range     108215 ns       107946 ns         6498
```

The `BM_for_each_if` became about 10 times faster than the same algorithm on the random data and about 2 times faster than the `partition_then_for_each_range`. With predictable data, `partition_then_for_each_range` brings no benefit and has some overhead of partitioning, but see how stable the benchmarks are across sorted and random data `109'564` and `108'215`? That's probably a good property to have in some cases.

The `partition_then_for_each_range` and the following algorithm, which is a composition of `std::partition` and `for_each_if` 
have the same performance, because the `if` in the loop becomes predictable:

```
inline void partition_then_for_each_if(std::vector<Animal>& animals) noexcept {
	auto const e = std::partition(begin(animals), end(animals), [](auto const& x) { return x.style == MovingStyle::hop; });
	std::partition(e, end(animals), [](auto const& x) { return x.style == MovingStyle::crawl; });
	for_each_if(animals);
}
```

```
---------------------------------------------------------------------------
Benchmark                                 Time             CPU   Iterations
---------------------------------------------------------------------------
BM_for_each_if                       459378 ns       458366 ns         1551
BM_partition_then_for_each_range     109564 ns       109562 ns         6495
BM_partition_then_for_each_if        123218 ns       123217 ns         5650
```

## Conclusion

Moving expensive branches out of loops or making the branches predictable improved the performance in the the demonstrational problem. It is may be understood intuitively in the analogy of how you do some routine repetitive work better when the repetition is simple and is the same each time.

## All code in one snippet

```
// @begin
#include <benchmark/benchmark.h>

#include <memory>
#include <random>
#include <ranges>
#include <vector>

enum class MovingStyle {
	hop,
	crawl,
	walk,
};

struct Animal {
	int id;
	MovingStyle style;
	int position;
	int energy_tank;
	
	bool operator==(Animal const&) const noexcept = default;
};

inline void generate_data(std::vector<Animal>& animals) {
	std::mt19937 gen(0);

	std::uniform_int_distribution<int> style(1, 3);	
	std::uniform_int_distribution<int> energy_tank(0, 100);

	animals.resize(100'000);
	std::generate(begin(animals), end(animals), [&, id = 0]() mutable {
			return Animal {
				.id = ++id,
				.style = [&] {
#ifdef P
					if (id < 30'000) {
						return MovingStyle::hop;
					} else if (id < 60'000) {
						return MovingStyle::crawl;
					} else {
						return MovingStyle::walk;
					}
#else
					switch (style(gen)) {
					case 1: return MovingStyle::hop;
					case 2: return MovingStyle::crawl;
					case 3: return MovingStyle::walk;
					}
					std::abort();
#endif
				}(),
				.energy_tank = energy_tank(gen),
			};
	});
}

inline void hop(Animal& x) noexcept {
	while (0 < x.energy_tank) {
		x.position += 3;
		x.energy_tank -= 3;
	}
}

inline void crawl(Animal& x) noexcept {
	while (0 < x.energy_tank) {
		x.position += 2;
		x.energy_tank -= 2;
	}
}

inline void walk(Animal& x) noexcept {
	while (0 < x.energy_tank) {
		x.position += 1;
		x.energy_tank -= 1;
	}
}

inline void for_each_if(std::vector<Animal>& animals) noexcept {
	for (auto& x : animals) {
		switch (x.style) {
			case MovingStyle::hop:
				hop(x);
				break;
			case MovingStyle::crawl:
				crawl(x);
				break;
			case MovingStyle::walk:
				walk(x);
				break;
		}
	}
}

inline void partition_then_for_each_range(std::vector<Animal>& animals) noexcept {
	auto e = std::partition(begin(animals), end(animals), [](auto const& x) { return x.style == MovingStyle::hop; });
	for (auto& x: std::ranges::subrange(begin(animals), e)) {
		hop(x);
	}

	auto e2 = std::partition(e, end(animals), [](auto const& x) { return x.style == MovingStyle::crawl; });
	for (auto& x: std::ranges::subrange(e, e2)) {
		crawl(x);
	}

	for (auto& x: std::ranges::subrange(e2, end(animals))) {
		walk(x);
	}
}

inline void partition_then_for_each_if(std::vector<Animal>& animals) noexcept {
	auto const e = std::partition(begin(animals), end(animals), [](auto const& x) { return x.style == MovingStyle::hop; });
	std::partition(e, end(animals), [](auto const& x) { return x.style == MovingStyle::crawl; });
	for_each_if(animals);
}

static void BM_for_each_if(benchmark::State& state) {
	std::vector<Animal> animals;
	generate_data(animals);

	for (auto _ : state) {
		for_each_if(animals);
		benchmark::DoNotOptimize(animals);
	}
}
BENCHMARK(BM_for_each_if);

static void BM_partition_then_for_each_range(benchmark::State& state) {
	std::vector<Animal> animals;
	generate_data(animals);

	for (auto _ : state) {
		partition_then_for_each_range(animals);
		benchmark::DoNotOptimize(animals);
	}
}
BENCHMARK(BM_partition_then_for_each_range);

static void BM_partition_then_for_each_if(benchmark::State& state) {
	std::vector<Animal> animals;
	generate_data(animals);

	for (auto _ : state) {
		partition_then_for_each_if(animals);
		benchmark::DoNotOptimize(animals);
	}
}
BENCHMARK(BM_partition_then_for_each_if);

int main(int argc, char** argv) {
	// test for correctness
	{
		std::vector<Animal> x;
		generate_data(x);
		auto y = x;

		for_each_if(x);
		partition_then_for_each_range(y);

		std::ranges::sort(x, [](auto const& x, auto const& y) { return x.id < y.id; });
		std::ranges::sort(y, [](auto const& x, auto const& y) { return x.id < y.id; });
		assert(x == y);
	}

	benchmark::Initialize(&argc, argv);
	benchmark::RunSpecifiedBenchmarks();
}
// @end

// unordered data benchmark:
// 	Mac: `clang++ -O2 -std=c++20 -I/opt/homebrew/Cellar/google-benchmark/1.8.3/include -L /opt/homebrew/Cellar/google-benchmark/1.8.3/lib/ -lbenchmark branch-prediction.cpp -o bp`
// 	./bp
//
//	Vim in-place: `:1|/@begin/,/@end/!gcc -x c++ -c -O2 -std=c++20 -o /tmp/bp.o - && gcc -o /tmp/bp /tmp/bp.o -lbenchmark -lstdc++ -lm && /tmp/bp`
//

// ordered data benchmark
// 	Mac: clang++ -DP -O2 -std=c++20 -I/opt/homebrew/Cellar/google-benchmark/1.8.3/include -L /opt/homebrew/Cellar/google-benchmark/1.8.3/lib/ -lbenchmark branch-prediction.cpp -o bp
//	./bp
//
//	Vim in-place: `:1|/@begin/,/@end/!gcc -x c++ -c -DP -O2 -std=c++20 -o /tmp/bp.o - && gcc -o /tmp/bp /tmp/bp.o -lbenchmark -lstdc++ -lm && /tmp/bp`
```
