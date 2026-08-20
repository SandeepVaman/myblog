How BoltDB Works: A High-Level Tour

This is Part 1 of Nuts and Bolt, a series where we break down the BoltDB key-value store piece by piece and, in the end, rebuild it in Rust.

---

Starting with a drawer

I’ve never been great at putting things back where they belong. Tools end up in the nearest drawer, cables get tossed in a box 'for now,' and small parts are scattered around the house. Everything seems fine until I actually need the 10mm socket, and then I’m turning the garage upside down while the job waits.

Every few months, when I clean out the attic, I rediscover things I forgot I owned. It’s a small, recurring reminder that how you store something decides how fast you can find it later.

Databases depend on this same idea. How quickly you can write and read data depends on how it’s organized on disk. Storage layout, indexes, query processing, and transactions are all different ways of answering one question: where do we put the bytes so we can find them again, correctly and quickly?

For a long time, I wondered how databases actually work inside, but big codebases always put me off. Then I came across Bolt and realized I’d been using it at work without even knowing. It’s the key-value store quietly running under etcd (which supports Kubernetes) and Consul by HashiCorp. I started reading it to learn how B+trees are stored on disk, and ended up learning about MVCC, transactions, and crash recovery as well.

This series is my way of sharing what I learned. This first post is the map: it covers the whole architecture at a high level and shows how the pieces fit together. Later posts will dive into each part. So let’s start at the top.

---

What is a key-value store, really?

At its simplest, a key-value store is a persistent version of a data structure you already know: a map / dictionary / hash table.

db.put("user:42", "Sandeep")
db.get("user:42")   // → "Sandeep"

You hand it a key and a value, both just bags of bytes. Later you hand it the key and get the value back. That’s the whole contract.

The interesting part isn’t the API — it’s the promises underneath it:

* Persistence — the data survives a process restart or a power cut.
* Ordering — you can walk keys in sorted order and do range scans (everything from "user:1" to "user:99").
* Consistency — a reader never sees a half-written update.
* Concurrency — many readers and writers, without stepping on each other.

BoltDB provides all of these features as an embedded store. There’s no server, no network, and no separate process. It’s just a library you import. Your application is the database. It uses a single .db file on disk, and only one process uses it at a time.

That rule—one process, one file—is not a weakness. It’s the design choice that makes everything else simple and fast. Let’s see how.

---

The big picture

Here’s BoltDB in one diagram. Don’t worry about the details yet; we’ll walk through each layer soon.

      Your code
         │  db.Update / db.View
         ▼
   ┌───────────────┐
   │  Transaction  │   a consistent snapshot of the whole DB
   └───────────────┘
         │  organizes data into…
         ▼
   ┌───────────────┐
   │    Buckets    │   named namespaces, each its own B+tree
   └───────────────┘
         │  which are…
         ▼
   ┌───────────────┐
   │    B+trees    │   sorted, on-disk trees of keys → values
   └───────────────┘
         │  whose tree nodes are stored as…
         ▼
   ┌───────────────┐
   │     Pages     │   fixed-size blocks, the atom of storage
   └───────────────┘
         │  living inside…
         ▼
   ┌───────────────┐
   │ One .db file  │   memory-mapped into your process
   └───────────────┘

From bottom to top: a single file is divided into fixed-size pages. Pages form B+trees. Each bucket is a B+tree, and every read or write happens inside a transaction that sees a consistent snapshot. Now let’s build that stack from the ground up.

---

Layer 1: One file, carved into pages

BoltDB stores everything in a single file. When it opens that file, it doesn’t use the usual read() and write() methods. Instead, it memory-maps the file.

Memory-mapping (mmap) is a neat trick: you ask the operating system to make a file appear as a region of your process’s memory. Reading byte offset 8192 of the file is now just reading memory address base + 8192. The OS loads pages from disk as you access them and caches them for you. There’s no parsing or copying; the bytes on disk and in memory are the same.

That file is divided into equal-sized blocks called pages, typically 4KB each (matching the OS’s own page size). Every page has a small header saying what it is:

┌────────────────────────────────────────────┐
│ Page header:  id │ flags │ count │ overflow │
├────────────────────────────────────────────┤
│                                            │
│              page contents…                │
│                                            │
└────────────────────────────────────────────┘

There are four kinds of page, distinguished by the flags field:

Column 1	Column 2
Page type	What it holds
meta	The database’s root pointer and bookkeeping (see Layer 5)
freelist	A list of pages that are free to be reused
branch	Interior B+tree nodes — keys that route you to children
leaf	The actual key-value pairs (and pointers to sub-buckets)


A brand-new BoltDB file is tiny: just four pages.

Page 0: meta      ┐  two copies, for safety
Page 1: meta      ┘  (see Layer 5)
Page 2: freelist     "no free pages yet"
Page 3: leaf         the empty root bucket

The page is the basic unit of BoltDB. Everything above this layer is really just pages linked together, and every read or write eventually comes down to 'which page, at which offset.'

The zero-copy trick: because the file is memory-mapped, BoltDB reads a page by casting a pointer straight into the mapped memory. There’s no deserialization step at all. The struct layout in memory matches the on-disk format. This 'struct as a lens over raw bytes' idea is so important that it will get its own post later in the series.

---

Layer 2: The B+tree — keeping keys sorted

Pages are how BoltDB stores bytes, but the B+tree is what keeps keys organized so they can be found quickly.

Think back to the drawer example. If I toss screws in randomly, finding an M4×12 means checking every screw. But if they’re sorted into labeled compartments, I can go straight to the right one. A B+tree takes this idea to the next level and is designed to work on disk.

A B+tree is a sorted tree with two kinds of nodes:

* Branch nodes (the interior nodes) don’t hold values. Instead, they act as signposts: 'keys less than cat are down this way; keys from cat to mouse are down that way.' Their only job is to guide you.
* Leaf nodes (the bottom row) hold the actual key-value pairs, kept in sorted order.

                    ┌───────────────────────────┐
                    │  Branch node              │
                    │  [ <cat ] [ cat…mouse ]…  │
                    └───────────────────────────┘
                        │              │
          ┌─────────────┘              └─────────────┐
          ▼                                          ▼
  ┌─────────────────┐                      ┌──────────────────────┐
  │ Leaf            │                      │ Leaf                 │
  │ apple → red     │                      │ cat   → orange       │
  │ banana → yellow │                      │ dog   → brown        │
  └─────────────────┘                      └──────────────────────┘

To look up a key, you start at the top and follow the signposts down until you reach the leaf that should contain it. This only takes a few steps, even with millions of keys. Since leaves are sorted, range scans and ordered iteration are easy: you find your starting key and move sideways.

Two properties make the B+tree a great fit for a disk-backed store:

1. It’s shallow and wide. Each node is a whole page holding many keys, so the tree stays only a few levels deep. Few levels means few page reads to reach any key.
2. It maps neatly onto pages. Each tree node is one page. Branch pages store branch nodes, and leaf pages store leaf nodes. The abstract tree matches the physical file.

(How nodes split when they get too full, and merge when they get too empty, is a story for the deep-dive on node.go.)

---

Layer 3: Buckets — namespaces on top of the tree

You rarely want all your keys in one flat space. You want users separate from sessions separate from config. In BoltDB, those namespaces are called buckets.

Here’s the clever part: a bucket is just its own B+tree. A value in a bucket can be either a plain value or a pointer to another bucket’s tree. This allows buckets to nest:

Root bucket
├── "users"      (a bucket)  ──► its own B+tree of user records
│     ├── "user:1" → {...}
│     └── "user:2" → {...}
├── "sessions"   (a bucket)  ──► its own B+tree
└── "config"     → "value"   (a plain key-value pair)

Every database has one hidden root bucket at the very top. Top-level buckets you create are entries in the root bucket’s tree. Buckets inside them are entries in their trees, and so on, all the way down. It’s trees within trees, with the same structure reused at every level. That’s what makes the design feel so tidy. A single meta pointer (Layer 5) points to the root bucket, and from there you can reach every key in the database.

---

Layer 4: Pages vs. Nodes — disk shape and memory shape

Here’s something that confused me at first, but makes things much clearer once you understand it.

BoltDB has two representations of the same tree node, and it uses them at different times:

* A page is the on-disk shape. It’s packed, read-only, and lives inside the memory-mapped file. Reading it is zero-copy; you just point at it.
* A node is the in-memory shape. It’s a regular heap-allocated struct with a list of entries that can grow. BoltDB creates a node when it needs to modify something.

   Reading                              Writing
   ────────                             ───────
   ┌──────────┐                         ┌──────────┐   materialize   ┌──────────┐
   │   Page   │  ← just read directly   │   Page   │ ═════════════►  │   Node   │
   │ (mmap,   │     from the mmap        │ (on disk)│                 │ (heap,   │
   │  read-   │                         └──────────┘   ◄═════════════ │ writable)│
   │  only)   │                              spill (write back)       └──────────┘
   └──────────┘

The rule is:

* Reads touch pages directly in the mmap. Nothing is copied, nothing is allocated. This is why reads are so fast.
* When you write, BoltDB turns the relevant page into a node, which is a mutable copy on the heap. You edit the node, and at commit time, BoltDB writes the node back out as new pages.

Notice the word 'fresh.' A write never overwrites the page it read from. That single decision is the key to the next layer.

---

Layer 5: Transactions, snapshots, and copy-on-write

Everything you do in BoltDB happens inside a transaction. There are two types, and the difference between them is central to the whole design:

* Read-only transactions — you can have many at once.
* Read-write transactions — there is only ever one at a time.

// Read-only: many can run concurrently
db.View(func(tx *bolt.Tx) error {
    b := tx.Bucket([]byte("users"))
    v := b.Get([]byte("user:42"))
    return nil
})

// Read-write: serialized, one at a time
db.Update(func(tx *bolt.Tx) error {
    b, _ := tx.CreateBucketIfNotExists([]byte("users"))
    return b.Put([]byte("user:42"), []byte("Sandeep"))
})

How can many readers work at the same time while a writer is changing the tree, without anyone seeing a half-finished update or having reads blocked by locks? The answer is copy-on-write (COW), and it’s beautifully simple.

Remember from Layer 4: a writer never changes existing pages. When it needs to update a leaf, it writes a new leaf page somewhere else. Now that the leaf is in a new place, its parent branch must point to the new spot, so the parent is also copied to a new page. This process continues all the way up to the root:

Before commit                    After a write (copy-on-write)

     root  ──► B ──► leaf             root'         ← new root page
                                     ╱     ╲
                                    B'      B        ← B' is new, B untouched
                                   ╱  ╲
                              leaf'    leaf          ← leaf' is new

The original pages (root, B, leaf) are left completely untouched. Any reader that started before this write keeps reading the old tree through the old root, seeing a perfectly consistent snapshot frozen at the moment its transaction began. Meanwhile, the writer builds a whole new version of the tree out of new pages.

This is MVCC, or Multi-Version Concurrency Control. Multiple versions of the tree exist at the same time. Readers don’t block writers, writers don’t block readers, and every transaction sees a stable, consistent view of the database. There are no read locks and no torn reads.

So what actually makes the writer’s new version “official”? That’s where the meta page comes in.

The meta page: one atomic switch

The very first pages of the file (page 0 and page 1) are meta pages. A meta page is small but important: it holds the pointer to the current root bucket, the pointer to the freelist, the latest transaction ID, and a checksum.

The meta page is the single source of truth. Whatever tree the current meta page points to is the database. Everything else is either the live tree or old data waiting to be reclaimed.

So a commit works like this:

1. Write all the new (copied) pages to free spots in the file.
2. Write the updated freelist.
3. fsync — force the OS to flush all of that to disk, so it’s durably stored.
4. Now write a new meta page whose root pointer points at the new tree, with an incremented transaction ID and a fresh checksum.
5. fsync again.

Step 4 is the key moment. Until the meta page is written, the database still officially points to the old tree, and the new pages sit there, invisible. As soon as the new meta page is written to disk, the database instantly switches to the new version. Never a state where the database is o half-updated.

Why two meta pages? For safety. BoltDB switches between them: even transactions write to page 0, odd ones write to page 1. If the machine loses power while writing a meta page and it gets corrupted, the other meta page still has the last good version. When opening the file, BoltDB reads both, checks their checksums, and picks the valid one with the highest transaction ID. That’s the whole crash-recovery story: there’s no write-ahead log, no replay, and no journal. Either the new meta page was written completely (so you get the new version) or it wasn’t (so you get the previous version). You can never end up in between.

This is what people mean when they say BoltDB is ACID. Transactions are Atomic (the meta-page flip), Consistent (COW builds a complete new tree), Isolated (MVCC snapshots), and Durable (fsync before the flip).

---

Layer 6: The freelist — recycling pages

There’s one loose end: if every write creates new pages and leaves the old ones behind, won’t the file keep growing forever?

That’s the job of the freelist. When a transaction commits, the old pages it replaced aren’t lost — they’re recorded as free, available for future writes to reuse. The next write that needs a page grabs one from the freelist instead of growing the file.

But there’s a subtle detail here. Remember those long-running read transactions still looking at the old tree? We can’t reuse a page while they might still need it. So a freed page isn’t given out right away; it becomes pending until every reader that could still be using it has finished. Only when no live transaction needs the old version does the page become truly free for reuse.

page replaced by a write
        │
        ▼
   ┌──────────┐   all older readers finished   ┌──────────┐
   │ pending  │ ─────────────────────────────► │   free   │ ──► reused
   └──────────┘                                └──────────┘

So the freelist is what connects MVCC to the physical file. It acts like an accountant, keeping track of exactly when an old version of the tree is safe to overwrite. This is also why a very long-lived read transaction can make a Bolt file grow: it holds old pages hostage, keeping them out of the free pool.

---

Putting it all together: a day in the life of a write

Let’s follow db.Update as it sets users/user:42 = "Sandeep" from top to bottom, connecting all six layers:

1. Begin a read-write transaction. It grabs the current meta page, which is a frozen snapshot of the whole tree. (Meanwhile, any number of readers can hold their own snapshots.)
2. Navigate to the bucket. Start from the root bucket, walk the B+tree to find the users bucket, then walk its B+tree down to the leaf page where user:42 belongs. All of this uses zero-copy reads straight from the mmap.
3. Materialize. Copy that leaf page into an in-memory node so we can modify it.
4. Modify. Insert user:42 → "Sandeep" into the node, keeping the entries sorted.
5. Commit — spill. Write the modified node out as a new leaf page. Because the leaf moved, copy-on-write its parent branch to a new page, and so on up to a new root. Grab the needed pages from the freelist (or grow the file if it’s empty).
6. Commit — durability. fsync all those new pages. Then write a new meta page pointing at the new root, and fsync again. The database has now atomically flipped to the new version.
7. Recycle. The pages we replaced go onto the freelist as pending, to become free once the readers still viewing the old version have finished.

A read (db.View → Get) is just steps 1 and 2, followed by reading the value. There’s no copying, no allocation, and no locks. That’s exactly why BoltDB reads are so fast.

---

Why the design is so satisfying

Take a step back and notice how few ideas are actually involved, and how much each one does:

* Pages provide a single, uniform unit of storage. Meta, freelist, branches, and leaves are all the same size and are addressed the same way.
* The B+tree is one structure, reused for every bucket and nested bucket. It provides sorted, fast lookups and easy range scans.
* Copy-on-write changes 'concurrent readers and a writer' from a locking nightmare into a non-problem: just don’t overwrite anything.
* Two meta pages make 'commit a transaction atomically' and 'recover from a crash' use the same simple mechanism: flip the pointer and pick the valid one when opening the file.
* The freelist connects everything back to a finite file, releasing old versions exactly when they’re safe to reuse, and not a moment before.

There’s no write-ahead log, no background compaction thread, and no separate recovery process. The whole thing is about 10,000 lines of Go. That’s small enough to read from start to finish, which is exactly what makes it such a good teacher. Every concept you’d find in a 'real' database—MVCC, ACID, B+trees, crash recovery—is here, in a form small enough to understand fully.

---

Where this series is going

This was the overview. In the coming posts, we’ll explore each part in detail:

* The page — the on-disk format, and the “struct as a lens over raw bytes” trick that makes reads zero-copy.
* The node & the B+tree — how modifications happen in memory, and how nodes split and merge to keep the tree balanced.
* The bucket — nested namespaces, and how a value can secretly be another tree.
* The cursor — how iteration and range scans walk the tree.
* Transactions & MVCC — the commit protocol and crash recovery in full detail.
* The freelist — page reclamation and the subtlety of pending pages.

And here’s why I went this deep in the first place: at the end of the series, I’ll open-source my Rust port of BoltDB. Re-implementing it in a language with a very different memory model is the ultimate test of whether you really understand the design. Go relies on a garbage collector and unsafe.Pointer casts, while Rust makes you handle ownership, lifetimes, and zero-copy layout directly. Each deep-dive post will compare the original Go code with how I tackled the same problem in Rust.

If you’ve ever wondered how databases actually work under the hood, like I did when I stared at MongoDB in production and wished I understood what it was doing, BoltDB is one of the best places to start. It’s small, it’s real, and every idea in it appears again in the bigger systems.

Next up: the page — where the bytes actually live.

— Sandeep