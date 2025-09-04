You’re on the right trail: nothing shows up in Operate because Operate’s **importers can’t read/write Elasticsearch**, so no new instances make it into the UI. Your three log snippets line up with ES trouble:

* **camunda.log:** “Some ES shards failed… retrying” → Operate importer can’t read zeebe indices.
* **connectors.log:** “Failed to import process elements” → also blocked by ES.
* **elasticsearch.log:** JVM flags include heap-dump on OOM → likely memory/disk/cluster health issue; often you’ll see OOM or shard allocation errors earlier in that log.

Here’s a focused way to debug and fix on Windows.

---

## 1) Quick health checks (before changing anything)

From PowerShell or Git Bash:

```bash
# Is ES up?
curl -s http://localhost:9200

# Cluster health
curl -s http://localhost:9200/_cluster/health?pretty

# Shards/indices overview (look for UNASSIGNED, RED)
curl -s http://localhost:9200/_cat/indices?v
curl -s http://localhost:9200/_cat/shards?v

# Explain *why* shards are unassigned (if any)
curl -s -XGET "http://localhost:9200/_cluster/allocation/explain?pretty"

# Disk usage (ES stops allocating when watermarks hit)
curl -s http://localhost:9200/_cat/allocation?v
```

What to look for:

* **status: red** or many **UNASSIGNED** shards.
* Disk free too low (ES “high watermark” \~95%).
* “out\_of\_memory” or “circuit\_breaking\_exception” in ES logs.

Also check service ports:

```bash
# Zeebe gateway
curl -s http://localhost:9600/actuator/health
# Operate
curl -s http://localhost:8080/operate/api/health
```

---

## 2) Usual culprits & quick fixes

### A) **Disk watermark reached**

ES will refuse to allocate shards → importers fail.

* Free space on the drive hosting `c8run/elasticsearch/data`.
* Then retry `c8run.exe stop` → `c8run.exe start`.

### B) **Elasticsearch heap too small (OOM)**

If the data grew, ES may OOM and get into a bad state.

* For Camunda Run’s bundled ES, set heap (both min=max) via env var before starting:

  * PowerShell:

    ```powershell
    $env:ES_JAVA_OPTS="-Xms2g -Xmx2g"
    .\c8run.exe start
    ```
  * Or edit the ES jvm options file under `c8run/elasticsearch/config/jvm.options` and set `-Xms2g` / `-Xmx2g` (pick values your RAM can handle).

### C) **Corrupted / incompatible indices after upgrade**

Upgrading Camunda Run but reusing the **old ES data directory** can leave incompatible index metadata.

* In a **dev** environment, the pragmatic fix is to wipe ES & tool data so everything re-creates cleanly (you’ll lose history in Operate/Tasklist):

  1. Stop everything:

     ```powershell
     .\c8run.exe stop
     ```
  2. Backup then delete these folders (paths relative to your Camunda Run directory):

     * `elasticsearch/data`
     * `operate/data`
     * `tasklist/data`
     * (optionally) `zeebe/data` if you don’t need running state
  3. Start fresh:

     ```powershell
     .\c8run.exe start
     ```

This resolves most “Some ES shards failed… retrying” issues in dev.

### D) **Antivirus/file locking**

On Windows, AV can lock ES data files causing shard failures. Exclude your `c8run/elasticsearch/data` folder from AV scanning.

### E) **Multiple ES instances / port conflicts**

Ensure only **one** ES is listening on `:9200`. Check:

```bash
netstat -aon | findstr :9200
```

Kill any stray process if needed.

---

## 3) Validate that Operate importers recover

After applying the fix (free disk / increase heap / fresh data):

1. Start Camunda Run:

   ```powershell
   .\c8run.exe start
   ```
2. Wait 30–60 seconds, then:

   ```bash
   curl -s http://localhost:9200/_cluster/health?pretty
   curl -s http://localhost:9200/_cat/indices?v | grep -E "zeebe|operate|tasklist"
   ```

   You should see indices like `zeebe-record-*`, `operate-*`, etc., **green or yellow**, not red.
3. Trigger your Java POST again; watch Operate for new instances.

---

## 4) Why your symptoms make sense

* Your Java client can still **start processes** (Zeebe gateway OK), but **Operate shows nothing** because its importer can’t ingest from ES (camunda.log “Some ES shards failed…”).
* Connectors importer failing is the same story—can’t write/read ES.
* The ES JVM flags you saw include heap dump options—very common to see **OOM** events in `elasticsearch.log` a bit earlier (search for “OutOfMemoryError”, “circuit\_breaking\_exception”, “flood\_stage”).

---

## 5) If you want to keep data (avoid wipe)

Instead of wiping:

* Fix disk space and heap.
* Use `_cluster/allocation/explain` to see why shards are unassigned and address the cause.
* You can try reroute/allocate APIs, but in dev it’s often faster to back up and start fresh.

---

If you paste the outputs of:

* `_cluster/health`
* `_cat/shards`
* one `_cluster/allocation/explain`

…I can tell you exactly which issue you’ve hit and the minimal change to get you green again.
