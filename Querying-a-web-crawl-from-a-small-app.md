---
title: "Querying a web crawl from a small app"
description: "Read Common Crawl's Parquet index and hyperlink graph directly from a small container with DuckDB, without copying the data — and without the big join that runs a 256 MB container out of memory."
category: Guides
tags: [hosting, data]
order: 4
checked: 2026-11-09
---

<!-- NAV:START -->
[Home](README.md) › [Guides](Category-Guides.md) · Tags: [hosting](Tag-hosting.md), [data](Tag-data.md)
<!-- NAV:END -->

# Querying a web crawl from a small app

**You can query Common Crawl from an ordinary small app: DuckDB reads the published Parquet and text files over HTTPS, so there is no cluster, no data warehouse and no local copy.** The work is choosing the right files, keeping the query inside the container's memory, and treating a multi-minute scan as a background job rather than a web request.

This page is about a website that searches Common Crawl and then asks who links to a result. The same shape applies to any app that wants web data for building [RAG](Glossary.md#rag) indexes, sampling pages, or analysing links.

## What Common Crawl publishes

**Two separate datasets: a URL index in Parquet, and a hyperlink graph in text files.** They are published on `data.commoncrawl.org` and are free to read.

The **URL index** is "a tabular/columnar index (Parquet format) to the Common Crawl archives", partitioned by two columns, `crawl` and `subset`, where subsets include `warc`, `robotstxt` and `crawldiagnostics` ([Common Crawl, index table](https://data.commoncrawl.org/cc-index/table/cc-main/index.html), checked 2026-11-09). It carries the URL, host and domain columns, fetch time and status, detected MIME type, language fields, and the location of the record inside the WARC file. Common Crawl's announcement of the format says "the index table of a single monthly crawl has about 300 GB" ([Common Crawl blog](https://commoncrawl.org/blog/index-to-warc-files-and-urls-in-columnar-format), checked 2026-11-09). You read a few of those gigabytes, not all of them.

The **hyperlink graph** comes in host-level and domain-level releases. Hostnames are stored "in reverse domain name notation"; all link types are counted, including links to images and scripts; and "the domain graph is built by aggregating the host graph at the pay-level domain (PLD) level" using the public suffix list ([Common Crawl, web graphs](https://commoncrawl.org/web-graphs), checked 2026-11-09). The machine-readable list of releases, newest first, is `https://index.commoncrawl.org/graphinfo.json`.

The graph and each crawl are immutable once published. That matters later: it makes every answer cacheable.

## Read it without copying it

**DuckDB's `httpfs` extension turns an HTTPS URL into a readable file, so a query can open a Parquet part without downloading it first.** The extension provides "a file system that allows reading remote/writing remote files", and for plain HTTP(S) "only file reading is supported" ([DuckDB docs, httpfs](https://duckdb.org/docs/2.0/core_extensions/httpfs/overview.html), checked 2026-11-09). It loads itself on first use.

Two practical details:

- **Pass explicit file paths.** A crawl publishes a manifest of its Parquet parts; unzip it, keep the paths you want, and hand DuckDB the list. Reading the manifest beats guessing at URL patterns, and it is how you find a crawl's files at all.
- **Filter on partition columns.** `WHERE subset = 'warc' AND url_host_registered_domain = $domain` lets Parquet statistics skip most row groups. Grouping by registered domain also means a search for `example.com` includes its subdomains, which is usually what a person means.

In SQL, the shape is:

```sql
SELECT url, fetch_status, content_mime_detected, fetch_time
FROM read_parquet($files, hive_partitioning = true)
WHERE subset = 'warc'
  AND url_host_tld = $tld
  AND url_host_registered_domain = $domain
LIMIT $limit;
```

Bind the file list and filters as parameters rather than pasting them into the string.

## Keep the query inside a small memory limit

**Set `memory_limit` below the container's memory and raise it only when a query needs it.** DuckDB's limit defaults to 80% of the machine's RAM, which on a shared host can be far more than the container is allowed; `SET memory_limit = '256MB'` makes the database spill or fail instead of being killed. `threads` defaults to the number of CPU cores and can be lowered the same way ([DuckDB docs, configuration](https://duckdb.org/docs/2.0/configuration/overview.html), checked 2026-11-09).

The failure to recognise is not "the query is slow" but **an out-of-memory error on a join**. A query that joins a huge lookup table to a filtered result makes DuckDB build a hash table for the whole lookup side, even though the answer needs a handful of rows. In this project, joining the graph's edge table to its full vertex table exhausted a 256 MB limit; the same data fitted once the query stopped doing that.

The fix is a two-step query:

1. **Aggregate first.** Group the large table down to the few identifiers you actually need — for example, the top 200 sites linking to one domain — and keep only ids and counts.
2. **Resolve the small set second.** Look up names for just those ids with `WHERE id IN (...)`, accepting a scan of the name table but holding only a couple of hundred rows in memory.

This "aggregate, then resolve" pattern works whenever the lookup table is bigger than the answer: spend one pass counting, then one bounded lookup, instead of a hash join over everything.

## Make a slow scan a job, not a request

**A remote scan can run for minutes, so start it, return an id, and let the page poll for progress.** Browser and proxy timeouts are measured in seconds; a DuckDB query over the graph is not. Running the query inside the request handler also makes cancelling impossible, because nobody is listening.

A small in-process job manager is enough: one running job at a time, a status of `running`, `done`, `error` or `cancelled`, a stage number, a human message, and elapsed time. The page polls `GET /api/jobs/:id` once a second and draws a progress bar. For cancellation, keep the DuckDB connection and call `interrupt()`; check the job's cancelled flag between stages so a cancel that lands between queries is not lost.

Say what the wait means. "Scanning the graph edge file. This can take several minutes" tells a visitor more than an animated bar with no words.

## Cache finished answers by release

**Because a crawl and a graph release never change, a result keyed by release and query stays correct forever.** Write each finished answer to its own JSON file, keyed by a hash of `crawl | domain | limit` for searches and `release | domain` for link lookups, and write atomically (temporary file, then rename) so a crash cannot leave half a record. On a [host](Glossary.md#host) with a volume, put that directory on the volume so it survives redeploys.

The first query for a domain pays the scan; every repeat returns immediately. A small admin page that lists entries with sizes and can delete them is worth the hour it takes, especially when the volume is small and an oversized cache should be trimmed rather than rediscovered.

## What to watch

- **The data host is a dependency.** Cold queries depend on `data.commoncrawl.org` being reachable and fast; add retries and a timeout, and show a clear error rather than a spinner.
- **Do not count on ordering or totals.** `LIMIT` caps rows, not distinct URLs, and row order is unspecified; a page can appear more than once.
- **Check the domain before querying.** Parse the input as a URL, reject IP addresses and credentials, and require a real public suffix; otherwise a typo becomes a scan.
- **State the graph's granularity.** The graph is per domain, so "who links here" means links into any page on the site, not into the exact URL the visitor clicked.

## Related pages

- [The cloud, for AI apps](Cloud-for-AI-apps.md): containers, volumes, and what the host provides
- [From idea to deployed AI app](Idea-to-deployed-AI-app.md): the deploy path this app follows
- [Why a deploy looks broken in one browser but fine in another](Stale-cache-after-a-deploy.md): the other kind of cache, and why an app cache is not a browser cache

## See also

<!-- SEEALSO:START -->
Glossary terms used on this page: [Container](Glossary.md#container), [Host](Glossary.md#host), [RAG](Glossary.md#rag).
<!-- SEEALSO:END -->

## Sources

All checked on 2026-11-09.

- Common Crawl, [CC Index Table](https://data.commoncrawl.org/cc-index/table/cc-main/index.html): Parquet format, the `crawl` and `subset` partitions, and the column schema
- Common Crawl, [Index to WARC files and URLs in columnar format](https://commoncrawl.org/blog/index-to-warc-files-and-urls-in-columnar-format): the format's announcement and the size of one monthly crawl's table
- Common Crawl, [Web Graphs](https://commoncrawl.org/web-graphs): host- and domain-level releases, reverse-domain notation, and pay-level-domain aggregation
- Common Crawl, [graphinfo.json](https://index.commoncrawl.org/graphinfo.json): the machine-readable release list
- DuckDB, [httpfs extension](https://duckdb.org/docs/2.0/core_extensions/httpfs/overview.html): remote file reading over HTTP(S)
- DuckDB, [Configuration](https://duckdb.org/docs/2.0/configuration/overview.html): `memory_limit` and `threads`

---

First version written with AI assistance, from a session that built a Common Crawl search app in a small container, hit the join's out-of-memory error, and replaced it with an aggregate-then-resolve query, and checked against the [sources](#sources) on 2026-11-09. Found a mistake? [Fix it or open an issue](CONTRIBUTING.md). Text licensed under [CC BY 4.0](LICENSE).
