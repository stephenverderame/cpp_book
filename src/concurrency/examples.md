## Linked List with Hand-Over-Hand Locking
```C++
#include <shared_mutex>
teamplate<typename T>
class LinkedList
{
    struct node {
        std::shared_mutex localLock;
        T data; ///< Invalid iff empty is true
        bool empty;

        std::unique_ptr<node> next;
        ///Invariant: next is null iff empty is true

        node() : data(), empty(true), next(nullptr) {}

        template<typename U>
        node(U&& data) : data(std::forward<T>(data)), empty(false),
            next(std::make_unique<node>()) {};

        template<typename U>
        void set(U&& data) {
            this->data = std::forward<T>(data);
            empty = false;
            if (!next) {
                next = std::make_unique<node>();
            }
        }
    };

    std::unique_ptr<node> root;
private:
    template<typename U&&>
    void unique_push(std::unique_ptr<node> * currentLast, U&& data) {
        std::unique_lock lk1(currentLast->get()->localLock);
        while (!currentLast->get()->empty) {
            currentLast = &currentLast->get()->next;
            std::unique_lock lk2(currentLast->get()->localLock);
            swap(lk1, lk2);
        }
        currentLast->get()->set(std::forward<U>(data));
    }
public:
    LinkedList() : root(std::make_unique<node>()) {};

    template<typename U>
    auto push_back(U&& data) -> std::enable_if_t<std::is_same_v<strip_t<U>, T>> 
    {
        using std::swap;
        auto n = &root;
        // n is a unique_ptr<node> *
        std::shared_lock lk1(n->get()->localLock);
        while (!n->get()->empty) {
            n = &n->get()->next;
            std::shared_lock lk2(n->get()->localLock);
            swap(lk1, lk2);

            // hand over hand locking
            // lk1 and lk2 are swapped, lk1 then goes out of scope
            // releasing the lock
        }

        lk1.unlock();
        // another thread can take an exclusive lock and make a change here
        // we must finish up last push on a writter lock
        // we cannot upgrade a shared_lock to unique_lock
        // bc if two threads try to do that at the same time they'll deadlock
        unique_push(n, std::forward<U>(data));
    }

    bool find(const T& e) const {
        using std::swap;
        auto n = &root;
        std::shared_lock lk1(n->get()->localLock);
        while (!n->get()->empty) {
            if (n->get()->data == e) return true;

            n = &n->get()->next;
            std::shared_lock lk2(n->get()->localLock);
            swap(lk1, lk2);

        }

        return false;
    }
};
```

Here's the idea behind hand-over hand locking: We take the first lock, and use that lock to secure the next lock.
Once the next lock is secure we release the first lock and continue.
This ensures the order in which we acquire and release the locks is the same among all threads.
We can use `swap()` to put the first lock in the new lock's variable (which is about to be destroyed and released the first lock),
and put the new lock into the first lock to keep the new lock around.

We cannot have hand-over-hand locking occurring in two directions simultaneously. This will deadlock because one direction will have
the opposite locking order as the other.

We use reader-writer locks because in this simple linked list, we must do a lot of traversing for both finding and inserting values.
Since we can have multiple threads traversing the list at once, we'll make those use the reader lock.
When a thread is ready to update the last value, it unlocks the reader lock and takes the writer lock. However,
at this point, another thread may have already gotten the writer lock, modified the last value, then released the lock.
So once we have the writer lock, we must ensure that the last node is still the last node, and if not, traverse to the last node.
During this traversal process we want to hold writer locks so that the thread can't keep failing to push a value onto the end of the list.

In order to make the code simpler, we guarantee that the root node is never null by storing an empty flag in each node.
This way, we don't need a separate lock for the root node pointer itself.
Sometimes you may find inserting dummy nodes in between nodes with actual data may also help for similar reasons.

Now what about delete? Well, to make things easier we'll make the root node to always be empty. This allows us not have a separate
root mutex outside the node. Since we'll always have a root node, we don't really need to dynamically allocate it with a pointer.
We'll also change to that a `nullptr` `next` indicates that the node is the last node. We won't need an empty boolean anymore since
the root node will always be empty.

Here's our modified linked list:

```C++
class LinkedList
{
    struct node {
        std::shared_mutex lock;
        T data;

        std::unique_ptr<node> next;
        ///Invariant: next is null iff node is terminal

        template<typename U>
        node(U&& data) : data(std::forward<T>(data)), next(nullptr) {};
    };

    node root;
private:
    template<typename U&&>
    void unique_push(node * currentLast, U&& data) {
        using std::swap;
        std::unique_lock lk1(currentLast->lock);
        while (currentLast->next) {
            currentLast = currentLast->next.get();
            std::unique_lock lk2(currentLast->lock);
            swap(lk1, lk2);
        }
        currentLast->data = std::forward<U>(data);
    }
public:
    LinkedList() : root(std::make_unique<node>()) {};

    template<typename U>
    auto push_back(U&& data) -> std::enable_if_t<std::is_same_v<strip_t<U>, T>> 
    {
        using std::swap;
        auto n = &root;
        // n is a node *
        std::shared_lock lk1(n->lock);
        while (n->next) {
            n = n->next.get();
            std::shared_lock lk2(n->lock);
            swap(lk1, lk2);
        }

        lk1.unlock();
        // another thread can take an exclusive lock and make a change here
        // we must finish up last push on a writter lock
        // we cannot upgrade a shared_lock to unique_lock
        // bc if two threads try to do that at the same time they'll deadlock
        unique_push(n, std::forward<U>(data));
    }

    bool find(const T& e) const {
        using std::swap;
        auto n = &root;
        std::shared_lock lk1(n->lock);
        while (n->next) {
            if (n->data == e) return true;

            n = n->next.get();
            std::shared_lock lk2(n->lock);
            swap(lk1, lk2);

        }

        return false;
    }

    void remove(const T& e) {
        auto n = &root;
        std::unique_lock lk(n->lock);
        while (n->next) {
            const auto next = n->next.get();
            std::unique_lock nextLock(next->lock);
            if (next->data == e) {
                n->next = std::move(next->next);
            } else {
                lk.unlock();
                n = next;
                lk = std::move(nextLock);
            }
        }
    }
};
```

Since the root is always empty and never null, during deletion we can always be checking if we should delete the next node and never have to worry about
keeping track of a previous node's lock. Notice also deletion always uses a `unique_lock`.
This ensures that a thread that started behind the deletion thread can never traverse past it.
This isn't particularly important, but it does prevent querying `find()` from returning `true` for a node
that's about to be deleted on another thread. However, if the querying thread started ahead of the deletion thread and stayed ahead, this
may still occur.

## Thread Queue

```C++
template<typename T>
class ThreadQueue {
    struct node {
        std::unique_ptr<T> data; ///< Terminal iff data is null
        std::unique_ptr<node> next;

        node(std::unique_ptr<T> && data) : data(std::move(data)), next(std::make_unique<node>()) {}

        node() : data(nullptr), next(nullptr) {}
    };

    std::unique_ptr<node> head;
    node * tail; ///< Tail always points to a terminal node
    std::mutex headMu, tailMu;
    // Invariant: always lock head before tail if locking both

    std::condition_variable dataCond;

    node* getTail() {
        std::scoped_lock lk(tailMu);
        return tail;
    }
public:
    ThreadQueue() : head(std::make_unique<node>()) {
        tail = head.get();
    }


    template<typename U>
    void push_front(U&& data) {
        auto new_node = std::make_unique<node>(
            std::make_unique<T>(std::forward<U>(data)));
        // allocation doesn't need a lock
        // perform it before getting lock

        // new_node can't be const so we can move it
        {
            std::scoped_lock lk(headMu);
            new_node->next = std::move(head);
            head = std::move(new_node);
        }
        dataCond.notify_one();

        // notifty condition variable after lock unlocks

    }

    template<typename U>
    void push_back(U&& data) {
        auto new_data = std::make_unique<T>(std::forward<U>(data));
        auto next_node = std::make_unique<node>();
        {
            std::scoped_lock lk(tailMu);
            tail->data = std::move(new_data);
            tail->next = std::move(next_node);
            tail = tail->next.get();

        }
        dataCond.notify_one();

    }

    std::optional<T> pop_front() {
        std::scoped_lock lk(headMu);
        if (head.get() != getTail()) {
            const std::optional<T> out = { std::move(*head->data.release()) };
            head = std::move(head->next);
            // new head is not nullptr because we know head != tail
            return out;
        }
        // due to invariant, if head is tail, then queue is empty
        return {};
    }

    T wait_pop() {
        std::unique_lock head_lk(headMu);
        dataCond.wait(head_lk, [this]() { return head.get() != getTail(); });
        // will not wait if head.get() != tail

        // head != tail here
        auto data = head->data.release();
        head = std::move(head->next);
        return *data;
    }

    bool empty() const {
        std::scoped_lock lk(headMu);
        return head.get() == getTail();
    }
};
```

For operations requiring both head and tail locks, we must enforce an order in which to lock them. I chose to always lock the head before locking the tail.
We create a `getTail` helper to not forget to lock the tail when doing tail-head comparisons to check emptiness.
This class has an invariant that tail always points to a terminal (next pointer is null) node. Therefore, when `head.get() == tail`,
the queue is empty. Also remember: things we want to move cannot be declared `const`. Move construction/assignment mutates the variable that we move from.
Therefore, many of these variables are declared `auto` even though it may appear that they should be `const auto`.

Also notice that `push_front()` doesn't modify the existing head node. This allows two threads to push to the front and back of the queue simultaneously.
`push_front()` adds a new head node, keeping the existing one as is, and `push_back()` modifies the current terminal node.
So even when `head.get() == tail`, the two operations can occur at the same time.