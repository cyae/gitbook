The common thread linking most of these gotchas is scalability. They're things that won't affect you while your database is small. But if one day you want your database not to be small, it pays to think about them in advance.

It's an easy trap to fall into if you're not aware of it, because all your queries in local development will typically run perfectly. And probably in production too at first, you'll have no issues. But as your application grows, the volume of data and complexity of your queries both increase. It's only then that you'll start to encounter problems, a textbook _"but it worked on my machine"_ scenario.

## 1. Keep the default value for `work_mem`
This setting governs how much memory is available to each query operation before it must start writing data to temporary files on disk. 
When `work_mem` becomes over-utilised, you'll see latency spikes as data is paged in and out, causing hash table and sorting operations to run much slower.
A good value depends on multiple factors: 
- the size of your Postgres instance, 
- the frequency and complexity of your queries, 
- the number of concurrent connections.
In prod, wo often starts with:
$work\_mem = (\$YOUR\_INSTANCE\_MEMORY * 0.8 - shared\_buffers) / \$YOUR\_ACTIVE\_CONNECTION_COUNT$
```

```