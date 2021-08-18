\subsection{Coroutines}\index{coroutines}
\index{C++20!coroutines}

At the time of writing this, I really don't know much about it. I'll explain the basics, but I will continue to look into this topic on my own time. In short, coroutines are functions that can be suspended and resumed mid-execution. They are stackless: they don't (nor could they) use a stack to store their activation records. Instead of the traditional subroutine model: a caller executes a subroutine from the start, waits until completion and then resumes, for a coroutine execution can be interleaved. The coroutines are more like partners, one makes progress, yields to the other, the second makes progress, yields to the first and so on. To this end they provide concurrency, but not parallelism. When you use the operators co\_return, co\_yield, and/or co\_await in a function, the compiler converts it to a coroutine with a finite state machine to control program flow and transfers. C++20 provides a pretty low-level API that's designed for library writers to build upon. For a good coroutine library, check out \href{https://github.com/lewissbaker/cppcoro}{cppcoro} and for more detailed information, check our \href{https://lewissbaker.github.io/}{Lewis Baker's github}.
\index{co\_await}\index{co\_yield}\index{co\_return}

Coroutines (henceforth referred to as "coro") can be resumed, suspended, and destroyed. Resuming a coro will transfer execution back at where it left off. Suspending one will save the current suspension point onto the activation frame and return control to the awaiter/caller. Destroying a coro cleans up the activation frame. The activation frame for coroutines are known as coroutine frames which contains space for parameters, local variables, temporaries, execution state (how to resume) and promises (for return values)

C++20 provides two interfaces for coros, a promise\_type and awaiter. Note that these interfaces are not interfaces in the OOP sense, but rather concepts with certain functions that the compiler calls when turning a function into a coroutine. It should also be noted that the coro promise is not the same as the concurrency promise.

An awaiter specifies three methods: await\_suspend(), await\_resume(), and await\_ready(). When you call co\_await on the object, the compiler first checks if the promise defines a function await\_transform(), which will be called if it exists otherwise the compiler just gets the awaiter directly.


await\_ready() returns a bool indicating if the next value to be produced by the coro is ready. Internally, when you co\_await an awaiter, it calls await\_ready(), if ready it will then call await\_resume() otherwise await\_suspend().

await\_suspend() takes in a handle to a coro, and can either return void or the handle to the coro to resume. The latter is known as a symmetric transfer and prevents allocating excess stack frames. This works by following a premise similar to tail recursion vs non-tail recursion. When a coro is suspended, the compiler stores all relevant state and creates a coroutine\_handle object.

await\_resume() is what produces the return type for a co\_await expression.

A promise\_type specifies methods for customizing the behavior of the coro itself. initial\_suspend() is called on the first suspension of the coro, final\_suspend(), yield\_value() (called on co\_yield), return\_value (called on co\_return), get\_return\_object() for returning the value, unhandled\_exception() which is called in a catch block for any exception the propagates out and some more. For intial and final suspend, the standard defined std::suspend\_always and std::suspend\_never can be returned to control suspension behavior.

co\_return takes the place of a normal return call, while co\_yield doesn't clean up the coro frame, but instead returns a value and resumes execution on the cooperating coro or caller.

The promise\_type is wrapped in an \inlinecode{std::coroutine_handle<>} which allows the holder to manage the coro and call operations such as resume(), destroy(), and done().

This probably made little sense, probably because I'm not even quite sure if it all made sense to me. So here's an example. A generator is something that evaluates lazily. If you are familiar with Python (I actually am not) range() is a generator. So that's what we'll do. We can do this without coros with just iterators, but where's the fun in that. The general structure will be that we have a Generator class with a promise type and an iterator. Each time the iterator is incremented, it resumes the coro which uses co\_yield to produce a value.
Let's first start by defining our promise type.
\begin{lstlisting}[language=C++]
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
        
        // this function called when on co_await
        // disable awaiting on the generator since
        // it can be an infinite stream of values
		void await_transform() = delete;
	};
\end{lstlisting}
We store the value of our coro as a variant, since it can hold either a value or an exception. we define the necessary functions to satify the promise\_type interface. Notice we define, but don't implement await\_transform which prevents anybody from calling co\_await on this promise.
Next we'll define the iterator:
\begin{lstlisting}[language=C++]
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
\end{lstlisting}
For an iterator in an enhanced for-loop, the compiler needs to be able to compare equality with == and !=, so we define those which delegate the comparisons to the handle. We also provide derefence capability, which will through if an exception is stored in the variant instead of a value. In the iterator incrementation, we resume the coro. If the coro finishes abruptly, then we know an error has occurred and rethrow the exception.
Here's the rest of the Generator:
\begin{lstlisting}[language=C++]
    // local variable handle to coro
	handle_type handle{ nullptr };

	friend void swap(Generator<T>& a, Generator<T>& b) {
		std::swap(a.handle, b.handle);
	}

	Generator() = default;

	Generator(handle_type& handle) :
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
\end{lstlisting}
We use RAII features to destroy the coro frame on destruction of the generator. We provide move capabilities and remove copy capabilities. Finally we provide begin() and end() to our generator so it can be used in a for loop. Notice our coroutine doesn't explictly create a Generator itself. Instead the compiler handles all of that and constructing the coro handle to pass to the generator. Here's the usage:
\begin{lstlisting}[language=C++]
int main() {
    for (auto i : range(10)) {
        printf("%d ", i);
    }
    printf("\n");
    return 0;
}
\end{lstlisting}

Here's another example. This one is a Task class that allows taks to be scheduled and lazily evaluated when you await them.
\begin{lstlisting}[language=C++]
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
\end{lstlisting}
A few things to point out is that main cannot be a coro. Thus no calls to co\_await, co\_yield etc. can occur inside it. Thus we define an operator() for getting the result of the task. We also use C++17's init-if syntax that allows you to form an if statement like so:\index{init-if}
\begin{lstlisting}[language=C++]
if(auto = /*initializer*/; /*condition*/){

}
\end{lstlisting}
The condition and following if-else block has scope of the initialized variable.
Usage of the task looks like the following:
\begin{lstlisting}[language=C++]
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
\end{lstlisting}
Once again, notice no co\_await calls were performed in main.