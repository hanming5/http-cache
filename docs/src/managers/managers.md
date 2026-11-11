# Backend Cache Manager Implementations

The following backend cache manager implementations are provided by this crate:

## [cacache](./cacache.md)

[`cacache`](https://github.com/zkat/cacache-rs) is a high-performance, concurrent, content-addressable disk cache, optimized for async APIs. Provides traditional buffered caching. Now **opt-in** (no longer the default); its upstream is dormant/unmaintained.

## [moka](./moka.md)

[`moka`](https://github.com/moka-rs/moka) is a fast, concurrent cache library inspired by the Caffeine library for Java. Provides in-memory caching with traditional buffering.

## [foyer](./foyer.md)

[`foyer`](https://github.com/foyer-rs/foyer) is a hybrid in-memory + disk cache that provides configurable eviction strategies (w-TinyLFU, S3-FIFO, SIEVE), optional disk storage, request deduplication, and Tokio-native async operations.

## [redb](./redb.md)

[`redb`](https://github.com/cberner/redb) is a simple, portable, pure-Rust embedded key/value database. `RedbManager` provides traditional buffered, persistent caching with **no async runtime dependency** — making it the recommended persistent backend for non-tokio clients such as smol-based `http-cache-ureq` or runtimes like Bevy. Now the **default** cache manager.

## [quick_cache](./quick-cache.md)

[`quick_cache`](https://github.com/arthurprs/quick-cache) is a lightweight and high performance concurrent cache optimized for low cache overhead. Provides traditional buffered caching operations.

## [streaming_cache](./streaming_cache.md)

[`StreamingManager`](https://github.com/06chaynes/http-cache/blob/main/http-cache/src/managers/streaming_cache.rs) uses an embedded [redb](https://github.com/cberner/redb) database for metadata and raw files on disk for response bodies, with [moka](https://github.com/moka-rs/moka) as an in-memory hot cache for metadata. Streams cached responses from disk in 64KB chunks without buffering entire bodies in memory. Ideal for large responses and long-lived caches that must survive restarts.

## Eviction and size limits

`http-cache` does not evict entries on its own. It decides whether a response may be stored and whether a stored response is fresh, stale, or needs revalidation, but how much the cache holds is left to the backend cache manager. A stale entry stays in storage until one of the following happens:

- it is revalidated and replaced with a newer response,
- a request invalidates it, such as a non-`GET`/`HEAD` request to the same URL that gets a `2xx` or `3xx` response, or a key returned by `cache_bust`,
- the backend evicts it.

The `default_ttl` and `max_ttl` fields of `HttpCacheOptions` change how long a response is considered fresh. They do not delete entries or bound the size of the cache.

| Manager | Storage | Size bounded? | How to limit it |
| --- | --- | --- | --- |
| [`MokaManager`](./moka.md) | Memory | Yes | Entry count with `Cache::new(n)`, total bytes with a weigher, and `time_to_live` / `time_to_idle` on the `moka` builder |
| [`QuickManager`](./quick-cache.md) | Memory | Yes | Entry count with `Cache::new(n)` |
| [`FoyerManager`](./foyer.md) | Memory, optional disk | Yes | Memory and disk capacity and the eviction strategy, set on `HybridCacheBuilder` |
| [`RedbManager`](./redb.md) | Disk | No | Remove entries yourself with `delete` or `clear` |
| [`CACacheManager`](./cacache.md) | Disk | No | Remove entries yourself with `delete` or `clear`. Create the manager with `remove_fully: true` so deletes also free the content on disk |
| [`StreamingManager`](./streaming_cache.md) | Disk, with in-memory metadata | Memory only | `capacity` bounds the in-memory metadata cache, not the files on disk. `max_body_size` skips responses larger than the limit. Use `delete` or `clear` to free disk space |

If you need a persistent cache with a size limit, `FoyerManager` with a disk device is the only built-in option. Otherwise, prune a disk-backed cache from your own code, for example by calling `clear` on a schedule.

For example, to cap a `MokaManager` at roughly 64 MiB of cached data instead of a number of entries, weigh each entry by the length of its serialized value:

```rust
use http_cache::MokaManager;
use moka::future::Cache;
use std::sync::Arc;

let cache = Cache::builder()
    .weigher(|_key: &String, value: &Arc<Vec<u8>>| {
        u32::try_from(value.len()).unwrap_or(u32::MAX)
    })
    .max_capacity(64 * 1024 * 1024)
    .build();
let manager = MokaManager::new(cache);
```
