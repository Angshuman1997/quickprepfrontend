# Streaming LLM Responses in UI

## Question
How do you stream LLM responses in a UI?

## Answer
Streaming LLM responses provides immediate feedback by displaying text as it's generated. The most common approach uses Server-Sent Events (SSE) with React.

## Basic Implementation

### Server-Side (Next.js API Route)
```javascript
// app/api/chat/stream/route.js
import { openai } from '@/lib/openai';

export async function POST(request) {
  const { message } = await request.json();

  const stream = await openai.chat.completions.create({
    model: "gpt-4",
    messages: [{ role: "user", content: message }],
    stream: true,
  });

  const readableStream = new ReadableStream({
    async start(controller) {
      for await (const chunk of stream) {
        const content = chunk.choices[0]?.delta?.content || '';
        
        if (content) {
          const data = `data: ${JSON.stringify({ content })}\n\n`;
          controller.enqueue(data);
        }
      }
      
      controller.enqueue(`data: ${JSON.stringify({ done: true })}\n\n`);
      controller.close();
    },
  });

  return new Response(readableStream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
    },
  });
}
```

### React Component
```jsx
import { useState, useEffect, useRef } from 'react';

export default function StreamingChat() {
  const [messages, setMessages] = useState([]);
  const [currentResponse, setCurrentResponse] = useState('');
  const [isStreaming, setIsStreaming] = useState(false);
  const [input, setInput] = useState('');

  const sendMessage = async () => {
    if (!input.trim() || isStreaming) return;

    const userMessage = { role: 'user', content: input };
    setMessages(prev => [...prev, userMessage]);
    setInput('');
    setIsStreaming(true);
    setCurrentResponse('');

    const response = await fetch('/api/chat/stream', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ message: input }),
    });

    const reader = response.body.getReader();
    const decoder = new TextDecoder();

    while (true) {
      const { done, value } = await reader.read();
      if (done) break;

      const chunk = decoder.decode(value);
      const lines = chunk.split('\n');

      for (const line of lines) {
        if (line.startsWith('data: ')) {
          const data = line.slice(6);
          const parsed = JSON.parse(data);
          
          if (parsed.done) {
            setIsStreaming(false);
            setMessages(prev => [...prev, { 
              role: 'assistant', 
              content: currentResponse 
            }]);
            setCurrentResponse('');
            return;
          }
          
          if (parsed.content) {
            setCurrentResponse(prev => prev + parsed.content);
          }
        }
      }
    }
  };

  return (
    <div className="chat">
      <div className="messages">
        {messages.map((msg, idx) => (
          <div key={idx} className={`message ${msg.role}`}>
            {msg.content}
          </div>
        ))}
        
        {isStreaming && (
          <div className="message assistant">
            {currentResponse}
            <span className="cursor">|</span>
          </div>
        )}
      </div>
      
      <input
        value={input}
        onChange={(e) => setInput(e.target.value)}
        onKeyPress={(e) => e.key === 'Enter' && sendMessage()}
        placeholder="Type your message..."
        disabled={isStreaming}
      />
      <button onClick={sendMessage} disabled={isStreaming}>
        Send
      </button>
    </div>
  );
}
```

## Key Concepts

### 1. Server-Sent Events (SSE)
- One-way communication from server to client
- Uses `text/event-stream` content type
- Data sent as `data: {json}\n\n` format

### 2. ReadableStream API
- Modern browser API for handling streams
- `getReader()` returns a reader to consume chunks
- `read()` returns `{ done, value }` objects

### 3. Progressive UI Updates
- Build response text chunk by chunk
- Show cursor during streaming
- Move to messages when complete

## Error Handling

```jsx
const sendMessage = async () => {
  try {
    const response = await fetch('/api/chat/stream', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ message: input }),
    });

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`);
    }

    // ... streaming logic ...

  } catch (error) {
    console.error('Streaming error:', error);
    setMessages(prev => [...prev, { 
      role: 'assistant', 
      content: 'Sorry, there was an error. Please try again.' 
    }]);
    setIsStreaming(false);
  }
};
```

## Interview Q&A

**Q: Why stream LLM responses instead of waiting for complete response?**

**A:** Better user experience - users see responses immediately and can start reading while AI continues generating, reducing perceived wait time.

**Q: How does SSE streaming work?**

**A:** Server sends data as events with "data: {json}" format. Client reads stream chunks using ReadableStream API and updates UI progressively.

**Q: What if the network connection fails during streaming?**

**A:** Handle errors gracefully, show partial response received, and allow users to retry. Consider implementing reconnection logic for better UX.

**Q: How do you prevent multiple concurrent streams?**

**A:** Disable input/send button during streaming, or use loading state to prevent new requests until current one completes.

**Q: What's the difference between SSE and WebSocket for streaming?**

**A:** SSE is simpler (one-way, automatic reconnection) but WebSocket is bidirectional and better for real-time collaboration.

## Alternative Approaches

### WebSocket (Bidirectional)
- Better for real-time features
- More complex server setup
- Can send data both ways

### Polling (Fallback)
- Check server periodically
- Works everywhere but less efficient
- Higher latency

## Interview Tips
- **Explain the flow**: Server streams → Client reads chunks → UI updates progressively
- **Mention UX benefits**: Immediate feedback, perceived speed, engagement
- **Discuss error handling**: Network failures, partial responses, user feedback
- **Performance considerations**: Debouncing updates, memory usage
- **Browser support**: SSE works in modern browsers, polling as fallback
- **Real implementations**: Share specific projects where you built streaming</content>
<parameter name="filePath">c:\Users\Angshuman\Desktop\MyScripts\QuickReadyInterview\QuickPrepFrontend\11-GenAI-Agents\stream-llm-responses-ui-simple.md