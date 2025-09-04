Thanks for sharing those results — they’re actually very revealing.

👉 Your Elasticsearch cluster is **healthy** (`status: "green"`, `0 unassigned shards`, everything `STARTED`). So ES itself is not “down” or broken — all shards are allocated, no red/yellow, no allocation issues.

That means the root cause is likely **not Elasticsearch storage corruption**, but instead **the integration layer (Operate importers) not pulling records from Zeebe into ES**.

---

### 🔍 What this suggests

* Zeebe is still running (since you can start processes from Java and get no errors).
* ES is up (green, healthy, 1014 indices/shards).
* But **no new data is being written into ES indices** by the importers — that’s why Operate shows nothing.

This lines up with your earlier log snippet:

```
io.camunda.operate.zeebeimport.elasticsearch.ElasticsearchRecordsReader - Exception occurred ...
while obtaining next Zeebe records batch: Some ES shards failed. Ignoring search response and retrying
```

Since ES now reports **healthy**, it may be that:

1. **The importers are stuck** in a bad retry loop from a transient failure yesterday.
2. **Version mismatch**: you’re running Camunda 8.7.x with Elasticsearch 8.13.x. Camunda bundles a specific ES version that it’s tested against. Running a higher ES (like 8.13) can cause subtle issues even if cluster health is green.
3. **Old indices** are present but Operate/Tasklist don’t like their mappings (e.g., index created with 8.7.2 configs, but now you’re on 8.7.11 and ES 8.13).

---

### ✅ Next steps to narrow down

1. **Check Operate’s logs in more detail** (`c8run/log/camunda.log`) for messages repeating after startup:

   * Search for: `ERROR`, `WARN`, `zeebeimport`, `operate`.
   * Is it still complaining about “Some ES shards failed”? If yes, which alias/index?

2. **Check which indices are being written to recently**:

   ```bash
   curl -s http://localhost:9200/_cat/indices?v | sort -k7 -r | head -20
   ```

   Look at the `docs.count` and `store.size`. Are `zeebe-*` indices still growing when you start new process instances? If not, the import isn’t working.

3. **Confirm Zeebe → ES pipeline is alive**:

   * Zeebe gateway health:

     ```bash
     curl -s http://localhost:9600/actuator/health
     ```
   * Operate health:

     ```bash
     curl -s http://localhost:8080/operate/api/health
     ```

---

### 🚑 Practical options (since you said dev box)

* **Option A (clean slate reset)**: If you don’t care about historic instances, wipe ES + Operate data directories:

  * Stop c8run
  * Delete:

    * `c8run/elasticsearch/data`
    * `c8run/operate/data`
    * `c8run/tasklist/data`
    * (optional) `c8run/zeebe/data`
  * Restart c8run → check Operate with a new instance.
  * This removes possible mapping mismatches.

* **Option B (version alignment)**: If you installed ES separately and it’s 8.13.4, but Camunda 8.7.11 bundles an older ES (8.7.x ships with ES 8.8.2 by default), you may need to run the **bundled ES**, not your system-wide 8.13. Mixing can cause importer incompatibility.

* **Option C (diagnose further)**:
  If you want to keep the data, check exactly which ES indices Operate is failing on. Then we can look at reindexing/mapping issues.

---

✅ **Summary:**
Your ES cluster is healthy (green, 0 unassigned), so the issue is almost certainly:

* **Camunda Operate importers stuck / mismatched with ES version**
* **Or old indices incompatible after version bump**

---

👉 Would you like me to give you the **exact wipe-and-restart steps** (safe for dev, destructive of old history) so you can confirm quickly whether Operate starts showing new processes again? Or do you prefer we go the **non-destructive path** and dig into logs/indices to salvage the existing data?
