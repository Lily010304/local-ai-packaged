# n8n Workflow Troubleshooting Guide

## Issue: "Model output doesn't fit required format" in Structured Output Parser

### Problem Description
The LLM returns nested JSON like `{"output": {"title": "...", "summary": "..."}}` instead of flat JSON like `{"title": "...", "summary": "..."}`.

### Root Cause
Ollama models with `"format": "json"` sometimes add wrapper keys despite instructions. This is common with:
- Smaller models (8B parameters or less)
- Models trained on function calling datasets
- When using JSON schema in prompts

### Already Implemented Solution ✅

The **"Normalize Output"** code node already handles this issue by:

1. **Unwrapping nested structures**:
   ```javascript
   if (obj.output && typeof obj.output === 'object' && !Array.isArray(obj.output))
     obj = obj.output;
   ```

2. **Removing JSON schema keys**:
   ```javascript
   const schemaKeys = ['type', 'properties', 'required', 'additionalProperties'];
   const hasSchema = schemaKeys.some(k => Object.prototype.hasOwnProperty.call(obj, k));
   ```

3. **Parsing code fences**:
   ```javascript
   if (t.startsWith('```')) {
     t = t.replace(/^```[a-zA-Z]*\n?/, '').replace(/```$/, '').trim();
   }
   ```

4. **Providing fallback values**:
   ```javascript
   return {
     title: 'Untitled Notebook',
     summary: 'Summary unavailable.',
     example_questions: DEFAULT_QUESTIONS,
     emoji: '📄'
   };
   ```

### How the Workflow Should Flow

```
Webhook → If → Extract/Set Text → Generate Title & Description (LLM)
                                          ↓
                                   Normalize Output (Unwraps nested JSON)
                                          ↓
                                   Random Color
                                          ↓
                                   Update Supabase Row
```

### Verification Steps

1. **Check n8n workflow is active**:
   ```powershell
   docker exec -it n8n n8n list:workflow
   ```

2. **Check Ollama model is available**:
   ```powershell
   docker exec -it ollama ollama list
   ```

3. **Test the workflow manually**:
   - Go to n8n UI: `http://localhost:5678`
   - Open "InsightsLM - Generate Notebook Details"
   - Click "Execute Workflow" with test data
   - Inspect each node's output

4. **Check if Normalize Output node is executing**:
   - In n8n, after execution, click "Normalize Output" node
   - Verify it has input data
   - Check the `output` object is properly formatted

### Common Issues & Fixes

#### Issue 1: Workflow Stops at "Generate Title & Description"
**Symptoms**: Node shows error, no data flows to "Normalize Output"

**Fix**: Check Ollama model connection
```powershell
# Check if Ollama container is running
docker ps | Select-String "ollama"

# Check if model exists
docker exec -it ollama ollama list | Select-String "qwen3"

# If model missing, pull it
docker exec -it ollama ollama pull qwen3:8b-q4_K_M
```

#### Issue 2: "Normalize Output" Not Unwrapping Nested JSON
**Symptoms**: `Update a row` node fails with "Cannot read property 'title' of undefined"

**Fix**: The code expects specific input structure. Verify the LLM output by:
1. Clicking "Generate Title & Description" node after execution
2. Check the `text` field in JSON output
3. Ensure it contains valid JSON (not HTML, not error message)

**Code to manually test parsing**:
```javascript
const input = '{"output": {"title": "Test", "summary": "Test summary", "example_questions": ["Q1?"], "emoji": "📄"}}';
const parsed = JSON.parse(input);
const unwrapped = parsed.output && typeof parsed.output === 'object' ? parsed.output : parsed;
console.log(unwrapped); // Should show flat object
```

#### Issue 3: Webhook Returns 202 But Nothing Happens
**Symptoms**: POST to webhook succeeds, but notebook doesn't update

**Fix**: Check workflow execution history
1. Go to n8n UI → "Executions" tab
2. Find the recent execution
3. Click to view detailed logs
4. Identify which node failed

Common causes:
- Supabase credentials expired
- Notebook ID doesn't exist in database
- Extract Text sub-workflow failed

#### Issue 4: LLM Returns Code Fences (```json...```)
**Symptoms**: Normalize Output extracts empty data

**Fix**: Already handled by this code:
```javascript
if (t.startsWith('```')) {
  t = t.replace(/^```[a-zA-Z]*\n?/, '').replace(/```$/, '').trim();
}
```

Verify by checking "Normalize Output" node input - should show cleaned JSON.

### Prevention Strategies

#### 1. Use Stronger Models (Better instruction following)
Replace `qwen3:8b-q4_K_M` with:
- `qwen3:14b` (larger, more accurate)
- `llama3.1:8b` (OpenAI-style training)
- `mistral:7b-instruct` (good at JSON)

Update in workflow:
```json
{
  "parameters": {
    "model": "llama3.1:8b",
    "options": {
      "temperature": 0.1,
      "format": "json"
    }
  }
}
```

#### 2. Lower Temperature (More deterministic)
Current: `"temperature": 0.2`
Recommended: `"temperature": 0.1` (or even 0.0 for pure JSON generation)

#### 3. Add System Message
Some models respond better to system messages:

```json
{
  "parameters": {
    "messages": {
      "messageValues": [
        {
          "type": "system",
          "message": "You are a JSON-only API. Never include explanations or wrappers. Return ONLY the requested JSON structure."
        },
        {
          "message": "=Generate JSON for: {{ $json.extracted_text }}"
        }
      ]
    }
  }
}
```

#### 4. Use Few-Shot Examples
Add example input/output pairs in the prompt:

```
EXAMPLE 1:
Input: "Python is a programming language"
Output: {"title": "Python Overview", "summary": "Introduction to Python programming language.", "example_questions": ["What is Python?"], "emoji": "🐍"}

EXAMPLE 2:
Input: "Machine learning uses algorithms"
Output: {"title": "Machine Learning Basics", "summary": "Overview of ML algorithms.", "example_questions": ["What is machine learning?"], "emoji": "🤖"}

NOW GENERATE FOR THIS INPUT:
{{ $json.extracted_text }}
```

### Testing the Fix

1. **Re-import the updated workflow**:
   ```powershell
   # In n8n UI, delete old workflow
   # Import InsightsLM___Generate_Notebook_Details.json
   ```

2. **Create test notebook**:
   ```sql
   INSERT INTO notebooks (id, title, generation_status) 
   VALUES ('test-123', 'Test Notebook', 'processing');
   ```

3. **Trigger workflow**:
   ```powershell
   $headers = @{
       "Authorization" = "Bearer 395ca650972342ee8a4778957ffed288294c6400827d4eb8b455f8f87edfe03b"
       "Content-Type" = "application/json"
   }
   
   $body = @{
       notebookId = "test-123"
       filePath = "https://raw.githubusercontent.com/python/cpython/main/README.rst"
       sourceType = "website"
   } | ConvertTo-Json
   
   Invoke-RestMethod -Uri "http://localhost:5678/webhook/6f3e4a1b-9b2d-4c7a-ae7a-1cde5f62b2f4" -Method POST -Headers $headers -Body $body
   ```

4. **Verify in Supabase**:
   ```sql
   SELECT id, title, description, generation_status, example_questions, icon
   FROM notebooks 
   WHERE id = 'test-123';
   ```

Expected result:
- `title`: Populated with document title
- `description`: Populated with summary
- `generation_status`: "completed"
- `example_questions`: Array of 3-5 questions
- `icon`: Emoji character

### When to Contact Support

If after following all steps above:
1. Normalize Output still shows nested `{"output": {...}}` in final JSON
2. Workflow executes but notebook row not updated
3. Ollama consistently returns invalid JSON

Then check:
1. n8n version: `docker exec -it n8n n8n --version`
2. Ollama version: `docker exec -it ollama ollama --version`
3. Model compatibility: Some models don't support `"format": "json"`

### Quick Reference: Error Messages

| Error | Cause | Fix |
|-------|-------|-----|
| "Model output doesn't fit required format" | Nested JSON or schema keys | Verify "Normalize Output" is connected |
| "Cannot read property 'title'" | Output structure invalid | Check LLM returned valid JSON |
| "404 Not Found" (webhook) | Expired ngrok URL | Update VITE_N8N_URL in .env |
| "Connection refused" (Ollama) | Ollama container down | `docker start ollama` |
| "Model not found: qwen3:8b" | Model not pulled | `docker exec -it ollama ollama pull qwen3:8b-q4_K_M` |

### Summary

✅ **The workflow is already designed to handle this error**
✅ **"Normalize Output" node unwraps nested JSON**
✅ **Updated prompt is more explicit about forbidden patterns**
✅ **Fallback values prevent total failures**

**Next time this happens**:
1. Check "Normalize Output" node executed
2. Verify LLM returned valid JSON (not error text)
3. If persists, lower temperature to 0.1
4. Consider switching to a stronger model

The error message "Model output doesn't fit required format" is misleading - it's actually **expected** with some models, and the workflow handles it gracefully through the normalization step.
