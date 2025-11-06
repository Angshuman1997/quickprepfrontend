# How to stream LLM responses in UI?

## Question
How to stream LLM responses in UI?

# How to stream LLM responses in UI?

## Question
How to stream LLM responses in UI?

## Answer

Streaming LLM responses creates a more engaging user experience by displaying text as it's generated. Here are the main approaches and implementation patterns:

## 1. Server-Sent Events (SSE) with React

### Backend API (Next.js)
```javascript
// app/api/chat/stream/route.js
import { openai } from '@/lib/openai';
import { NextResponse } from 'next/server';

export async function POST(request) {
  const { message } = await request.json();

  const stream = await openai.chat.completions.create({
    model: "gpt-4",
    messages: [{ role: "user", content: message }],
    stream: true,
  });

  const encoder = new TextEncoder();

  const readableStream = new ReadableStream({
    async start(controller) {
      for await (const chunk of stream) {
        const content = chunk.choices[0]?.delta?.content || '';
        
        if (content) {
          const data = `data: ${JSON.stringify({ content })}\n\n`;
          controller.enqueue(encoder.encode(data));
        }
      }
      
      controller.enqueue(encoder.encode(`data: [DONE]\n\n`));
      controller.close();
    },
  });

  return new Response(readableStream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive',
    },
  });
}
```

### React Streaming Component
```jsx
// components/StreamingResponse.js
import { useState, useEffect, useRef } from 'react';

export default function StreamingResponse({ message, onComplete }) {
  const [displayedText, setDisplayedText] = useState('');
  const [isComplete, setIsComplete] = useState(false);
  const eventSourceRef = useRef(null);

  useEffect(() => {
    if (!message) return;

    setDisplayedText('');
    setIsComplete(false);

    const eventSource = new EventSource('/api/chat/stream');
    eventSourceRef.current = eventSource;

    eventSource.onmessage = (event) => {
      const data = JSON.parse(event.data);
      
      if (data.content) {
        setDisplayedText(prev => prev + data.content);
      } else if (event.data === '[DONE]') {
        setIsComplete(true);
        onComplete?.(displayedText);
        eventSource.close();
      }
    };

    eventSource.onerror = (error) => {
      console.error('SSE Error:', error);
      setIsComplete(true);
      eventSource.close();
    };

    // Send the message
    fetch('/api/chat/stream', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ message }),
    });

    return () => {
      eventSource.close();
    };
  }, [message, onComplete]);

  return (
    <div className="streaming-response">
      <div className="response-text">
        {displayedText}
        {!isComplete && <span className="cursor">|</span>}
      </div>
      {isComplete && (
        <div className="completion-indicator">✓</div>
      )}
    </div>
  );
}
```

## 2. WebSocket Implementation

### WebSocket Server
```javascript
// lib/websocket-handler.js
import { WebSocketServer } from 'ws';
import { openai } from './openai';

export function handleWebSocketConnection(ws, request) {
  ws.on('message', async (data) => {
    const { message, sessionId } = JSON.parse(data.toString());
    
    try {
      const stream = await openai.chat.completions.create({
        model: "gpt-4",
        messages: [{ role: "user", content: message }],
        stream: true,
      });

      for await (const chunk of stream) {
        const content = chunk.choices[0]?.delta?.content;
        
        if (content) {
          ws.send(JSON.stringify({
            type: 'chunk',
            content,
            sessionId
          }));
        }
      }
      
      ws.send(JSON.stringify({
        type: 'complete',
        sessionId
      }));
      
    } catch (error) {
      ws.send(JSON.stringify({
        type: 'error',
        error: error.message,
        sessionId
      }));
    }
  });
}
```

### React WebSocket Client
```jsx
// components/WebSocketStreaming.js
import { useState, useEffect, useRef } from 'react';

export default function WebSocketStreaming() {
  const [response, setResponse] = useState('');
  const [isStreaming, setIsStreaming] = useState(false);
  const wsRef = useRef(null);
  const sessionRef = useRef(null);

  useEffect(() => {
    const ws = new WebSocket('ws://localhost:8080');
    
    ws.onopen = () => console.log('WebSocket connected');
    ws.onclose = () => console.log('WebSocket disconnected');
    
    ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      
      if (data.sessionId !== sessionRef.current) return;
      
      switch (data.type) {
        case 'chunk':
          setResponse(prev => prev + data.content);
          break;
        case 'complete':
          setIsStreaming(false);
          break;
        case 'error':
          console.error('Streaming error:', data.error);
          setIsStreaming(false);
          break;
      }
    };
    
    wsRef.current = ws;
    
    return () => ws.close();
  }, []);

  const sendMessage = (message) => {
    if (!wsRef.current || wsRef.current.readyState !== WebSocket.OPEN) return;
    
    sessionRef.current = Date.now().toString();
    setResponse('');
    setIsStreaming(true);
    
    wsRef.current.send(JSON.stringify({
      message,
      sessionId: sessionRef.current
    }));
  };

  return (
    <div className="websocket-streaming">
      <button onClick={() => sendMessage("Hello, how are you?")}>
        Send Message
      </button>
      
      <div className="response-area">
        {response}
        {isStreaming && <span className="cursor">|</span>}
      </div>
    </div>
  );
}
```

## 3. Custom Hook for Streaming

### Reusable Streaming Hook
```javascript
// hooks/useLLMStream.js
import { useState, useCallback, useRef } from 'react';

export function useLLMStream(endpoint = '/api/stream') {
  const [text, setText] = useState('');
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState(null);
  const abortControllerRef = useRef(null);

  const streamResponse = useCallback(async (payload) => {
    setIsLoading(true);
    setError(null);
    setText('');
    
    abortControllerRef.current = new AbortController();
    
    try {
      const response = await fetch(endpoint, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload),
        signal: abortControllerRef.current.signal,
      });

      if (!response.ok) {
        throw new Error(`HTTP ${response.status}`);
      }

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
            
            if (data === '[DONE]') {
              setIsLoading(false);
              return;
            }
            
            try {
              const parsed = JSON.parse(data);
              if (parsed.content) {
                setText(prev => prev + parsed.content);
              }
              if (parsed.error) {
                throw new Error(parsed.error);
              }
            } catch (e) {
              // Handle parse errors
            }
          }
        }
      }
      
    } catch (err) {
      if (err.name !== 'AbortError') {
        setError(err.message);
      }
    } finally {
      setIsLoading(false);
    }
  }, [endpoint]);

  const cancel = useCallback(() => {
    if (abortControllerRef.current) {
      abortControllerRef.current.abort();
    }
  }, []);

  const reset = useCallback(() => {
    setText('');
    setError(null);
    cancel();
  }, [cancel]);

  return {
    text,
    isLoading,
    error,
    streamResponse,
    cancel,
    reset
  };
}
```

### Using the Hook
```jsx
// components/StreamingChat.js
import { useState } from 'react';
import { useLLMStream } from '@/hooks/useLLMStream';

export default function StreamingChat() {
  const [input, setInput] = useState('');
  const { text, isLoading, error, streamResponse, cancel } = useLLMStream();

  const handleSubmit = async (e) => {
    e.preventDefault();
    if (!input.trim() || isLoading) return;
    
    await streamResponse({ message: input });
    setInput('');
  };

  return (
    <div className="streaming-chat">
      <form onSubmit={handleSubmit}>
        <input
          value={input}
          onChange={(e) => setInput(e.target.value)}
          placeholder="Ask me anything..."
          disabled={isLoading}
        />
        <button type="submit" disabled={isLoading}>
          {isLoading ? 'Streaming...' : 'Send'}
        </button>
        {isLoading && (
          <button type="button" onClick={cancel}>
            Cancel
          </button>
        )}
      </form>
      
      {error && <div className="error">Error: {error}</div>}
      
      <div className="response">
        {text}
        {isLoading && <span className="cursor">|</span>}
      </div>
    </div>
  );
}
```

## 4. Progressive Enhancement with Fallbacks

### Smart Streaming Component
```jsx
// components/SmartStreaming.js
import { useState, useEffect } from 'react';

export default function SmartStreaming({ message }) {
  const [response, setResponse] = useState('');
  const [isComplete, setIsComplete] = useState(false);
  const [method, setMethod] = useState('detecting');

  useEffect(() => {
    if (!message) return;

    setResponse('');
    setIsComplete(false);

    // Detect best streaming method
    if (typeof WebSocket !== 'undefined') {
      setMethod('websocket');
      startWebSocketStream(message);
    } else if (typeof EventSource !== 'undefined') {
      setMethod('sse');
      startSSEStream(message);
    } else {
      setMethod('polling');
      startPollingStream(message);
    }
  }, [message]);

  const startWebSocketStream = (msg) => {
    // WebSocket implementation
  };

  const startSSEStream = (msg) => {
    // SSE implementation
  };

  const startPollingStream = async (msg) => {
    // Polling fallback
    const response = await fetch('/api/chat', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ message: msg }),
    });
    
    const data = await response.json();
    setResponse(data.response);
    setIsComplete(true);
  };

  return (
    <div className="smart-streaming">
      <div className="method-indicator">
        Using: {method}
      </div>
      
      <div className="response-text">
        {response}
        {!isComplete && <span className="cursor">|</span>}
      </div>
      
      {isComplete && <div className="complete">✓ Complete</div>}
    </div>
  );
}
```

## 5. Advanced UI Patterns

### Typing Effect with Variable Speed
```jsx
// components/TypingEffect.js
import { useState, useEffect } from 'react';

export default function TypingEffect({ text, speed = 50, onComplete }) {
  const [displayedText, setDisplayedText] = useState('');
  const [currentIndex, setCurrentIndex] = useState(0);

  useEffect(() => {
    if (!text) return;

    setDisplayedText('');
    setCurrentIndex(0);

    const timer = setInterval(() => {
      setCurrentIndex(prev => {
        const next = prev + 1;
        
        if (next <= text.length) {
          setDisplayedText(text.slice(0, next));
          return next;
        } else {
          clearInterval(timer);
          onComplete?.();
          return prev;
        }
      });
    }, speed);

    return () => clearInterval(timer);
  }, [text, speed, onComplete]);

  return (
    <div className="typing-effect">
      {displayedText}
      {currentIndex < text.length && <span className="cursor">|</span>}
    </div>
  );
}
```

### Markdown Streaming
```jsx
// components/MarkdownStreamer.js
import { useState, useEffect } from 'react';
import ReactMarkdown from 'react-markdown';

export default function MarkdownStreamer({ markdownText }) {
  const [renderedHTML, setRenderedHTML] = useState('');
  const [isComplete, setIsComplete] = useState(false);

  useEffect(() => {
    if (!markdownText) return;

    setRenderedHTML('');
    setIsComplete(false);

    let currentIndex = 0;
    const timer = setInterval(() => {
      currentIndex += 1;
      
      if (currentIndex <= markdownText.length) {
        const partialMarkdown = markdownText.slice(0, currentIndex);
        setRenderedHTML(partialMarkdown);
      } else {
        clearInterval(timer);
        setIsComplete(true);
      }
    }, 20);

    return () => clearInterval(timer);
  }, [markdownText]);

  return (
    <div className="markdown-streamer">
      <ReactMarkdown>{renderedHTML}</ReactMarkdown>
      {!isComplete && <span className="cursor">|</span>}
    </div>
  );
}
```

## 6. Error Handling & Recovery

### Resilient Streaming
```jsx
// components/ResilientStreamer.js
import { useState, useCallback } from 'react';

export default function ResilientStreamer({ message }) {
  const [response, setResponse] = useState('');
  const [error, setError] = useState(null);
  const [retryCount, setRetryCount] = useState(0);
  const maxRetries = 3;

  const attemptStream = useCallback(async (attemptNumber = 0) => {
    try {
      setError(null);
      setRetryCount(attemptNumber);
      
      const result = await streamFromAPI(message);
      setResponse(result);
      
    } catch (err) {
      if (attemptNumber < maxRetries) {
        // Exponential backoff
        setTimeout(() => {
          attemptStream(attemptNumber + 1);
        }, Math.pow(2, attemptNumber) * 1000);
      } else {
        setError(`Failed after ${maxRetries + 1} attempts: ${err.message}`);
      }
    }
  }, [message]);

  useEffect(() => {
    if (message) {
      setResponse('');
      attemptStream();
    }
  }, [message, attemptStream]);

  return (
    <div className="resilient-streamer">
      {error ? (
        <div className="error-display">
          <p>Error: {error}</p>
          <button onClick={() => attemptStream()}>
            Retry
          </button>
        </div>
      ) : (
        <div className="response">
          {response}
          {retryCount > 0 && (
            <div className="retry-indicator">
              Retry attempt: {retryCount}
            </div>
          )}
        </div>
      )}
    </div>
  );
}
```

## 7. Performance Optimizations

### Debounced Updates
```javascript
// hooks/useDebouncedStream.js
import { useState, useEffect, useRef } from 'react';

export function useDebouncedStream(initialText = '') {
  const [displayText, setDisplayText] = useState(initialText);
  const bufferRef = useRef('');
  const timeoutRef = useRef(null);

  const updateStream = (newChunk) => {
    bufferRef.current += newChunk;
    
    if (timeoutRef.current) {
      clearTimeout(timeoutRef.current);
    }
    
    timeoutRef.current = setTimeout(() => {
      setDisplayText(prev => prev + bufferRef.current);
      bufferRef.current = '';
    }, 16); // ~60fps
  };

  const finalize = () => {
    if (timeoutRef.current) {
      clearTimeout(timeoutRef.current);
    }
    if (bufferRef.current) {
      setDisplayText(prev => prev + bufferRef.current);
      bufferRef.current = '';
    }
  };

  return { displayText, updateStream, finalize };
}
```

## CSS Styling

```css
.streaming-response {
  font-family: 'Monaco', 'Menlo', monospace;
  font-size: 16px;
  line-height: 1.5;
  padding: 20px;
  border: 1px solid #e1e5e9;
  border-radius: 8px;
  background: #fafbfc;
  min-height: 100px;
}

.response-text {
  white-space: pre-wrap;
  word-wrap: break-word;
}

.cursor {
  animation: blink 1s infinite;
  color: #0366d6;
  font-weight: bold;
}

@keyframes blink {
  0%, 50% { opacity: 1; }
  51%, 100% { opacity: 0; }
}

.completion-indicator {
  color: #28a745;
  font-size: 18px;
  margin-top: 10px;
}

.error-display {
  color: #dc3545;
  padding: 10px;
  border: 1px solid #f5c6cb;
  border-radius: 4px;
  background: #f8d7da;
}

.retry-indicator {
  color: #ffc107;
  font-size: 12px;
  margin-top: 5px;
}
```

## Key Considerations

### When to Use Each Method
- **SSE**: Best for server-to-client streaming, simple to implement
- **WebSocket**: Ideal for bidirectional communication, real-time collaboration
- **Polling**: Fallback for environments without streaming support
- **Custom Hook**: Reusable across components, clean separation of concerns

### Performance Tips
- Debounce UI updates to prevent excessive re-renders
- Use `React.memo` for streaming components
- Implement virtual scrolling for long conversations
- Cache responses when appropriate
- Handle network interruptions gracefully

### UX Best Practices
- Show clear loading states
- Provide cancellation options
- Handle errors gracefully with retry mechanisms
- Use smooth animations for text appearance
- Consider accessibility (screen readers, reduced motion)

## Interview Tips
- **Explain trade-offs**: SSE vs WebSocket vs polling
- **Performance focus**: Discuss debouncing, memory management
- **Error handling**: Show robust error recovery patterns
- **User experience**: Talk about loading states, smooth animations
- **Real examples**: Share specific implementations you've built
- **Browser compatibility**: Mention fallbacks and progressive enhancement