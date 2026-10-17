# Order Book Performance Report

Himanshu Chauhan (24BCS10071), Harshit Sen (24BCS10003)

Machine: i5-13420H laptop, WSL2 Ubuntu 26.04, g++ 15.2 with `-O2`, perf 7.0. The timing numbers are the medians of 5 runs from `./bench --impl X` (raw output in `results/bench_*.txt`). The perf numbers come from `perf stat -r 5` on `./bench --impl X --runs 1`.

## Results table

| Metric | Implementation A | Implementation B | Implementation C |
|--------|------------------|------------------|------------------|
| Total time (ms) | 276.65 | 367.64 | 93.80 |
| ns / add | 206.8 | 179.2 | 53.7 |
| ns / cancel | 519.3 | 1080.2 | 143.9 |
| ns / BBO | 19.5 | 18.9 | 19.1 |
| IPC | TODO (lab machine) | TODO (lab machine) | TODO (lab machine) |
| Branch misses (%) | TODO (lab machine) | TODO (lab machine) | TODO (lab machine) |
| L1-dcache-load-misses | TODO (lab machine) | TODO (lab machine) | TODO (lab machine) |
| Cache misses | TODO (lab machine) | TODO (lab machine) | TODO (lab machine) |
| Page faults | 19,305 | 21,713 | 22,164 |
| task-clock, whole program (ms) | 369.32 | 463.04 | 139.79 |

The four TODO rows need hardware counters, and WSL2 reports them as `<not supported>`. dTLB-load-misses is missing for the same reason. The numbers move around a fair bit between runs on this laptop (perf shows up to 8% spread for A), so we only trust differences that are much bigger than that.

perf record, top functions (self time):

| A | B | C |
|---|---|---|
| BookA::cancel_order 30.4% | BookB::cancel_order 38.7% | __vdso_clock_gettime 34.7% |
| BookA::add_order 27.7% | BookB::add_to 12.1% | run_once<BookC> 25.9% |
| unordered_map operator[] 9.1% | run_once<BookB> 11.7% | make_ops 13.6% |
| __vdso_clock_gettime 8.0% | __vdso_clock_gettime 7.9% | BookC::add_order 8.9% |
| make_ops 4.6% | _Rb_tree_rebalance_for_erase 7.2% | mt19937 ~8% |

For C, most of BookC's code got inlined into `run_once`, so that 25.9% is mostly the book itself.

## 11.1 Which one is fastest?

C is fastest for add and cancel. It is about 4x faster than A on add and 3.6x faster on cancel. Over the whole run it takes 94 ms against 277 ms for A and 368 ms for B.

The reason is that C never searches for anything. An add writes `orders[id]` at the end of a vector and does `levels[tick] += qty`. A cancel reads `orders[id]` and subtracts from `levels[tick]`. A has to hash the id, walk a bucket list and then go down a red-black tree holding up to 5000 price levels. B goes down a tree of about 500,000 orders on every cancel.

For get_bbo, all three come out at 19 ns and we can't tell them apart. All three just read two values they already have (begin() for A, back() for B, two ints for C). The 19 ns is mostly the two `steady_clock::now()` calls around it. perf record agrees: in C, 35% of all samples are in `__vdso_clock_gettime`, more than the book code. To really measure get_bbo we would have to time a batch of calls together instead of one at a time.

One thing that surprised us: B's add (179 ns) is a bit faster than A's (207 ns), even though B uses a tree for orders. Our ids go up by one each time, so a new id always goes to the far right of the tree, and that path is already in cache from the last insert. A hashes the id to a random bucket instead, and the hash table also rehashes several times as it grows to 500k entries. Both books do two lookups per add (a check that the id is new, then `operator[]`). In A the second one shows up as the 9% for `_Map_base::operator[]`.

B's cancel is the worst at 1080 ns. A random id means a walk down about 19 levels of the tree, and the lower levels are almost never in cache. Then erasing a node rebalances the tree (`_Rb_tree_rebalance_for_erase` is 7% on its own).

## 11.2 Branch prediction

We couldn't measure branch-misses on WSL, so this part is what we expect from reading the code. We still need to check it on the lab machine.

We think B will have the most branch misses. Every cancel goes down the order tree and every add or cancel also runs our binary search over the level vector. At each step of both, whether we go left or right depends on a random id or a random price, so the CPU is basically guessing and gets it wrong about half the time. That is around 19 + 13 hard branches per cancel.

A has the same problem in its level tree (about 12 levels), but its order lookup is a hash, which has only a couple of branches per lookup. C has very few data dependent branches. The `if (tick > best_bid)` check is almost always false once the book fills up, and the scan loop in cancel almost never runs because with 500k orders spread over 10,000 ticks a level is very rarely emptied.

## 11.3 Cache behaviour

Again we don't have cache-miss counts yet, but the layouts are quite different.

A and B both allocate a separate heap node for every order (an `unordered_map` node in A, a tree node in B), so about 500k small objects are live at the end. These end up spread over roughly 25 to 35 MB of heap, and finding one means following pointers. Each pointer hop can be a cache miss, and with random cancel ids most of them are. B is hit harder because a tree lookup is about 19 hops and a hash lookup is about 2.

The level storage in A is also a tree (5000 nodes per side), but that is small enough to stay mostly in L2. B's level vectors are contiguous and only 80 KB per side, so that part of B is cache friendly. It just doesn't matter much next to the order tree.

C's level array is one contiguous block of 120 KB that stays in L2, and the depth walk reads it in order. The order vector is about 17 MB, so a cancel of a random id is still a cache miss, but it is exactly one miss with no pointers to follow. We think this is most of the 144 ns cancel time in C.

We expect C to show the fewest cache misses and B the most once we get the lab numbers.

## 11.4 Page faults

C has the most page faults (22,164), then B (21,713), then A (19,305). Part of this is the same for everyone: the workload vector alone is 1,000,000 x 40 bytes = 40 MB, which is around 9,800 pages of 4 KB.

C being the highest was not what we expected, since it is the fastest. There are two reasons for it. First, the order vector is indexed by id, so it keeps a slot for every order ever added (about 700k), even the ones that were cancelled long ago. Cancelled orders never give their memory back. Second, the vector grows by doubling. Each time it grows, a new bigger block is allocated and touched, the old one is copied over and freed, so the total memory touched is about twice the final size. These blocks are big enough that glibc gets them straight from mmap, so every new block means fresh page faults.

A has the fewest because when an order is cancelled its hash node is freed, and the next add can reuse that memory without a new page. B also reuses freed nodes, but a tree node is bigger than a hash node, so it needs more pages overall.

So page faults here are mostly about how much memory the program touches overall, not about how fast each operation is.

## 11.5 Production choice

We would pick C, but not as it is now.

On latency it is clearly ahead: about 54 ns to add and 144 ns to cancel, against 200 to 1000 ns for the other two. It is also more predictable. Every add and cancel does about the same small amount of work. A has occasional rehashes, and A and B call malloc and free on every add and cancel, and malloc time varies. C's one bad case is a cancel that empties the best level, where it scans until it finds the next non-empty tick. With a band of 5000 ticks that scan has a hard upper limit, and in a busy book it is short.

Memory is where C is weakest right now. The level array is fixed and small (120 KB), which is fine. The order vector is the problem, because it grows with every id ever seen. In a real system ids come from the exchange and are not small and dense like ours, so we would need to map them to slot numbers ourselves and reuse slots after a cancel (a free list). With that change the memory would be fixed and could be allocated once at startup.

On maintainability A is the nicest. It is short, uses containers everyone knows, and is hard to get wrong. C has more ways to break. If best_bid isn't updated correctly, every BBO after that is wrong, which is exactly why we built the --verify mode. We would still go with C plus the slot mapping, keep A around as the reference in tests, and keep running the same comparison against it.

## 11.6 What we would try next

- Time get_bbo in batches of 1000 calls so the clock isn't most of what we measure.
- Pre-size C's order vector (or use a fixed pool with a free list) so it never grows during the run. This should remove most of C's page faults.
- In C, keep a bitmap of non-empty ticks so finding the next best level is a few word scans instead of a loop over every tick.
- In A, call `reserve()` on the unordered_map up front and replace the count + operator[] pair with a single `emplace`.
- Take out `double` from the API entirely and pass ticks as ints.
