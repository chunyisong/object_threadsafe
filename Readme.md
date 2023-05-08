This repository for examples of articles

## How to make any object thread-safe with the speed of lock-free algorithms

`safe_ptr.h` contains C++ code of:

* `safe_ptr<>` - make any your custom object thread-safe, even when passed to a function

* `contention_free_shared_mutex<>` - 10 X times faster than `std::shared_mutex`

* `contfree_safe_ptr<>` - make any your custom object thread-safe with the speed of lock-free algorithms

----

| ![Performance, MOps](https://user-images.githubusercontent.com/4096485/79151220-c03c4180-7dd2-11ea-9b73-e281fa561a78.png) | ![Median Latency, usec](https://user-images.githubusercontent.com/4096485/79151246-caf6d680-7dd2-11ea-9527-db1a0c0a7364.png) |
|---|---|

----

### Additional contents for articles


**There are examples:**

* **any_object_threadsafe** - how to use `safe_ptr<>` - Article 1: https://www.codeproject.com/Articles/1183379/We-make-any-object-thread-safe
* **contfree_shared_mutex** - how to use `contention_free_shared_mutex<>` - Article 2: https://www.codeproject.com/Articles/1183423/We-make-a-std-shared-mutex-times-faster
* **examples** - all examples from online compilers in articles


**There are benchmarks:**

* **bench_contfree** - Benchmark contention free shared mutex - Article 2: https://www.codeproject.com/Articles/1183423/We-make-a-std-shared-mutex-times-faster

* **benchmark** - Benchmark contention free shared mutex (extended) - Article 3: https://www.codeproject.com/Articles/1183446/Thread-safe-std-map-with-the-speed-of-lock-free-ma

* **CDS_test** - Benchmark lock-free lib-CDS and `std::map` guarded by contention free shared mutex - Article 3: https://www.codeproject.com/Articles/1183446/Thread-safe-std-map-with-the-speed-of-lock-free-ma

* **Real_app_bench** - Benchmark lock-free lib-CDS and `std::map` guarded by contention free shared mutex

  (Simulates real application - added 20 usec delay between each inter-thread data exchange) - Article 3: https://www.codeproject.com/Articles/1183446/Thread-safe-std-map-with-the-speed-of-lock-free-ma


----

For your Pull Requests use a separate branch: https://github.com/AlexeyAB/object_threadsafe/tree/for_pull_requests

Before Pull Requests - please run all necessary benchmarks, preferably on a server CPU, such as in folders: CDS_test, Real_app_bench or benchmark - and insert images of the results charts.

---


1. How to use `safe_ptr<>` - in the same way as `std::shared_ptr<>`: http://coliru.stacked-crooked.com/a/ccf634f1a5e7f991


```cpp
// g++ -std=c++14 -O2 -pthread main.cpp && ./a.out
#include <iostream>
#include <thread>
#include <vector>
#include <string>
#include <map>

#include "safe_ptr.h"


int main() {
    sf::safe_ptr< std::map< std::string, int > > safe_map_string_int;

    std::thread t1([&]() { safe_map_string_int->emplace("apple", 1); });
    std::thread t2([&]() { safe_map_string_int->emplace("potato", 2); });
    t1.join(); t2.join();

    std::cout << "apple = " << (*safe_map_string_int)["apple"] <<
        ", potato = " << (*safe_map_string_int)["potato"] << std::endl;
    return 0;
}
```

----

2. How to use `sf::contention_free_shared_mutex` - in the same way as `std::shared_mutex`: http://coliru.stacked-crooked.com/a/11c191b06aeb5fb6

```cpp
#include <iostream>
#include <thread>
#include <vector>

#include "safe_ptr.h"


template <typename T>
void func(T &s_m, int &a, int &b)
{
	for (size_t i = 0; i < 100000; ++i)
	{
		// x-lock for modification
		{
			s_m.lock();
			a++;
			b++;
			s_m.unlock();
		}

		// s-lock for reading
		{
			s_m.lock_shared();
			assert(a == b);		// will never happen
			s_m.unlock_shared();
		}
	}
}

int main() {

	int a = 0;
	int b = 0;
	sf::contention_free_shared_mutex< > s_m;

	// 20 threads
	std::vector< std::thread > vec_thread(20);
	for (auto &i : vec_thread) i = std::move(std::thread([&]() { func(s_m, a, b); }));
	for (auto &i : vec_thread) i.join();

	std::cout << "a = " << a << ", b = " << b << std::endl;
	getchar();

	return 0;
}
```

----

2.1. How to use `contfree_safe_ptr< std::map >`: http://coliru.stacked-crooked.com/a/b78467b7a3885e5b

```cpp
#include "safe_ptr.h"

sf::contfree_safe_ptr< std::map<std::string, int> > safe_map_strings_global;   // cont-free shared-mutex


template<typename T>
void func(sf::contfree_safe_ptr<T> safe_map_strings)
{
    // read-only (shared lock during access)
    sf::contfree_safe_ptr<T> const &readonly_safe_map_string = safe_map_strings;

    for (size_t i = 0; i < 100000; ++i)
    {
        // two Shared locks (recursive)
        assert(readonly_safe_map_string->at("apple") == readonly_safe_map_string->at("potato"));

        std::lock_guard<decltype(safe_map_strings)> lock(safe_map_strings); // 1-st eXclusive lock
        safe_map_strings->at("apple") += 1;                                 // 2-nd recursive eXclusive lock
        safe_map_strings->find("potato")->second += 1;                      // 3-rd recursive eXclusive lock
    }
}

int main() {

    (*safe_map_strings_global)["apple"] = 0;
    (*safe_map_strings_global)["potato"] = 0;

    std::vector<std::thread> vec_thread(10);
    for (auto &i : vec_thread) i = std::move(std::thread([&]() { func(safe_map_strings_global); }));
    for (auto &i : vec_thread) i.join();

    std::cout << "END: potato is " << safe_map_strings_global->at("potato") <<
        ", apple is " << safe_map_strings_global->at("apple") << std::endl;


    return 0;
}
```


----

3. High-performance usage of `contfree_safe_ptr< std::map >` and `safe_obj<>`: http://coliru.stacked-crooked.com/a/f7079aa1acc00a78

```cpp
#include "safe_ptr.h"

using namespace sf;

struct field_t { int money, time; field_t(int m, int t) : money(m), time(t) {} field_t() : money(0), time(0) {} };
typedef safe_obj<field_t, spinlock_t> safe_obj_field_t;


contfree_safe_ptr< std::map<int, safe_obj_field_t> > safe_map_contfree_rowlock_global;



template<typename T>
void benchmark_safe_ptr_rowlock(T safe_map, size_t const iterations_count, size_t const container_size)
{
    const unsigned int seed = (unsigned)std::chrono::system_clock::now().time_since_epoch().count();
    std::default_random_engine generator(seed);
    std::uniform_int_distribution<size_t> index_distribution(0, container_size);

    enum { insert_op, delete_op, update_op, read_op };
    std::uniform_int_distribution<size_t> operation_distribution(insert_op, read_op);    // 0 - 4

    for (size_t i = 0; i < iterations_count; ++i)
    {
        int const rnd_index = index_distribution(generator);    // 0 - container_size
        int const num_op = operation_distribution(generator);   // insert_op, update_op, delete_op, read_op

        switch (num_op) {
        case insert_op: {
            slock_safe_ptr(safe_map)->find(rnd_index);  // find for pre-cache to L1 with temprorary S-lock

            safe_map->emplace(rnd_index, (field_t(rnd_index, rnd_index)));  // insert with X-lock on Table
            break;
        }
        case delete_op: {
            slock_safe_ptr(safe_map)->find(rnd_index);  // find for pre-cache to L1 with temprorary S-lock

            size_t erased_elements = safe_map->erase(rnd_index);    // erase with X-lock on Table
        }
            break;
        case update_op: {
            auto s_safe_map = slock_safe_ptr(safe_map); // S-lock on Table
            auto it = s_safe_map->find(rnd_index);
            if (it != s_safe_map->cend()) {
                auto x_field = xlock_safe_ptr(it->second);
                x_field->money += rnd_index;   // X-lock on field, still S-lock on Table (must necessarily be)
            }
        }
            break;
        case read_op: {
            auto s_safe_map = slock_safe_ptr(safe_map); // S-lock on Table
            auto it = s_safe_map->find(rnd_index);
            if (it != s_safe_map->cend()) {
                auto s_field = slock_safe_ptr(it->second);
                volatile int money = s_field->money;   // S-lock on field, still S-lock on Table (must be)
                // volatile here only to avoid optimization for unused money-variable
            }
        }
            break;
        default: std::cout << "\n wrong way! \n";  break;
        }
    }
}


int main() {

    const size_t iterations_count = 100000;     // operations of data exchange between threads
    const size_t container_size = 100000;       // elements in container
    std::vector<std::thread> vec_thread(8);     // threads number
    std::cout << "Threads: " << vec_thread.size() << std::endl;

    std::cout << "Testing safe<map,contf>rowlock... \n";
    for (auto &i : vec_thread) i = std::move(std::thread([&]() {
        benchmark_safe_ptr_rowlock(safe_map_contfree_rowlock_global, iterations_count, container_size);
    }));
    for (auto &i : vec_thread) i.join();

    std::cout << "Successfully completed." << std::endl;
    return 0;
}
```

----

4. Disadvantages of the Execute Around Idiom

However, implicit locks were used for Insert and Delete operations. Our safe_ptr<std::map> object was automatically locked by using the Execute Around Pointer idiom and was unlocked immediately after termination of the Insert or Delete operation.

Example: [40] http://coliru.stacked-crooked.com/a/89f5ebd2d96296d3

But you can forget to use explicit locks in the Update or Read operations. In this case, safe_ptr<std::map> will be unlocked immediately after completion of search operation, and then you continue to use:

the found iterator, which can be invalidated by another thread
or a found element that can be deleted by another thread
To partially solve this problem, instead of safe_ptr<> and safe_obj<>, you can use safe_hide_ptr<> and safe_hide_obj<> - they do not use the Execute Around Pointer and you can access the class members only after explicit locking:

```C++
safe_hide_obj<field_t, spinlock_t=""> field_hide_tmp;
  //safe_obj<field_t, spinlock_t=""> &field_tmp = field_hide_tmp; // conversion denied -
                                                                  // compile-time error

  //field_hide_tmp->money = 10;                     // access denied - compile-time error
  auto x_field = xlock_safe_ptr(field_hide_tmp);    // locked until x_field is alive
  x_field->money = 10;                              // access granted
</field_t,></field_t,>
```
```c++

template<typename T> using contfree_safe_ptr = safe_ptr<T, contention_free_shared_mutex<>,
    std::unique_lock<contention_free_shared_mutex<>>, shared_lock_guard<contention_free_shared_mutex<>> >;

template<typename T> using contfree_safe_hide_ptr = safe_hide_ptr<T, contention_free_shared_mutex<>,
    std::unique_lock<contention_free_shared_mutex<>>, shared_lock_guard<contention_free_shared_mutex<>> >;
// ---------------------------------------------------------------



struct field_t { int money, time; field_t(int m, int t) : money(m), time(t) {} field_t() : money(0), time(0) {} };
typedef safe_hide_obj<field_t, spinlock_t> safe_obj_field_t;


//contfree_safe_ptr< std::map<int, safe_obj_field_t> > safe_map_contfree_rowlock_global;
contfree_safe_hide_ptr< std::map<int, safe_obj_field_t> > safe_map_contfree_rowlock_global;



template<typename T>
void benchmark_safe_ptr_rowlock(T safe_map, size_t const iterations_count, size_t const container_size)
{
    const unsigned int seed = (unsigned)std::chrono::system_clock::now().time_since_epoch().count();
    std::default_random_engine generator(seed);
    std::uniform_int_distribution<size_t> index_distribution(0, container_size);

    enum { insert_op, delete_op, update_op, read_op };
    std::uniform_int_distribution<size_t> operation_distribution(insert_op, read_op);    // 0 - 4

    for (size_t i = 0; i < iterations_count; ++i)
    {
        int const rnd_index = index_distribution(generator);    // 0 - container_size
        int const num_op = operation_distribution(generator);   // insert_op, update_op, delete_op, read_op

        switch (num_op) {
        case insert_op: {
            xlock_safe_ptr(safe_map)->emplace(rnd_index, (field_t(rnd_index, rnd_index)));  // insert with X-lock on Table
            break;
        }
        case delete_op: {
            size_t erased_elements = xlock_safe_ptr(safe_map)->erase(rnd_index);    // erase with X-lock on Table
        }
            break;
        case update_op: {
            auto s_safe_map = slock_safe_ptr(safe_map); // S-lock on Table
            auto it = s_safe_map->find(rnd_index);
            if (it != s_safe_map->cend()) {
                auto x_field = xlock_safe_ptr(it->second);
                x_field->money += rnd_index;   // X-lock on field, still S-lock on Table (must necessarily be)
            }
        }
            break;
        case read_op: {
            auto s_safe_map = slock_safe_ptr(safe_map); // S-lock on Table
            auto it = s_safe_map->find(rnd_index);
            if (it != s_safe_map->cend()) {
                auto s_field = slock_safe_ptr(it->second);
                volatile int money = s_field->money;   // S-lock on field, still S-lock on Table (must necessarily be)
                // volatile here only to avoid optimization for unused money-variable
            }
        }
            break;
        default: std::cout << "\n wrong way! \n";  break;
        }
    }
}

```

Example: [41] http://coliru.stacked-crooked.com/a/d65deacecfe2636b

Here, for thread-safe work with the row, we used safe_obj. Inside safe_obj<T>, there is an object of T type, but not a pointer to it as in safe_ptr<T>. Therefore, when using safe_obj, you do not need to allocate memory separately for the object itself and change the atomic reference count, as it is required in safe_ptr. Therefore, the Insert and Delete operations are much faster with safe_obj than with safe_ptr.

It is worth noting that during copying safe_obj, it is not the pointer that is copied to the object, but it is the object itself being copied. There, the source and the final safe_obj is pre-locked.

Earlier, you could make a mistake and write the following - an error code:

```C++
auto it = safe_map->find(rnd_index); // X-lock, find(), X-unlock
if (it != s_safe_map->cend())
    volatile int money = it->second ->money;  // X-lock, operator=(), X-unlock
```
but now, such a request is not compiled and will require explicit locking of objects - the correct code:

```C++
auto s_safe_map = slock_safe_ptr(safe_map); // S-lock on Table
auto it = s_safe_map->find(rnd_index);
if (it != s_safe_map->cend()) {
    auto s_field = slock_safe_ptr(it->second);
    volatile int money = s_field->money;   // S-lock on field, still S-lock on Table
    // volatile here only to avoid optimization for unused money-variable
} // S-unlock Table, S-unlock field
```
However, you still have the danger of using locks as temporary objects - not correct:

```C++
auto it = slock_safe_ptr(safe_map)->find(rnd_index); // S-lock, find(), S-unlock
if (it != s_safe_map->cend()) {
    volatile int money = slock_safe_ptr(it->second)->money;   // S-lock, =, S-unlock
}
```
You have a choice:

To use safe_ptr and safe_obj in order to be able to explicitly or automatically (Execute Around Idiom) lock your object
Or to use safe_hide_ptr and safe_hide_obj, leaving only the ability to explicitly lock your object
It’s up to you what to choose:

to use convenient automatic locking (Execute Around Idiom)
or slightly to reduce the possibility of an error by explicitly locking an object