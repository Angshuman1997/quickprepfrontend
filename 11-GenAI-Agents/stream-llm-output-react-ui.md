# How would you stream LLM output into a React UI?

## Question
How would you stream LLM output into a React UI?

# How would you stream LLM output into a React UI?

## Question
How would you stream LLM output into a React UI?

## Answer

Streaming LLM output in React provides a better user experience by showing responses as they're generated, rather than waiting for the complete response. Here are several approaches with complete implementations:

## 1. Server-Sent Events (SSE) Approach

### Backend Implementation
```javascript
// Next.js API route with SSE
// app/api/chat/stream/route.js
import { openai } from '@/lib/openai';
import { NextResponse } from 'next/server';

export async function POST(request) {
  const { message } = await request.json();

  try {
    const stream = await openai.chat.completions.create({
      model: "gpt-4",
      messages: [{ role: "user", content: message }],
      stream: true,
      max_tokens: 1000,
      temperature: 0.7,
    });

    // Create ReadableStream for SSE
    const readableStream = new ReadableStream({
      async start(controller) {
        try {
          for await (const chunk of stream) {
            const content = chunk.choices[0]?.delta?.content;
            
            if (content) {
              // Send content chunk as SSE
              const data = `data: ${JSON.stringify({ 
                content, 
                done: false 
              })}\n\n`;
              controller.enqueue(data);
              
              // Small delay to prevent overwhelming the client
              await new Promise(resolve => setTimeout(resolve, 10));
            }
          }
          
          // Send completion signal
          controller.enqueue(`data: ${JSON.stringify({ 
            content: '', 
            done: true 
          })}\n\n`);
          
        } catch (error) {
          controller.enqueue(`data: ${JSON.stringify({ 
            error: 'Stream failed', 
            done: true 
          })}\n\n`);
        }
        
        controller.close();
      },
    });

    return new Response(readableStream, {
      headers: {
        'Content-Type': 'text/event-stream',
        'Cache-Control': 'no-cache',
        'Connection': 'keep-alive',
        'Access-Control-Allow-Origin': '*',
        'Access-Control-Allow-Headers': 'Cache-Control',
      },
    });
    
  } catch (error) {
    return NextResponse.json(
      { error: 'Failed to start stream' },
      { status: 500 }
    );
  }
}
```

### React Client Implementation
```jsx
// components/StreamingChat.js
import { useState, useRef, useEffect } from 'react';

export default function StreamingChat() {
  const [messages, setMessages] = useState([]);
  const [currentMessage, setCurrentMessage] = useState('');
  const [input, setInput] = useState('');
  const [isStreaming, setIsStreaming] = useState(false);
  const messagesEndRef = useRef(null);

  const scrollToBottom = () => {
    messagesEndRef.current?.scrollIntoView({ behavior: "smooth" });
  };

  useEffect(scrollToBottom, [messages, currentMessage]);

  const sendMessage = async () => {
    if (!input.trim() || isStreaming) return;

    const userMessage = { role: 'user', content: input };
    setMessages(prev => [...prev, userMessage]);
    setInput('');
    setIsStreaming(true);
    setCurrentMessage('');

    try {
      const response = await fetch('/api/chat/stream', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ message: input }),
      });

      if (!response.ok) {
        throw new Error('Stream failed');
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
            
            try {
              const parsed = JSON.parse(data);
              
              if (parsed.error) {
                setMessages(prev => [...prev, { 
                  role: 'assistant', 
                  content: `Error: ${parsed.error}` 
                }]);
                setIsStreaming(false);
                return;
              }
              
              if (parsed.done) {
                setIsStreaming(false);
                setMessages(prev => [...prev, { 
                  role: 'assistant', 
                  content: currentMessage 
                }]);
                setCurrentMessage('');
                return;
              }
              
              if (parsed.content) {
                setCurrentMessage(prev => prev + parsed.content);
              }
              
            } catch (parseError) {
              console.error('Parse error:', parseError);
            }
          }
        }
      }
      
    } catch (error) {
      console.error('Streaming error:', error);
      setMessages(prev => [...prev, { 
        role: 'assistant', 
        content: 'Sorry, there was an error processing your request.' 
      }]);
      setIsStreaming(false);
    }
  };

  return (
    <div className="streaming-chat">
      <div className="messages-container">
        {messages.map((msg, idx) => (
          <div key={idx} className={`message ${msg.role}`}>
            <div className="content">{msg.content}</div>
          </div>
        ))}
        
        {isStreaming && (
          <div className="message assistant streaming">
            <div className="content">
              {currentMessage}
              <span className="cursor">|</span>
            </div>
          </div>
        )}
        
        <div ref={messagesEndRef} />
      </div>
      
      <div className="input-container">
        <input
          value={input}
          onChange={(e) => setInput(e.target.value)}
          onKeyPress={(e) => e.key === 'Enter' && !e.shiftKey && sendMessage()}
          placeholder="Type your message..."
          disabled={isStreaming}
        />
        <button 
          onClick={sendMessage} 
          disabled={isStreaming || !input.trim()}
        >
          {isStreaming ? 'Streaming...' : 'Send'}
        </button>
      </div>
    </div>
  );
}
```

## 2. WebSocket Approach

### WebSocket Server Implementation
```javascript
// WebSocket server (using ws library)
// lib/websocket-server.js
import WebSocket from 'ws';
import { openai } from './openai';

export function setupWebSocketServer(server) {
  const wss = new WebSocket.Server({ server });

  wss.on('connection', (ws) => {
    console.log('Client connected');

    ws.on('message', async (data) => {
      try {
        const { message, sessionId } = JSON.parse(data.toString());
        
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
        
        // Send completion
        ws.send(JSON.stringify({
          type: 'done',
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

    ws.on('close', () => {
      console.log('Client disconnected');
    });
  });

  return wss;
}
```

### React WebSocket Client
```jsx
// components/WebSocketChat.js
import { useState, useRef, useEffect } from 'react';

export default function WebSocketChat() {
  const [messages, setMessages] = useState([]);
  const [currentMessage, setCurrentMessage] = useState('');
  const [input, setInput] = useState('');
  const [isConnected, setIsConnected] = useState(false);
  const [isStreaming, setIsStreaming] = useState(false);
  const wsRef = useRef(null);
  const sessionIdRef = useRef(Date.now().toString());

  useEffect(() => {
    // Connect to WebSocket
    const ws = new WebSocket('ws://localhost:8080');
    
    ws.onopen = () => {
      console.log('Connected to WebSocket');
      setIsConnected(true);
    };
    
    ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      
      if (data.sessionId !== sessionIdRef.current) return;
      
      switch (data.type) {
        case 'chunk':
          setCurrentMessage(prev => prev + data.content);
          break;
        case 'done':
          setIsStreaming(false);
          setMessages(prev => [...prev, { 
            role: 'assistant', 
            content: currentMessage 
          }]);
          setCurrentMessage('');
          break;
        case 'error':
          setMessages(prev => [...prev, { 
            role: 'assistant', 
            content: `Error: ${data.error}` 
          }]);
          setIsStreaming(false);
          break;
      }
    };
    
    ws.onclose = () => {
      console.log('WebSocket disconnected');
      setIsConnected(false);
    };
    
    wsRef.current = ws;
    
    return () => {
      ws.close();
    };
  }, []);

  const sendMessage = () => {
    if (!input.trim() || !isConnected || isStreaming) return;
    
    const userMessage = { role: 'user', content: input };
    setMessages(prev => [...prev, userMessage]);
    setInput('');
    setIsStreaming(true);
    setCurrentMessage('');
    
    sessionIdRef.current = Date.now().toString();
    
    wsRef.current.send(JSON.stringify({
      message: input,
      sessionId: sessionIdRef.current
    }));
  };

  return (
    <div className="websocket-chat">
      <div className="connection-status">
        Status: {isConnected ? '🟢 Connected' : '🔴 Disconnected'}
      </div>
      
      <div className="messages-container">
        {messages.map((msg, idx) => (
          <div key={idx} className={`message ${msg.role}`}>
            {msg.content}
          </div>
        ))}
        
        {isStreaming && (
          <div className="message assistant streaming">
            {currentMessage}
            <span className="cursor">|</span>
          </div>
        )}
      </div>
      
      <div className="input-container">
        <input
          value={input}
          onChange={(e) => setInput(e.target.value)}
          onKeyPress={(e) => e.key === 'Enter' && sendMessage()}
          placeholder="Type your message..."
          disabled={!isConnected || isStreaming}
        />
        <button onClick={sendMessage} disabled={!isConnected || isStreaming}>
          Send
        </button>
      </div>
    </div>
  );
}
```

## 3. Polling Approach (Fallback)

### When to Use Polling
```javascript
// Use polling when SSE/WebSocket not available
// Useful for environments that don't support streaming
const useStreamingFallback = () => {
  const [isSupported, setIsSupported] = useState(true);
  
  useEffect(() => {
    // Check if EventSource is supported
    setIsSupported(typeof EventSource !== 'undefined');
  }, []);
  
  return !isSupported;
};
```

### Polling Implementation
```jsx
// components/PollingChat.js
import { useState, useEffect } from 'react';

export default function PollingChat() {
  const [messages, setMessages] = useState([]);
  const [currentMessage, setCurrentMessage] = useState('');
  const [input, setInput] = useState('');
  const [isPolling, setIsPolling] = useState(false);
  const [requestId, setRequestId] = useState(null);

  const sendMessage = async () => {
    if (!input.trim() || isPolling) return;

    const userMessage = { role: 'user', content: input };
    setMessages(prev => [...prev, userMessage]);
    setInput('');
    setIsPolling(true);
    setCurrentMessage('');

    try {
      // Start the streaming request
      const response = await fetch('/api/chat/start', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ message: input }),
      });
      
      const { requestId: newRequestId } = await response.json();
      setRequestId(newRequestId);
      
      // Start polling for updates
      pollForUpdates(newRequestId);
      
    } catch (error) {
      setMessages(prev => [...prev, { 
        role: 'assistant', 
        content: 'Error starting request' 
      }]);
      setIsPolling(false);
    }
  };

  const pollForUpdates = async (reqId) => {
    const pollInterval = setInterval(async () => {
      try {
        const response = await fetch(`/api/chat/status/${reqId}`);
        const data = await response.json();
        
        if (data.status === 'completed') {
          clearInterval(pollInterval);
          setMessages(prev => [...prev, { 
            role: 'assistant', 
            content: data.fullResponse 
          }]);
          setIsPolling(false);
          setRequestId(null);
        } else if (data.status === 'streaming') {
          setCurrentMessage(data.partialResponse);
        }
        
      } catch (error) {
        clearInterval(pollInterval);
        setMessages(prev => [...prev, { 
          role: 'assistant', 
          content: 'Error during polling' 
        }]);
        setIsPolling(false);
      }
    }, 500); // Poll every 500ms
  };

  return (
    <div className="polling-chat">
      <div className="messages-container">
        {messages.map((msg, idx) => (
          <div key={idx} className={`message ${msg.role}`}>
            {msg.content}
          </div>
        ))}
        
        {isPolling && (
          <div className="message assistant streaming">
            {currentMessage}
            <span className="cursor">|</span>
          </div>
        )}
      </div>
      
      <div className="input-container">
        <input
          value={input}
          onChange={(e) => setInput(e.target.value)}
          onKeyPress={(e) => e.key === 'Enter' && sendMessage()}
          placeholder="Type your message..."
          disabled={isPolling}
        />
        <button onClick={sendMessage} disabled={isPolling}>
          Send
        </button>
      </div>
    </div>
  );
}
```

## 4. Advanced Streaming with React Hooks

### Custom Streaming Hook
```javascript
// hooks/useStreamingChat.js
import { useState, useCallback, useRef } from 'react';

export function useStreamingChat(apiEndpoint = '/api/chat/stream') {
  const [messages, setMessages] = useState([]);
  const [currentMessage, setCurrentMessage] = useState('');
  const [isStreaming, setIsStreaming] = useState(false);
  const [error, setError] = useState(null);
  const abortControllerRef = useRef(null);

  const sendMessage = useCallback(async (message) => {
    if (isStreaming) return;
    
    setIsStreaming(true);
    setError(null);
    setCurrentMessage('');
    
    // Add user message
    const userMessage = { role: 'user', content: message, timestamp: Date.now() };
    setMessages(prev => [...prev, userMessage]);
    
    // Create abort controller for cancellation
    abortControllerRef.current = new AbortController();
    
    try {
      const response = await fetch(apiEndpoint, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ message }),
        signal: abortControllerRef.current.signal,
      });

      if (!response.ok) {
        throw new Error(`HTTP ${response.status}`);
      }

      const reader = response.body.getReader();
      const decoder = new TextDecoder();
      let buffer = '';

      while (true) {
        const { done, value } = await reader.read();
        if (done) break;

        buffer += decoder.decode(value, { stream: true });
        const lines = buffer.split('\n');
        buffer = lines.pop() || ''; // Keep incomplete line in buffer

        for (const line of lines) {
          if (line.startsWith('data: ')) {
            const data = line.slice(6);
            
            try {
              const parsed = JSON.parse(data);
              
              if (parsed.error) {
                throw new Error(parsed.error);
              }
              
              if (parsed.done) {
                setMessages(prev => [...prev, { 
                  role: 'assistant', 
                  content: currentMessage,
                  timestamp: Date.now()
                }]);
                setCurrentMessage('');
                setIsStreaming(false);
                return;
              }
              
              if (parsed.content) {
                setCurrentMessage(prev => prev + parsed.content);
              }
              
            } catch (parseError) {
              console.error('Parse error:', parseError);
            }
          }
        }
      }
      
    } catch (err) {
      if (err.name === 'AbortError') {
        setError('Request cancelled');
      } else {
        setError(err.message);
        setMessages(prev => [...prev, { 
          role: 'assistant', 
          content: `Error: ${err.message}`,
          timestamp: Date.now()
        }]);
      }
    } finally {
      setIsStreaming(false);
      abortControllerRef.current = null;
    }
  }, [isStreaming, currentMessage]);

  const cancelStream = useCallback(() => {
    if (abortControllerRef.current) {
      abortControllerRef.current.abort();
    }
  }, []);

  const clearMessages = useCallback(() => {
    setMessages([]);
    setCurrentMessage('');
    setError(null);
  }, []);

  return {
    messages,
    currentMessage,
    isStreaming,
    error,
    sendMessage,
    cancelStream,
    clearMessages
  };
}
```

### Using the Custom Hook
```jsx
// components/ChatWithHook.js
import { useStreamingChat } from '@/hooks/useStreamingChat';

export default function ChatWithHook() {
  const {
    messages,
    currentMessage,
    isStreaming,
    error,
    sendMessage,
    cancelStream,
    clearMessages
  } = useStreamingChat();

  const [input, setInput] = useState('');

  const handleSubmit = (e) => {
    e.preventDefault();
    if (input.trim()) {
      sendMessage(input);
      setInput('');
    }
  };

  return (
    <div className="chat-with-hook">
      <div className="controls">
        <button onClick={clearMessages} disabled={isStreaming}>
          Clear Chat
        </button>
        {isStreaming && (
          <button onClick={cancelStream}>
            Cancel
          </button>
        )}
      </div>
      
      {error && (
        <div className="error">
          Error: {error}
        </div>
      )}
      
      <div className="messages">
        {messages.map((msg, idx) => (
          <div key={idx} className={`message ${msg.role}`}>
            <div className="content">{msg.content}</div>
            <div className="timestamp">
              {new Date(msg.timestamp).toLocaleTimeString()}
            </div>
          </div>
        ))}
        
        {isStreaming && (
          <div className="message assistant streaming">
            <div className="content">
              {currentMessage}
              <span className="cursor">|</span>
            </div>
          </div>
        )}
      </div>
      
      <form onSubmit={handleSubmit}>
        <div className="input-group">
          <input
            value={input}
            onChange={(e) => setInput(e.target.value)}
            placeholder="Type your message..."
            disabled={isStreaming}
          />
          <button type="submit" disabled={isStreaming || !input.trim()}>
            {isStreaming ? 'Streaming...' : 'Send'}
          </button>
        </div>
      </form>
    </div>
  );
}
```

## 5. Error Handling & Recovery

### Robust Error Handling
```jsx
// Enhanced streaming with error recovery
export function useResilientStreaming(apiEndpoint) {
  const [retryCount, setRetryCount] = useState(0);
  const maxRetries = 3;
  
  const sendMessage = async (message) => {
    for (let attempt = 0; attempt <= maxRetries; attempt++) {
      try {
        setRetryCount(attempt);
        const result = await attemptStream(message);
        setRetryCount(0);
        return result;
      } catch (error) {
        if (attempt === maxRetries) {
          throw error;
        }
        
        // Exponential backoff
        await new Promise(resolve => 
          setTimeout(resolve, Math.pow(2, attempt) * 1000)
        );
      }
    }
  };
  
  return { sendMessage, retryCount };
}
```

## Performance Optimizations

### 1. **Debounced Streaming**
```javascript
// Prevent too frequent updates
import { useCallback, useRef } from 'react';

export function useDebouncedStreaming() {
  const timeoutRef = useRef(null);
  const bufferRef = useRef('');
  
  const updateStream = useCallback((newContent) => {
    bufferRef.current += newContent;
    
    if (timeoutRef.current) {
      clearTimeout(timeoutRef.current);
    }
    
    timeoutRef.current = setTimeout(() => {
      setCurrentMessage(prev => prev + bufferRef.current);
      bufferRef.current = '';
    }, 50); // Update every 50ms
  }, []);
  
  return updateStream;
}
```

### 2. **Virtual Scrolling for Long Conversations**
```jsx
// For very long conversations
import { FixedSizeList as List } from 'react-window';

export function VirtualizedChat({ messages, currentMessage }) {
  const allMessages = [
    ...messages,
    currentMessage ? { role: 'assistant', content: currentMessage } : null
  ].filter(Boolean);
  
  return (
    <List
      height={400}
      itemCount={allMessages.length}
      itemSize={60}
      width="100%"
    >
      {({ index, style }) => (
        <div style={style}>
          <MessageComponent message={allMessages[index]} />
        </div>
      )}
    </List>
  );
}
```

## CSS Styling for Streaming Effects

```css
/* Streaming chat styles */
.streaming-chat {
  max-width: 800px;
  margin: 0 auto;
  height: 600px;
  display: flex;
  flex-direction: column;
}

.messages-container {
  flex: 1;
  overflow-y: auto;
  padding: 1rem;
  border: 1px solid #ddd;
  border-radius: 8px;
  margin-bottom: 1rem;
}

.message {
  margin-bottom: 1rem;
  padding: 0.75rem;
  border-radius: 8px;
}

.message.user {
  background: #007bff;
  color: white;
  margin-left: 2rem;
}

.message.assistant {
  background: #f8f9fa;
  margin-right: 2rem;
}

.message.streaming {
  border: 2px solid #28a745;
  animation: pulse 2s infinite;
}

.cursor {
  animation: blink 1s infinite;
}

@keyframes blink {
  0%, 50% { opacity: 1; }
  51%, 100% { opacity: 0; }
}

@keyframes pulse {
  0% { border-color: #28a745; }
  50% { border-color: #20c997; }
  100% { border-color: #28a745; }
}

.input-container {
  display: flex;
  gap: 0.5rem;
}

.input-container input {
  flex: 1;
  padding: 0.75rem;
  border: 1px solid #ddd;
  border-radius: 4px;
}

.input-container button {
  padding: 0.75rem 1.5rem;
  background: #007bff;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.input-container button:disabled {
  background: #6c757d;
  cursor: not-allowed;
}
```

## Interview Tips
- **Explain trade-offs**: Discuss SSE vs WebSocket vs polling
- **Performance considerations**: Mention debouncing, virtual scrolling
- **Error handling**: Show robust error recovery and user feedback
- **User experience**: Talk about loading states, cancellation, smooth scrolling
- **Real implementations**: Share specific projects where you implemented streaming
- **Browser compatibility**: Mention fallbacks for older browsers
- **Scalability**: Discuss connection limits, server resources