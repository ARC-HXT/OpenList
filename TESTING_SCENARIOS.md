# Testing Scenarios for 123open Upload Issue

## Test Matrix

| Test # | Source | Destination | File Size | Expected Result | Purpose |
|--------|--------|-------------|-----------|-----------------|---------|
| 1 | Baidu | 123open | 50 MB | ❌ Timeout? | Establish baseline - when does it fail? |
| 2 | Baidu | 123open | 150 MB | ❌ Timeout | Confirm large file issue |
| 3 | Baidu | 123open | 500 MB | ❌ Timeout | Test with very large file |
| 4 | Local | 123open | 150 MB | ✅ Success | Verify 123open can handle the size |
| 5 | Aliyun | 123open | 150 MB | ✅ Success? | Test with different source |
| 6 | Baidu | 123open | 150 MB (exists) | ✅ Quick Upload | Verify quick upload works |
| 7 | Baidu | Aliyun | 150 MB | ✅ Success | Verify Baidu download works |

## Detailed Test Steps

### Test 1: Small File from Baidu to 123open
**Purpose**: Determine minimum file size that triggers the issue

**Steps**:
1. Find or create a 50 MB file in Baidu Netdisk
2. Copy to 123open via OpenList
3. Check logs for completion status

**Expected Logs**:
```log
[123open] File: test-50mb.zip - CacheFullAndHash completed in ~1m
[123open] Upload - all N chunks uploaded successfully in ~2m
[123open] File: test-50mb.zip - Complete poll #X: Completed=true, FileID=12345
```

**If it fails**: Issue affects files smaller than expected.
**If it succeeds**: Issue is size-dependent or timing-dependent.

---

### Test 2: Large File from Baidu to 123open (Expected Failure)
**Purpose**: Reproduce the reported issue

**Steps**:
1. Use a 150 MB file in Baidu Netdisk
2. Copy to 123open via OpenList
3. Wait for timeout (up to 60 seconds after upload completes)
4. Capture complete logs

**Expected Logs**:
```log
[123open] File: test-150mb.zip - CacheFullAndHash completed in ~3m
[123open] Upload - all N chunks uploaded successfully in ~5m
[123open] File: test-150mb.zip - Starting complete() polling (max 60 attempts)
[123open] File: test-150mb.zip - Complete poll #1-60: Completed=false
[123open] File: test-150mb.zip - Upload timeout after 60 polls
```

**Focus on**: What do the complete() responses actually contain?

---

### Test 3: Direct Upload to 123open (Expected Success)
**Purpose**: Prove that 123open can handle the file size and format

**Steps**:
1. Upload the same 150 MB file directly to 123open (not from another cloud storage)
2. Use local file or direct HTTP upload
3. Compare logs with Test 2

**Expected Logs**:
```log
[123open] File: test-150mb.zip - CacheFullAndHash completed in <30s
[123open] Upload - all N chunks uploaded successfully in ~3m
[123open] File: test-150mb.zip - Complete poll #5: Completed=true, FileID=12345
```

**Compare with Test 2**:
- MD5 computation time (should be much faster)
- Upload time (might be similar)
- Complete response (should succeed)

---

### Test 4: Quick Upload Test (Expected Success)
**Purpose**: Verify that the 123open API works when file already exists

**Steps**:
1. Manually upload a file to 123open first (via web interface)
2. Try to copy the same file from Baidu to 123open
3. Should trigger quick upload (秒传)

**Expected Logs**:
```log
[123open] File: test-150mb.zip - Create response: Reuse=true, FileID=12345
[123open] File: test-150mb.zip - Quick upload successful (reuse)
```

**Result**: Should complete immediately without timeout.

---

### Test 5: Baidu to Aliyun (Expected Success)
**Purpose**: Verify that the issue is specific to 123open as destination

**Steps**:
1. Copy the same 150 MB file from Baidu to Aliyun
2. Should complete successfully according to problem description

**Expected Result**: Success without timeout.

---

## Data Collection Template

For each test, record:

```markdown
### Test N: [Description]

**File**: [filename and size]
**Source**: [storage name and driver]
**Destination**: [storage name and driver]

**Timing**:
- CacheFullAndHash: [duration]
- Chunk upload: [duration]
- Total time before timeout/success: [duration]

**Complete() Responses** (first 5 and last 5):
Poll #1: Code=[?], Message=[?], Completed=[?], FileID=[?]
Poll #2: Code=[?], Message=[?], Completed=[?], FileID=[?]
...
Poll #60: Code=[?], Message=[?], Completed=[?], FileID=[?]

**Result**: [Success/Timeout/Error]

**Notes**: [Any observations]
```

## Key Metrics to Compare

| Metric | Baidu→123open | Local→123open | Baidu→Aliyun |
|--------|---------------|---------------|--------------|
| MD5 computation time | ? | ? | ? |
| Chunk upload time | ? | ? | ? |
| Time to first complete poll | ? | ? | N/A |
| Complete poll responses | ? | ? | N/A |
| Success rate | 0% | ? | 100% |

## Analysis Checklist

After running tests, answer these questions:

- [ ] What error codes/messages appear in complete() responses?
- [ ] Is the timeout duration the issue, or does complete() never return true?
- [ ] Does file size affect the failure rate?
- [ ] How does MD5 computation time compare between direct and cross-storage?
- [ ] Are all chunks uploaded successfully before timeout?
- [ ] Is there a pattern in the timing (e.g., always fails at 60 seconds)?
- [ ] Does the stream from Baidu have any observable issues?

## Automated Test Script (Optional)

If you want to automate testing, you could create a script:

```bash
#!/bin/bash
# test_upload.sh

SIZES=(50 100 150 200)
SOURCE="Baidu"
DEST="123open"

for SIZE in "${SIZES[@]}"; do
    echo "Testing ${SIZE}MB file"
    # Your OpenList API call here to trigger copy
    # Wait for completion or timeout
    # Parse logs
    # Record results
done
```

## Expected Outcome

After running these tests, you should be able to:

1. **Confirm the exact failure scenario**
   - File size threshold (if any)
   - Source/destination combination
   
2. **Identify the API response pattern**
   - What complete() actually returns
   - Whether it's a timeout issue or API error
   
3. **Isolate the root cause**
   - Baidu-specific issue?
   - 123open API limitation?
   - Data integrity problem?
   - Timing/synchronization issue?

4. **Determine the fix approach**
   - Adjust polling strategy?
   - Fix MD5 handling?
   - Implement retry logic?
   - Handle specific error codes?

## Reporting Format

When reporting results, please provide:

```markdown
## Test Results Summary

**Environment**:
- OpenList version: [version]
- Date: [date]
- Network: [conditions]

**Test Results**:
[Use the data collection template for each test]

**Key Findings**:
1. [Finding 1]
2. [Finding 2]
...

**Complete() Response Patterns**:
[What you observed in the polling logs]

**Conclusion**:
[Your analysis of what the root cause appears to be]

**Attached Logs**:
- test1_50mb.log
- test2_150mb_failure.log
- test3_150mb_direct_success.log
```

## Next Steps

Based on test results, the next steps will be:

1. **If complete() returns error codes**: Research those specific error codes in 123open API docs
2. **If complete() never returns true**: Investigate why server processing fails
3. **If MD5 mismatch**: Fix the CacheFullAndHash implementation
4. **If timing issue**: Adjust polling strategy (longer timeout, exponential backoff)
5. **If Baidu stream issue**: Implement better stream handling/validation

---

**Remember**: The goal is not to fix it yet, but to understand exactly what's happening through detailed logging and systematic testing.
