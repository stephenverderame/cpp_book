# Coroutines

At the time of writing this, I really don't know much about these. I'll explain the basics, but I will continue to look into this topic on my own time.
In short, coroutines are functions that can be suspended and resumed mid-execution.
They are stackless: they don't use a stack to store their activation records.
I believe this is a quality of C++20 coroutines and not a requirement of coroutines in general.
Instead of the traditional subroutine model: a caller executes a subroutine from the start,
waits until completion and then resumes, a coroutine execution can be interleaved.
The coroutines are more like partners, one makes progress, yields to the other, the second makes progress, yields to the first and so on.
To this end, they provide concurrency, but not parallelism.
When you use the operators `co_return`, `co_yield`, and/or `co_await` in a function,
the compiler converts it to a coroutine with a finite state machine to control program flow and transfers.
C++20 provides a pretty low-level API that's designed for library writers to build upon.
For a good coroutine library, check out [cppcoro](https://github.com/lewissbaker/cppcoro) and for more detailed information, check our [Lewis Baker's github](https://lewissbaker.github.io/).

Coroutines (henceforth referred to as "coro") can be resumed,
suspended, and destroyed.
Resuming a coro will transfer execution back to it at where it left off.
Suspending one will save the current suspension point onto the activation frame and return control to the awaiter/caller.
Destroying a coro cleans up the activation frame.
The activation frame for coroutines are known as coroutine frames which contains space for parameters, local variables, temporaries, execution state (how to resume) and promises (for return values).

C++20 provides two interfaces for coros, a `promise_type` and `awaiter`.
Note that these interfaces are not interfaces in the OOP sense, but rather concepts with certain functions (static interfaces) that the compiler calls when turning a function into a coroutine.
It should also be noted that the coro promise is not the same as the concurrency promise.

An awaiter specifies three methods: `await_suspend()`, `await_resume()`, and `await_ready()`.
When you call `co_await` on the object, the compiler first checks if the promise defines a function `await_transform()`, which will be called if it exists otherwise the compiler just gets the awaiter directly.

`await_ready()` returns a bool indicating if the next value to be produced by the coro is ready.
Internally, when you `co_await` on an awaiter, it calls `await_ready()`, if it returns true, it will then call `await_resume()`, otherwise `await_suspend()` is called.

`await_suspend()` takes in a handle to a coro, and can either return `void` or the handle to the coro to resume.
The latter is known as a *symmetric transfer* and prevents allocating excess stack frames.
This works by following a premise similar to tail recursion vs non-tail recursion.
When a coro is suspended, the compiler stores all relevant state and creates a `coroutine_handle` object.

`await_resume()` is what produces the return type for a `co_await` expression.

A `promise_type` specifies methods for customizing the behavior of the coro itself.
The `promose_type` interface has a few methods including `initial_suspend()`, `final_suspend()`, `yield_value()`, `return_value()`, `get_return_object()`, and `unhandled_exception()`.
`initial_suspend()` is called on the first suspension of the coro, `final_suspend()` and `yield_value()` are called from `co_yield`, `return_value()` is called from `co_return`,
and `get_return_object()` is for returning the result of the coro.
`unhandled_exception()` is called in a catch block for any exception the propagates out.
For initial and final suspend, the standard defined `std::suspend_always` and `std::suspend_never` can be returned to control suspension behavior.

`co_return` takes the place of a normal `return` call, while `co_yield` doesn't clean up the coro frame, but instead returns a value and resumes execution on the cooperating coro or caller.

The `promise_type` is wrapped in an `std::coroutine_handle<>` which allows the holder to manage the coro and call operations such as `resume()`, `destroy()`, and `done()`.

This probably made little sense, probably because I'm not even quite sure if it all made sense to me.
So here's an example. A generator is something that evaluates lazily. If you are familiar with Python `range()` is an example of a generator.
SSo for this example we'll implement a `range()` function with coroutines.
We can do this without coros with just iterators, but where's the fun in that.
The general structure will be that we have a `Generator` class with a promise type and an iterator.
Each time the iterator is incremented, it resumes the coro which uses `co_yield` to produce a value.
Let's first start by defining our promise type.
```C++
template<typename T>
struct Generator {
	// must be named promise_type
	struct promise_type {
		std::variant<T const*, std::exception_ptr> value;

		void throwIfException() {
			if (value.index() == 1) {
				std::rethrow_exception(std::get<1>(value));
			}
		}

		//Functions required for a coroutine promise

		std::suspend_always initial_suspend() {
			// suspend on creation to step through manually
			return {};
		}
		std::suspend_always final_suspend() noexcept {
			// suspend on return instead of destroying state
			return {};
		}
		std::suspend_always yield_value(T const& other) {
			value = std::addressof(other);
            // assign data to value

            // std::addressof() gets the address of a variable even if someone
            // (evily) overloaded `operator&`
			return {};
		}

		void return_void() {}

        // gets the return object from the promise
		Generator<T> get_return_object() {
			return Generator<T>(handle_type::from_promise(*this));
		}
		// called by compiler if coro produces an exception
		void unhandled_exception() {
			value = std::current_exception();
		}
        
        // this function called by co_await
        //
        // disable awaiting on the generator since
        // it can be an infinite stream of values
		void await_transform() = delete;
	};
```
We store the value of our coro as a variant, since it can hold either a value or an exception.
We define the necessary functions to satisfy the `promise_type` interface.
Notice we define, but don't implement `await_transform` which prevents anybody from calling `co_await` on this promise.
Next we'll define the iterator:
```C++
	using handle_type = std::coroutine_handle<promise_type>;
	struct iterator {
		handle_type handle;
		
		// iterator trait typedefs
		using iterator_category = std::input_iterator_tag;
		using value_type = T;
		using difference_type = ptrdiff_t;
		using pointer = T const*;
		using reference = T const&;

		iterator& operator++() {
		    // resume the coro
			handle.resume();

			if (handle.done()) {
				auto & pro = handle.promise();
				handle = nullptr;
				pro.throwIfException();
			}
			return *this;
		}

		bool operator==(const iterator& other) const {
			return handle == other.handle;
		}

		bool operator!=(const iterator& other) const {
			return !(*this == other);
		}

		T const& operator*() {
			return *std::get<0>(handle.promise().value);
		}

		T const& operator->() {
			return operator*();
		}
	};
```
For an iterator in an enhanced for-loop, the compiler needs to be able to compare equality with `==` and `!=`,
so we define those which delegate the comparisons to the handle.
We also provide derefence capability, which will throw if an exception is stored in the variant instead of a value.
In the iterator incrementation, we resume the coro. If the coro finishes abruptly, then we know an error has occurred and rethrow the exception.
Here's the rest of the Generator:
```C++
    // local variable handle to coro
	handle_type handle{ nullptr };

	friend void swap(Generator<T>& a, Generator<T>& b) {
		std::swap(a.handle, b.handle);
	}

	Generator() = default;

	Generator(const handle_type& handle) :
		handle(handle) {};

	Generator(handle_type&& handle) :
		handle(std::move(handle)) {}

	Generator(Generator<T>&& other) {
		swap(*this, other);
	}

	Generator& operator=(Generator<T>&& other) {
		swap(*this, other);
		return *this;
	}

	Generator(Generator<T>&) = delete;
	Generator<T>& operator=(Generator<T>&) = delete;

	~Generator() {
		if (handle)
			handle.destroy();
	}

	iterator begin() {
		handle.resume();

		if (handle.done()) {
			handle.promise().throwIfException();
			return { nullptr };
		}

		return { handle };
	}
	iterator end() {
		return { nullptr };
	}
};

template<typename T>
Generator<T> range(T begin, T end) {
	while (begin != end) {
		co_yield begin++;
	}
}
template<typename T>
Generator<T> range(T end) {
	T begin{};
	while (begin != end) {
		co_yield begin++;
	}
}
```
We use RAII features to destroy the coro frame on destruction of the generator.
We provide move capabilities and remove copy capabilities.
Finally, we provide `begin()` and `end()` methods to our generator so it can be used in a for loop.
Notice our coroutine doesn't explicitly create a Generator itself.
Instead, the compiler handles all of that and constructs the coro handle to pass to the generator. Here's the usage:
```C++
int main() {
    for (auto i : range(10)) {
        printf("%d ", i);
    }
    printf("\n");
    return 0;
}
```

Here's another example. This one is a Task class which allows tasks to be scheduled and lazily evaluated when you await them.

```C++
template<typename T>
struct Task {
	struct promise_type;
	using handle_t = std::coroutine_handle<promise_type>;
	
	// promise type
	// similar as last time
	struct promise_type {
		std::variant<T, std::exception_ptr> value;
		std::coroutine_handle<> previous;

		Task get_return_object() {
			return { handle_t::from_promise(*this) };
		}
		std::suspend_always initial_suspend() { return {}; }
		struct final_awaiter {
			bool await_ready() noexcept { return false; }
			void await_resume() noexcept {}
			std::coroutine_handle<> await_suspend(handle_t handle) 
			    noexcept 
			{
				if (auto prev 
				    = handle.promise().previous; prev) {
				    //symmetric transfer to "caller" if there was one
				    //this is known as init-if syntax and
				    //is in C++17
					return prev;
				}
				else
					return std::noop_coroutine();
			}
		};
		final_awaiter final_suspend() noexcept {
			return {};
		}
		void unhandled_exception() {
			value = std::current_exception();
		}
		void return_value(T&& val) {
		    // sets value on co_return
			value = std::forward<T>(val);
		}

	};

	handle_t handle;


	Task(handle_t h) :
		handle(h) {};
	~Task() {
		if (handle)
			handle.destroy();
	}
	//move operations not defined
	
	
	struct awaiter {
		handle_t handle;
		// always execute the coro
		bool await_ready() noexcept { return false; }
		
		// this is the return value of an 
		// co_await expression
		// rethrow if we caught an error
		T await_resume() {
			auto var = handle.promise().value;
			if (var.index() == 1)
				std::rethrow_exception(std::get<1>(var));
			else
				return std::get<0>(var);
		}
		auto await_suspend(handle_t h) {
			handle.promise().previous = h;
			return handle;
		}
	};
	// overload co_await
	awaiter operator co_await() {
		return awaiter{ handle };
	}
	
	// get result from a non coroutine function
	// main cannot be a coro
	T operator()() {
		handle.resume();
		auto var = handle.promise().value;
		if (var.index() == 0)
			return std::get<0>(var);
		else
			std::rethrow_exception(std::get<1>(var));
	}
};
```
A few things to point out is that `main()` cannot be a coro.
Thus, no calls to `co_await`, `co_yield` etc. can occur inside it.
To circumvent this in our simply example, we define an `operator()` for getting the result of the task from a normal function.
We also use C++17's init-if syntax that allows you to form an if statement like so:
```C++
if(auto = /*initializer*/; /*condition*/){

}
```
The condition block and following if-else block has scope of the initialized variable.
Usage of the task looks like the following:
```C++
Task<int> getNum() {
    co_return 5;
}
Task<int> getCollector() {
    auto v1 = getNum();
    auto v2 = getNum();
    co_return co_await v1 + co_await v2;
}
int main() {
    auto result = getCollector();
    printf("%d \n", result());
    return 0;
}
```
Once again, notice no `co_await` calls were performed in main.