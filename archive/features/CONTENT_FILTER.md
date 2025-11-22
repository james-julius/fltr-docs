# Content Filter for Chat Endpoints

## Overview

The FLTR chat endpoints now include comprehensive content filtering to protect against inappropriate content, PII leakage, and malicious inputs. The filter is applied to all incoming chat requests before they reach the LLM.

## Features

### 1. **OpenAI Moderation API Integration**
- Checks all messages against OpenAI's moderation API
- Detects: hate speech, harassment, violence, self-harm, sexual content, etc.
- Blocks requests that violate content policies
- Can be disabled via environment variable

### 2. **PII (Personally Identifiable Information) Detection**
Detects and handles:
- Email addresses
- Phone numbers (US format)
- Social Security Numbers (SSN)
- Credit card numbers
- IP addresses

**Two modes of operation:**
- **Block mode** (default): Rejects requests containing PII
- **Redact mode**: Automatically replaces PII with `[REDACTED]` and continues

### 3. **Input Validation**
- Maximum message length: 10,000 characters per message
- Maximum total length: 50,000 characters across all messages
- Maximum number of messages: 100
- Detects malicious patterns: `<script>` tags, `javascript:` protocol, event handlers

### 4. **PostHog Analytics Integration**
Tracks filtered requests with:
- Filter result (allowed/blocked)
- Reason for blocking
- Types of violations
- PII types detected
- Whether content was redacted

## Configuration

### Environment Variables

Add these to your `.env` file:

```bash
# Enable OpenAI moderation API to check for inappropriate content
# Requires OPENAI_API_KEY to be set. Set to "false" to disable
ENABLE_CONTENT_MODERATION="true"

# Enable PII (Personally Identifiable Information) detection
# Detects: emails, phone numbers, SSNs, credit cards, IP addresses
# Set to "false" to disable
ENABLE_PII_DETECTION="true"

# Auto-redact PII instead of blocking requests
# If "true", PII will be replaced with [REDACTED] and request continues
# If "false", requests with PII will be blocked
# Default: false (blocks requests with PII)
AUTO_REDACT_PII="false"
```

## Usage

The content filter is automatically applied to both chat endpoints:
- `/api/chat` - General chat with dataset tools
- `/api/chat/dataset/[datasetId]` - Dataset-specific chat

### Filter Response Format

When a request is blocked, the API returns a 400 error with details:

```json
{
  "error": "Content filter violation",
  "violations": [
    "Content moderation failed: hate, violence",
    "PII detected: email, phone"
  ]
}
```

### Example: Blocked Request

**Request:**
```json
{
  "messages": [
    {
      "role": "user",
      "content": "My email is john@example.com and phone is 555-123-4567"
    }
  ]
}
```

**Response (with `AUTO_REDACT_PII=false`):**
```json
{
  "error": "PII detected",
  "violations": ["PII detected: email, phone"]
}
```

### Example: Redacted Request

**Request:**
```json
{
  "messages": [
    {
      "role": "user",
      "content": "Contact me at john@example.com"
    }
  ]
}
```

**Internal Processing (with `AUTO_REDACT_PII=true`):**
```json
{
  "messages": [
    {
      "role": "user",
      "content": "Contact me at [EMAIL_REDACTED]"
    }
  ]
}
```

The request continues with redacted content, and the user receives a normal response.

## PostHog Tracking

All filter events are tracked in PostHog with the following properties:

```typescript
{
  event: "content_filter_result",
  distinctId: conversationId,
  properties: {
    allowed: boolean,
    reason?: string,
    violations?: string[],
    piiFound?: { type: string, count: number }[],
    piiRedacted: boolean,
    route: "/api/chat" | "/api/chat/dataset",
    model?: string,
    datasetId?: string,
    messageCount: number,
    contentFiltered: "passed" | "redacted" | "blocked"
  }
}
```

### Analyzing Filter Data in PostHog

You can create insights to track:
- **Filter block rate**: `content_filter_result` where `allowed = false`
- **PII detection rate**: Count of events with `piiFound.length > 0`
- **Most common violations**: Group by `violations`
- **Filter performance**: Compare `contentFiltered` by route/model

## Testing

Run the test suite:

```bash
cd nextjs
pnpm test content-filter.test.ts
```

The test suite covers:
- ✅ Valid message acceptance
- ✅ Length validation
- ✅ Malicious pattern detection
- ✅ Email detection
- ✅ Phone number detection
- ✅ SSN detection
- ✅ Credit card detection
- ✅ Auto-redaction
- ✅ Multiple PII types
- ✅ Edge cases

## Implementation Details

### File Structure

```
nextjs/
├── src/
│   ├── lib/
│   │   └── content-filter.ts          # Core filter logic
│   ├── app/
│   │   └── api/
│   │       └── chat/
│   │           ├── route.ts            # General chat with filter
│   │           └── dataset/
│   │               └── [datasetId]/
│   │                   └── route.ts    # Dataset chat with filter
│   └── __tests__/
│       └── lib/
│           └── content-filter.test.ts  # Test suite
└── .env.example                        # Configuration examples
```

### Core Functions

#### `filterChatContent(messages, config)`

Main filter function that:
1. Validates message count and length
2. Detects PII (if enabled)
3. Calls OpenAI moderation API (if enabled)
4. Tracks results in PostHog
5. Returns filter result with allowed/blocked status

#### `validateMessageSync(message, maxLength)`

Synchronous validation for quick checks:
- Length validation
- Malicious pattern detection (XSS, script injection)

#### `detectPii(messages)`

Detects PII using regex patterns:
- Email: `[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}`
- Phone: `(\+\d{1,3}[-.\s]?)?\(?\d{3}\)?[-.\s]?\d{3}[-.\s]?\d{4}`
- SSN: `\d{3}-\d{2}-\d{4}`
- Credit Card: `\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}`
- IP Address: `\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}`

#### `redactPii(messages)`

Replaces PII with redaction markers:
- `[EMAIL_REDACTED]`
- `[PHONE_REDACTED]`
- `[SSN_REDACTED]`
- `[CARD_REDACTED]`
- `[IP_REDACTED]`

#### `checkModeration(messages)`

Calls OpenAI Moderation API and returns:
```typescript
{
  flagged: boolean,
  categories: string[]  // e.g., ["hate", "violence"]
}
```

## Best Practices

### Development

1. **Test thoroughly**: Use the test suite to verify filter behavior
2. **Monitor PostHog**: Track filter metrics to tune thresholds
3. **Review false positives**: Adjust PII patterns if needed
4. **Consider user experience**: Decide between blocking vs. redaction

### Production

1. **Enable all filters**: Set `ENABLE_CONTENT_MODERATION=true` and `ENABLE_PII_DETECTION=true`
2. **Choose redaction mode carefully**: `AUTO_REDACT_PII=true` provides better UX but may allow some PII
3. **Monitor filter impact**: Track block rates and user feedback
4. **Review violations regularly**: Use PostHog to identify patterns

### Privacy

1. **PostHog tracking**: Set `privacyMode: true` if you don't want to send message content to PostHog
2. **Moderation API**: OpenAI may store moderation requests for 30 days
3. **PII redaction**: Redacted messages are sent to the LLM, not original content
4. **Logging**: Don't log full message content in production

## Limitations

1. **PII Detection**: Regex-based, may have false positives/negatives
2. **Moderation API**: Requires OpenAI API key and has rate limits
3. **Language Support**: PII patterns are primarily for English/US formats
4. **Performance**: Adds ~200-500ms latency per request (moderation API call)

## Future Enhancements

- [ ] Add support for international phone/SSN formats
- [ ] Implement custom PII patterns per organization
- [ ] Add ML-based content classification
- [ ] Support for custom blocklists/allowlists
- [ ] Rate limiting based on filter violations
- [ ] Automatic user warnings on repeated violations

## Support

For issues or questions:
- Open an issue on GitHub
- Check PostHog analytics for filter patterns
- Review test suite for expected behavior

## License

Same as FLTR project license.
