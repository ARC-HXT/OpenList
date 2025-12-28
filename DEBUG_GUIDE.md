# Debug Guide: 123open Upload Timeout Issue

## Quick Start

This enhanced version adds comprehensive logging to help analyze the timeout issue when copying large files from Baidu Netdisk to 123open storage.

## What Was Added

### 1. Detailed Logging Throughout the Upload Process

The code now logs every critical step:

- **Transfer initialization**: Source/destination storage info, file metadata
- **MD5 computation**: When it starts, how long it takes, final hash
- **Upload session creation**: API response details (Reuse, FileID, PreuploadID)
- **Chunk uploads**: Each chunk's progress, MD5, timing, and status
- **Complete polling**: **Most important** - Every poll attempt with full API response

### 2. Critical Data Points Captured

The most important logs to watch are the `complete()` polling responses:

```log
[123open] File: <filename> - Complete poll #N: Code=<code>, Message=<msg>, Completed=<bool>, FileID=<id>
```

This will show:
- What error codes the 123open API actually returns (like the mysterious 20103)
- Whether `Completed` stays false or changes
- Whether `FileID` is set or remains 0
- Any error messages in the response

## How to Use

### Step 1: Enable Logging

Make sure your OpenList configuration has logging enabled. Set log level to at least INFO (DEBUG is even better for detailed chunk logs).

### Step 2: Reproduce the Issue

Copy a large file (>100MB recommended) from Baidu Netdisk to 123open storage.

### Step 3: Collect the Logs

The logs will show the entire flow. Filter for the most relevant entries:

```bash
grep "\[123open\]\|\[Transfer\]" /path/to/openlist.log > debug_output.log
```

### Step 4: Analyze the Complete() Responses

Look at the complete polling section. You'll see 60 attempts (or until success). Example:

```log
[123open] File: largefile.zip - Starting complete() polling (max 60 attempts)
[123open] File: largefile.zip - Complete poll #1: Code=0, Message="", Completed=false, FileID=0
[123open] File: largefile.zip - Complete poll #2: Code=0, Message="", Completed=false, FileID=0
...
[123open] File: largefile.zip - Complete poll #60: Code=0, Message="", Completed=false, FileID=0
[123open] File: largefile.zip - Upload timeout after 60 polls (60s total)
```

## What to Look For

### Scenario 1: Normal Processing Delay
```log
Complete poll #1-5: Completed=false, FileID=0
Complete poll #6: Completed=true, FileID=12345
```
**Meaning**: Server just needed time to process. Solution: Increase polling count or interval.

### Scenario 2: Unknown Error Code
```log
Complete poll #1-60: Code=20103, Message="Unknown error", Completed=false
```
**Meaning**: Server is returning an error, possibly validation failure. Need to investigate error code meaning.

### Scenario 3: Stuck State
```log
Complete poll #1-60: Code=0, Message="", Completed=false, FileID=0
```
**Meaning**: Server stuck without giving feedback. Possible reasons:
- Data integrity issue (MD5 mismatch)
- Server-side processing failure
- Missing chunks

### Scenario 4: Partial Success
```log
Complete poll #1-30: Completed=false, FileID=0
Complete poll #31-60: Code=404, Message="Upload session not found"
```
**Meaning**: Upload session expired or was cleaned up. Timing issue.

## Comparison Test

To isolate the issue, also test:

### Direct Upload (Should Work)
Upload the same file size directly to 123open (not from Baidu):
```
Local file → 123open
```

Compare the logs:
1. MD5 computation time
2. Upload duration
3. Complete() response pattern
4. Time from "upload completed" to first complete poll

### Different Source (Should Work)
Upload from Aliyun instead of Baidu:
```
Aliyun → 123open
```

Compare to identify Baidu-specific issues.

## Log Locations

Key log entries by component:

### Transfer Task (`internal/fs/copy_move.go`)
```log
[Transfer] Source: /baidu (baidu_netdisk) -> Dest: /123open (123_open), File: test.zip, Size: 104857600 bytes
[Transfer] Link obtained for test.zip in 250ms, URL: https://...
[Transfer] Stream created for test.zip in 100ms, Size: 104857600 bytes, Hash available: true (MD5: abc123...)
[Transfer] Upload completed for test.zip in 5m30s
```

### Upload Session (`drivers/123_open/driver.go`)
```log
[123open] Starting upload - file: test.zip, size: 104857600 bytes
[123open] File: test.zip - Initial hash available: false, hash: 
[123open] File: test.zip - Computing MD5 via CacheFullAndHash (this may take time for large files)
[123open] File: test.zip - CacheFullAndHash completed in 2m15s, MD5: abc123...
[123open] File: test.zip - Creating upload session with MD5: abc123...
[123open] File: test.zip - Create response: Reuse=false, FileID=0, PreuploadID=xyz789, SliceSize=5242880
[123open] File: test.zip - Starting chunk upload
[123open] File: test.zip - Chunk upload completed in 3m10s
[123open] File: test.zip - Starting complete() polling (max 60 attempts)
```

### Chunk Upload (`drivers/123_open/upload.go`)
```log
[123open] Upload - file: test.zip, domain: https://upload.123pan.com, total size: 104857600, chunk size: 5242880
[123open] Upload - file: test.zip, total chunks: 20, upload threads: 3
[123open] Upload - file: test.zip, chunk 1/20 uploaded successfully in 2.5s
[123open] Upload - file: test.zip, chunk 2/20 uploaded successfully in 2.3s
...
[123open] Upload - file: test.zip, all 20 chunks uploaded successfully in 3m10s
```

### Complete Polling (THE MOST IMPORTANT)
```log
[123open] File: test.zip - Complete poll #1: Code=0, Message="", Completed=false, FileID=0
[123open] File: test.zip - Complete poll #2: Code=0, Message="", Completed=false, FileID=0
...
```

## Common Issues and Solutions

### Issue: CacheFullAndHash Takes Too Long
**Log**: `CacheFullAndHash completed in 10m30s`
**Cause**: Slow download from Baidu
**Solution**: May need to optimize caching strategy

### Issue: Chunks Upload Slowly
**Log**: `chunk N/M uploaded in 30s` (consistently slow)
**Cause**: Network issues or rate limiting
**Solution**: Check network, adjust thread count

### Issue: Complete Always Returns False
**Log**: All 60 polls show `Completed=false, FileID=0`
**Cause**: Most likely the actual bug - need to check error codes and messages
**Solution**: Analyze the Code and Message fields in complete responses

## Next Steps After Analysis

Once you have the logs, look for:

1. **What error code/message does complete() actually return?**
   - This is the key to understanding the failure

2. **How long does each phase take?**
   - MD5 computation
   - Chunk upload
   - Time between last chunk and first complete poll

3. **Are all chunks uploaded successfully?**
   - Check for any chunk upload failures
   - Verify chunk count matches expected

4. **Does the stream from Baidu have any issues?**
   - Link expiration time
   - Stream size correctness
   - MD5 availability

## Reporting Results

When sharing results, please include:

1. Complete log output (filtered for `[123open]` and `[Transfer]`)
2. File size that was tested
3. How long before timeout occurred
4. Any patterns in the complete() responses
5. Comparison with direct upload (if tested)

## Technical Details

For full technical analysis, see: `ANALYSIS_123OPEN_TIMEOUT.md`

This includes:
- Complete code flow diagrams
- 5 detailed root cause hypotheses
- Expected findings scenarios
- In-depth analysis methodology
