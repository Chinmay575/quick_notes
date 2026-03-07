# 27 — Data Structures, Algorithms & System Design for Mobile

---

## Q1. Essential data structures for mobile interviews.

**Answer:**

```dart
// Array / List — O(1) access, O(n) insert/delete
List<int> list = [1, 2, 3];
list.add(4);           // O(1) amortized
list.insert(0, 0);     // O(n)
list.removeAt(2);      // O(n)

// HashMap / Map — O(1) average lookup
Map<String, int> map = {'a': 1, 'b': 2};
map['c'] = 3;          // O(1) average

// Set — O(1) lookup, no duplicates
Set<int> set = {1, 2, 3};
set.contains(2);       // O(1)

// Stack (LIFO)
List<int> stack = [];
stack.add(1);           // push
stack.removeLast();     // pop
stack.last;             // peek

// Queue (FIFO)
Queue<int> queue = Queue();
queue.add(1);           // enqueue
queue.removeFirst();    // dequeue
queue.first;            // peek

// LinkedList — O(1) insert/delete at known position
// Tree — hierarchical data (widget tree, DOM)
// Graph — social networks, navigation maps
// Heap/PriorityQueue — top-K problems, scheduling
```

---

## Q2. Common algorithm patterns for mobile interviews.

**Answer:**

### Two Pointers
```dart
// Remove duplicates from sorted array
int removeDuplicates(List<int> nums) {
  if (nums.isEmpty) return 0;
  int slow = 0;
  for (int fast = 1; fast < nums.length; fast++) {
    if (nums[fast] != nums[slow]) {
      slow++;
      nums[slow] = nums[fast];
    }
  }
  return slow + 1;
}
```

### Sliding Window
```dart
// Maximum sum subarray of size k
int maxSumSubarray(List<int> nums, int k) {
  int windowSum = 0, maxSum = 0;
  for (int i = 0; i < nums.length; i++) {
    windowSum += nums[i];
    if (i >= k) windowSum -= nums[i - k];
    if (i >= k - 1) maxSum = max(maxSum, windowSum);
  }
  return maxSum;
}
```

### Binary Search
```dart
int binarySearch(List<int> nums, int target) {
  int left = 0, right = nums.length - 1;
  while (left <= right) {
    int mid = left + (right - left) ~/ 2;
    if (nums[mid] == target) return mid;
    if (nums[mid] < target) left = mid + 1;
    else right = mid - 1;
  }
  return -1;
}
```

### BFS/DFS
```dart
// BFS (level-order traversal, shortest path)
List<List<int>> bfs(TreeNode root) {
  List<List<int>> result = [];
  Queue<TreeNode> queue = Queue()..add(root);
  while (queue.isNotEmpty) {
    List<int> level = [];
    int size = queue.length;
    for (int i = 0; i < size; i++) {
      TreeNode node = queue.removeFirst();
      level.add(node.val);
      if (node.left != null) queue.add(node.left!);
      if (node.right != null) queue.add(node.right!);
    }
    result.add(level);
  }
  return result;
}
```

### Dynamic Programming
```dart
// Fibonacci (bottom-up)
int fib(int n) {
  if (n <= 1) return n;
  int prev = 0, curr = 1;
  for (int i = 2; i <= n; i++) {
    int next = prev + curr;
    prev = curr;
    curr = next;
  }
  return curr;
}
```

---

## Q3. String manipulation questions.

**Answer:**

```dart
// Reverse a string
String reverse(String s) => s.split('').reversed.join('');

// Check palindrome
bool isPalindrome(String s) {
  s = s.toLowerCase().replaceAll(RegExp(r'[^a-z0-9]'), '');
  int left = 0, right = s.length - 1;
  while (left < right) {
    if (s[left] != s[right]) return false;
    left++;
    right--;
  }
  return true;
}

// Anagram check
bool isAnagram(String s, String t) {
  if (s.length != t.length) return false;
  Map<String, int> count = {};
  for (var c in s.split('')) count[c] = (count[c] ?? 0) + 1;
  for (var c in t.split('')) {
    count[c] = (count[c] ?? 0) - 1;
    if (count[c]! < 0) return false;
  }
  return true;
}

// First non-repeating character
int firstUnique(String s) {
  Map<String, int> count = {};
  for (var c in s.split('')) count[c] = (count[c] ?? 0) + 1;
  for (int i = 0; i < s.length; i++) {
    if (count[s[i]] == 1) return i;
  }
  return -1;
}
```

---

## Q4. Sorting algorithms — know the trade-offs.

**Answer:**

| Algorithm | Average | Worst | Space | Stable? | Use Case |
|-----------|---------|-------|-------|---------|----------|
| Quick Sort | O(n log n) | O(n²) | O(log n) | No | General purpose |
| Merge Sort | O(n log n) | O(n log n) | O(n) | Yes | Linked lists, stability needed |
| Heap Sort | O(n log n) | O(n log n) | O(1) | No | Memory constrained |
| Insertion Sort | O(n²) | O(n²) | O(1) | Yes | Small/nearly sorted arrays |
| Tim Sort | O(n log n) | O(n log n) | O(n) | Yes | **Default in Dart, Python, Java** |

```dart
// Dart's built-in sort is Tim Sort (stable, adaptive)
list.sort((a, b) => a.compareTo(b));
list.sort((a, b) => a.name.compareTo(b.name));
```

---

## Q5. System Design: Design a Chat Application (Mobile).

**Answer:**

```
Architecture:
┌─────────┐     ┌──────────┐     ┌──────────┐
│ Flutter  │────▶│ API      │────▶│ Database │
│ App      │◀───▶│ Gateway  │     │ (Users,  │
│          │     │          │     │ Messages)│
│          │◀───▶│ WebSocket│     └──────────┘
│          │     │ Server   │     ┌──────────┐
│          │     │          │────▶│ Redis    │
└─────────┘     └──────────┘     │ (PubSub) │
                                  └──────────┘
```

**Key decisions:**
- **Real-time:** WebSocket for message delivery
- **Offline:** SQLite/Hive for local message storage
- **Sync:** Message queuing for offline messages, delivered when back online
- **Media:** Upload to S3/CDN, store URL in message
- **Push notifications:** FCM/APNs for background delivery
- **Pagination:** Cursor-based for message history
- **Encryption:** End-to-end encryption (Signal Protocol)
- **Status:** Sent → Delivered → Read receipts via WebSocket

---

## Q6. System Design: Design an Image-Heavy Social Feed.

**Answer:**

**Key components:**
1. **Image loading pipeline:**
   - Download → Decode → Resize → Cache (memory + disk)
   - Progressive loading (blur placeholder → thumbnail → full)
   - Prefetch next page while scrolling

2. **Caching strategy:**
   - L1: Memory cache (LRU, ~50MB)
   - L2: Disk cache (LRU, ~200MB)
   - L3: CDN (server-side)

3. **Feed pagination:**
   - Cursor-based (`?after=<last_post_id>`)
   - Prefetch next page at 70% scroll

4. **Performance:**
   - Lazy loading with `ListView.builder`
   - Image resizing (don't load 4K images for thumbnails)
   - `cacheWidth`/`cacheHeight` to reduce memory
   - `RepaintBoundary` for each feed item

```dart
// Implementation sketch
CachedNetworkImage(
  imageUrl: post.imageUrl,
  memCacheWidth: 400,
  placeholder: (_, __) => ShimmerPlaceholder(),
  errorWidget: (_, __, ___) => Icon(Icons.error),
)
```

---

## Q7. System Design: Design an Offline-First Mobile App.

**Answer:**

```
┌──────────────────────────────────────────┐
│                Flutter App                │
│                                          │
│  ┌─────────┐    ┌──────────────────────┐ │
│  │   UI    │───▶│   Repository         │ │
│  │  Layer  │◀───│                      │ │
│  └─────────┘    │  ┌────────┐ ┌──────┐ │ │
│                 │  │ Remote │ │Local │ │ │
│                 │  │  API   │ │  DB  │ │ │
│                 │  └────────┘ └──────┘ │ │
│                 │                      │ │
│                 │  ┌────────────────┐  │ │
│                 │  │ Sync Engine    │  │ │
│                 │  │ (conflict      │  │ │
│                 │  │  resolution)   │  │ │
│                 │  └────────────────┘  │ │
│                 └──────────────────────┘ │
└──────────────────────────────────────────┘
```

**Strategies:**
1. **Cache-first:** Read from cache, update from network in background
2. **Network-first:** Try network, fall back to cache
3. **Stale-while-revalidate:** Return cache immediately, update in background

**Conflict resolution:**
- Last-write-wins (simple but may lose data)
- Server-wins / Client-wins
- Merge (field-level comparison)
- Manual (user decides)

**Sync patterns:**
- Operation queue for offline mutations
- Retry with exponential backoff
- Connectivity listener to trigger sync

---

## Q8. Time and Space Complexity — Quick Reference.

**Answer:**

| Complexity | Name | Example |
|-----------|------|---------|
| O(1) | Constant | Array access, HashMap lookup |
| O(log n) | Logarithmic | Binary search |
| O(n) | Linear | Linear search, single loop |
| O(n log n) | Log-linear | Merge sort, Tim sort |
| O(n²) | Quadratic | Nested loops, bubble sort |
| O(2ⁿ) | Exponential | Recursive fibonacci (naive) |
| O(n!) | Factorial | Permutations |

**Mobile-specific performance:**
- 60 FPS = 16ms per frame budget
- Anything >16ms should be off-loaded (isolate/background thread)
- JSON parsing of large data → use `compute()` or `Isolate.run()`
- Image decoding → let the framework handle it (don't decode on UI thread)

---

## Q9. Implement an LRU Cache.

**Answer:**

LRU (Least Recently Used) Cache evicts the least recently accessed item when capacity is full. Uses HashMap + Doubly Linked List for O(1) get/put.

```dart
class LRUCache<K, V> {
  final int capacity;
  final _map = <K, _Node<K, V>>{};
  _Node<K, V>? _head; // most recently used
  _Node<K, V>? _tail; // least recently used

  LRUCache(this.capacity);

  V? get(K key) {
    final node = _map[key];
    if (node == null) return null;
    _moveToHead(node);
    return node.value;
  }

  void put(K key, V value) {
    if (_map.containsKey(key)) {
      final node = _map[key]!;
      node.value = value;
      _moveToHead(node);
    } else {
      final node = _Node(key, value);
      _map[key] = node;
      _addToHead(node);
      if (_map.length > capacity) {
        final removed = _removeTail();
        _map.remove(removed.key);
      }
    }
  }

  void _addToHead(_Node<K, V> node) {
    node.next = _head;
    node.prev = null;
    if (_head != null) _head!.prev = node;
    _head = node;
    _tail ??= node;
  }

  void _removeNode(_Node<K, V> node) {
    if (node.prev != null) node.prev!.next = node.next;
    else _head = node.next;
    if (node.next != null) node.next!.prev = node.prev;
    else _tail = node.prev;
  }

  void _moveToHead(_Node<K, V> node) {
    _removeNode(node);
    _addToHead(node);
  }

  _Node<K, V> _removeTail() {
    final node = _tail!;
    _removeNode(node);
    return node;
  }
}

class _Node<K, V> {
  K key;
  V value;
  _Node<K, V>? prev;
  _Node<K, V>? next;
  _Node(this.key, this.value);
}

// Usage
final cache = LRUCache<String, String>(3);
cache.put('a', '1');
cache.put('b', '2');
cache.put('c', '3');
cache.get('a');     // returns '1', moves 'a' to front
cache.put('d', '4'); // evicts 'b' (least recently used)
```

**Time complexity:** O(1) for both get and put.

---

## Q10. Implement debounce and throttle.

**Answer:**

```dart
// Debounce: wait for N ms of inactivity before executing
// Use case: search-as-you-type
class Debouncer {
  final Duration delay;
  Timer? _timer;

  Debouncer({this.delay = const Duration(milliseconds: 300)});

  void call(VoidCallback action) {
    _timer?.cancel();
    _timer = Timer(delay, action);
  }

  void dispose() => _timer?.cancel();
}

// Usage
final debouncer = Debouncer();
textController.addListener(() {
  debouncer.call(() => search(textController.text));
});

// Throttle: execute at most once per N ms
// Use case: scroll events, button taps
class Throttler {
  final Duration delay;
  DateTime? _lastRun;

  Throttler({this.delay = const Duration(milliseconds: 300)});

  void call(VoidCallback action) {
    final now = DateTime.now();
    if (_lastRun == null || now.difference(_lastRun!) >= delay) {
      _lastRun = now;
      action();
    }
  }
}
```

---

## Q11. System Design: Design an E-Commerce Mobile App.

**Answer:**

**Key features:** Product catalog, search, cart, checkout, order tracking, reviews.

```
Architecture:
┌─────────────────────────────────────────┐
│              Mobile App                 │
├──────────┬──────────┬───────────────────┤
│ Product  │   Cart   │  Order/Payment    │
│ Module   │  Module  │    Module         │
├──────────┴──────────┴───────────────────┤
│        Local Cache (Hive/Room)          │
│        Image Cache (cached_network_image)│
├─────────────────────────────────────────┤
│          API Gateway / BFF              │
├──────┬───────┬────────┬─────────────────┤
│Product│ Cart │ Order  │ Payment │Search │
│Service│Service│Service│ Service │Service│
├──────┴───────┴────────┴─────────────────┤
│     Database (PostgreSQL + Redis)       │
│     Search (Elasticsearch)              │
│     CDN (images, thumbnails)            │
└─────────────────────────────────────────┘
```

**Key design decisions:**
- **Search**: Elasticsearch with autocomplete, filters, fuzzy matching. Debounce 300ms on client.
- **Product images**: CDN with multiple resolutions. Load placeholder → thumbnail → full.
- **Cart**: Local-first (works offline), sync to server on network.
- **Checkout**: Optimistic locking for inventory. Payment via Stripe/Razorpay.
- **Caching**: Product list cached 5 min, product detail cached 1 hour.
- **Pagination**: Cursor-based for product list, infinite scroll.
- **Real-time**: WebSocket for order status, push notifications for delivery updates.

---

## Q12. System Design: Design a Ride-Sharing App (Mobile Client).

**Answer:**

**Core flows:** Request ride → match driver → real-time tracking → payment → rating.

```
┌─────────────────────────────────┐
│         Rider App               │
├─────┬───────┬───────┬───────────┤
│ Map │ Ride  │Payment│  Rating   │
│ View│Request│Module │  Module   │
├─────┴───────┴───────┴───────────┤
│      Location Service           │
│      WebSocket (real-time)      │
├─────────────────────────────────┤
│         API Gateway             │
├──────┬────────┬─────────────────┤
│Match │Tracking│  Pricing        │
│Engine│Service │  Service        │
└──────┴────────┴─────────────────┘
```

**Key design decisions:**
- **Location**: High-accuracy GPS, 1-second updates during ride, 30-second updates while idle.
- **Real-time tracking**: WebSocket for driver location during ride. Fallback to polling every 5s.
- **Matching**: Server-side — find nearby drivers using geospatial indexing (PostGIS / GeoHash).
- **Map rendering**: Google Maps SDK, draw polyline route, animate driver marker.
- **ETA**: Server calculates using traffic data, updates every 30s.
- **Offline**: Cache recent addresses, saved places. Ride can't be requested offline.
- **Battery**: Reduce GPS frequency when app backgrounded. Use significant location changes.

---

## Q13. Common graph and tree problems for mobile interviews.

**Answer:**

```dart
// Binary Tree Level-Order Traversal (BFS)
List<List<int>> levelOrder(TreeNode? root) {
  if (root == null) return [];
  final result = <List<int>>[];
  final queue = Queue<TreeNode>()..add(root);

  while (queue.isNotEmpty) {
    final level = <int>[];
    for (var i = queue.length; i > 0; i--) {
      final node = queue.removeFirst();
      level.add(node.val);
      if (node.left != null) queue.add(node.left!);
      if (node.right != null) queue.add(node.right!);
    }
    result.add(level);
  }
  return result;
}

// Detect cycle in linked list (Floyd's algorithm)
bool hasCycle(ListNode? head) {
  var slow = head, fast = head;
  while (fast?.next != null) {
    slow = slow!.next;
    fast = fast!.next!.next;
    if (slow == fast) return true;
  }
  return false;
}

// Number of islands (DFS on grid)
int numIslands(List<List<String>> grid) {
  int count = 0;
  for (int r = 0; r < grid.length; r++) {
    for (int c = 0; c < grid[0].length; c++) {
      if (grid[r][c] == '1') {
        count++;
        _dfs(grid, r, c);
      }
    }
  }
  return count;
}

void _dfs(List<List<String>> grid, int r, int c) {
  if (r < 0 || c < 0 || r >= grid.length || c >= grid[0].length || grid[r][c] != '1') return;
  grid[r][c] = '0'; // mark visited
  _dfs(grid, r + 1, c);
  _dfs(grid, r - 1, c);
  _dfs(grid, r, c + 1);
  _dfs(grid, r, c - 1);
}
```
