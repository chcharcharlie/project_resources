# Web Inspector MCP - Discoverability Improvements

## Changes Made to Improve Claude's Tool Usage

I've updated your MCP server to make it much clearer to Claude when to use the `fetch_webpage` tool. Here are the key changes:

### 1. Enhanced Tool Description
**Before:**
```
"Fetch the HTML content of one or more web pages using distributed browser network"
```

**After:**
```
"Alternative web fetcher for when standard fetch fails. Uses distributed browser network with real human IPs to bypass blocks, handle JavaScript-rendered content, and access pages that return "Failed to fetch" errors. Ideal fallback when web_fetch encounters access denied, 403, or connection refused errors."
```

### 2. Key Improvements
- **Clear positioning as a fallback**: The description now explicitly states it's an "alternative" for "when standard fetch fails"
- **Specific error keywords**: Mentions "Failed to fetch", "403", "access denied" - the exact errors Claude sees
- **Use case clarity**: Explains it bypasses blocks and handles JavaScript content
- **Trigger phrases**: The description includes terms that should trigger usage when Claude encounters fetch failures

### 3. Additional Updates
- Updated package.json with better keywords
- Enhanced README with "When to Use This Tool" section
- Added CLAUDE_GUIDE.md with usage examples
- Improved example files

## What You Need to Do

1. **Rebuild and restart the MCP server** in Claude Desktop:
   ```bash
   cd ~/claude_space/repos/claude_web-inspector-mcp
   npm run build
   ```

2. **Restart Claude Desktop** to reload the MCP server with the new description

3. **Test the improved behavior**:
   - Try asking Claude to fetch a page that typically blocks
   - When Claude encounters "Failed to fetch", it should now automatically consider using your tool
   - You can also use trigger phrases like "try alternative methods if the normal fetch fails"

## Expected Behavior

With these changes, Claude should now:
1. First try the built-in `web_fetch` tool
2. When it encounters errors (403, failed to fetch, etc.), recognize your tool as a fallback option
3. Automatically use `fetch_webpage` without requiring explicit instruction

## Tips for Better Results

- Let Claude fail first - this reinforces the tool's role as a fallback
- Mention if a site is "protected" or "blocks bots" to trigger proactive usage
- Use phrases like "use any available tools" when you want to ensure fallback usage

The key insight is that Claude needs clear signals in the tool description itself about when to use alternative tools. By including the specific error messages and positioning it as a fallback option, Claude should now make better tool selection decisions automatically.
