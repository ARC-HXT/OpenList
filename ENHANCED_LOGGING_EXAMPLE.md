# Enhanced Logging Example

## What Changed

The logging for the `complete()` polling has been enhanced to always show the full API response details, even when errors occur.

## Before Enhancement

When the API returned an error, only the error message was logged:

```log
WARN[2025-12-28 17:31:18] [123open] File: sharedassets5.assets.resS - Complete poll #1: API error: 文件正在校验中,请间隔1秒后再试
WARN[2025-12-28 17:31:19] [123open] File: sharedassets5.assets.resS - Complete poll #2: API error: 文件上传失败
```

**Problem**: We couldn't see the actual error Code that the API returned. The Code value was lost in the error conversion.

## After Enhancement

Now, the full API response is logged even when there's an error:

```log
WARN[2025-12-28 17:31:18] [123open] File: sharedassets5.assets.resS - Complete poll #1: Code=10001, Message=文件正在校验中,请间隔1秒后再试, Completed=false, FileID=0 (API Error: 文件正在校验中,请间隔1秒后再试)
WARN[2025-12-28 17:31:19] [123open] File: sharedassets5.assets.resS - Complete poll #2: Code=20103, Message=文件上传失败, Completed=false, FileID=0 (API Error: 文件上传失败)
```

**Benefit**: Now we can see:
- The actual error **Code** (e.g., 10001, 20103)
- The error **Message** (Chinese text)
- The **Completed** status (false)
- The **FileID** value (0)
- The original error for context

## Why This Matters

The error codes are undocumented or poorly documented in the 123open API. By capturing the actual Code values, we can:

1. **Identify patterns**: Does error code 20103 always mean "upload failed"?
2. **Distinguish error types**: Is code 10001 a "processing" state vs 20103 a "failure" state?
3. **Debug edge cases**: What other error codes exist that we haven't seen?
4. **Implement proper handling**: Once we know the codes, we can handle them specifically

## Implementation Details

### Code Changes

1. **In `drivers/123_open/upload.go` - `complete()` function**:
   - Changed from `return nil, err` to `return &resp, err` on error
   - This ensures the response struct is returned even when there's an error
   - The response struct contains the parsed API response with Code, Message, etc.

2. **In `drivers/123_open/driver.go` - `Put()` method**:
   - Updated the error logging to access `uploadCompleteResp` fields directly
   - Added Code, Message, Completed, and FileID to the warning log
   - Kept the error message at the end for context

### Why This Works

The `Request()` function in `util.go` already parses the response into the struct before returning an error. Our change ensures that parsed response is not discarded when an error occurs, allowing the caller to log all the details.

## Expected Diagnostic Value

With this enhanced logging, when users encounter the timeout issue, we'll see exactly what the 123open API is returning:

### Scenario 1: Processing Delay
```log
Poll #1: Code=0, Message=文件正在校验中, Completed=false, FileID=0
Poll #2: Code=0, Message=文件正在校验中, Completed=false, FileID=0
...
Poll #10: Code=0, Message=, Completed=true, FileID=123456
```
**Diagnosis**: Server just needs more time, increase polling duration.

### Scenario 2: Validation Failure
```log
Poll #1: Code=0, Message=文件正在校验中, Completed=false, FileID=0
Poll #2: Code=20103, Message=文件上传失败, Completed=false, FileID=0
Poll #3-60: Code=20103, Message=文件上传失败, Completed=false, FileID=0
```
**Diagnosis**: Upload validation failed (possibly MD5 mismatch), need to investigate data integrity.

### Scenario 3: Unknown Error Code
```log
Poll #1-60: Code=99999, Message=未知错误, Completed=false, FileID=0
```
**Diagnosis**: Undocumented error code, need to contact 123open support or find workaround.

## Testing

To verify the enhancement works correctly:

1. Build the project: `go build -tags jsoniter -o openlist .`
2. Run OpenList with INFO or DEBUG logging enabled
3. Attempt to copy a file from Baidu to 123open
4. Check the logs for the enhanced complete() polling output
5. Verify all fields are present in both success and error cases

## Related Documentation

- Technical analysis: See `ANALYSIS_123OPEN_TIMEOUT.md` section on "Hypothesis 3: Server-Side Processing Delay"
- Debug guide: See `DEBUG_GUIDE.md` section on "Scenario 2: Unknown Error Code"
- Testing: See `TESTING_SCENARIOS.md` for systematic test cases
