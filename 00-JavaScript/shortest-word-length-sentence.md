# Write code to return the shortest word length in a sentence

## Question
Write code to return the shortest word length in a sentence.

## Answer

Finding the shortest word length in a sentence involves splitting the sentence into words and finding the minimum word length. This is a common algorithmic problem with several approaches.

## 1. **Using split() and Math.min()**

```javascript
function findShortestWordLength(sentence) {
    if (!sentence || sentence.trim() === '') return 0;

    // Split sentence into words and find minimum length
    const words = sentence.trim().split(/\s+/);
    return Math.min(...words.map(word => word.length));
}

// Examples
const sentence1 = "The quick brown fox jumps over the lazy dog";
console.log(findShortestWordLength(sentence1)); // 3 ("The", "fox", "dog" are 3 chars)

const sentence2 = "I love programming";
console.log(findShortestWordLength(sentence2)); // 1 ("I")

const sentence3 = "Supercalifragilisticexpialidocious is a long word";
console.log(findShortestWordLength(sentence3)); // 2 ("is", "a")
```

## 2. **Using reduce() Method**

```javascript
function findShortestWordLengthReduce(sentence) {
    if (!sentence || sentence.trim() === '') return 0;

    const words = sentence.trim().split(/\s+/);
    return words.reduce((minLength, word) => {
        return Math.min(minLength, word.length);
    }, Infinity);
}

// Example
const sentence = "JavaScript is an amazing language";
console.log(findShortestWordLengthReduce(sentence)); // 2 ("is", "an")
```

## 3. **Step-by-Step Approach**

```javascript
function findShortestWordLengthStepByStep(sentence) {
    // Step 1: Handle edge cases
    if (!sentence || sentence.trim() === '') {
        return 0;
    }

    // Step 2: Split sentence into words
    // Using regex to handle multiple spaces
    const words = sentence.trim().split(/\s+/);

    // Step 3: Initialize minimum length
    let minLength = Infinity;

    // Step 4: Iterate through words to find minimum
    for (const word of words) {
        if (word.length < minLength) {
            minLength = word.length;
        }
    }

    // Step 5: Return result
    return minLength;
}

// Example
const sentence = "The quick brown fox";
console.log(findShortestWordLengthStepByStep(sentence)); // 3
```

## 4. **Advanced Techniques**

### **Finding Shortest Word with Details**

```javascript
function findShortestWordInfo(sentence) {
    if (!sentence || sentence.trim() === '') {
        return { length: 0, word: '', position: -1 };
    }

    const words = sentence.trim().split(/\s+/);
    let shortestWord = words[0];
    let shortestPosition = 0;

    words.forEach((word, index) => {
        if (word.length < shortestWord.length) {
            shortestWord = word;
            shortestPosition = index;
        }
    });

    return {
        length: shortestWord.length,
        word: shortestWord,
        position: shortestPosition
    };
}

// Example
const sentence = "The quick brown fox jumps over the lazy dog";
const result = findShortestWordInfo(sentence);
console.log(result);
// { length: 3, word: 'The', position: 0 }
```

### **Finding All Shortest Words**

```javascript
function findAllShortestWords(sentence) {
    if (!sentence || sentence.trim() === '') return [];

    const words = sentence.trim().split(/\s+/);
    const minLength = Math.min(...words.map(word => word.length));

    return words.filter(word => word.length === minLength);
}

// Example
const sentence = "I am a developer and I love coding";
console.log(findAllShortestWords(sentence)); // ['I', 'a', 'I'] (all length 1)
```

### **Handling Punctuation and Special Cases**

```javascript
function findShortestWordLengthRobust(sentence) {
    if (!sentence || typeof sentence !== 'string') return 0;

    // Remove punctuation and split into words
    const words = sentence
        .replace(/[^\w\s]/g, '') // Remove punctuation
        .trim()
        .split(/\s+/)
        .filter(word => word.length > 0); // Remove empty strings

    if (words.length === 0) return 0;

    return Math.min(...words.map(word => word.length));
}

// Examples
console.log(findShortestWordLengthRobust("Hello, world!")); // 5 ("Hello", "world")
console.log(findShortestWordLengthRobust("I can't do it.")); // 1 ("I")
console.log(findShortestWordLengthRobust("A, B, C!!!")); // 1 ("A", "B", "C")
```

### **Case-Insensitive Word Length**

```javascript
function findShortestWordLengthCaseInsensitive(sentence) {
    if (!sentence || sentence.trim() === '') return 0;

    // Convert to lowercase for processing, but preserve original words
    const words = sentence.trim().split(/\s+/);
    const wordObjects = words.map(word => ({
        original: word,
        length: word.toLowerCase().length
    }));

    const minLength = Math.min(...wordObjects.map(w => w.length));
    const shortestWords = wordObjects.filter(w => w.length === minLength);

    return {
        length: minLength,
        words: shortestWords.map(w => w.original)
    };
}

// Example
const sentence = "The Quick Brown FOX";
const result = findShortestWordLengthCaseInsensitive(sentence);
console.log(result);
// { length: 3, words: ['The', 'FOX'] }
```

## Real React Examples:

### 1. **Sentence Analyzer Component**

```javascript
function SentenceAnalyzer({ sentence }) {
    const analyzeSentence = (text) => {
        if (!text || text.trim() === '') {
            return { wordCount: 0, shortestLength: 0, shortestWords: [] };
        }

        const words = text.toLowerCase().split(/\s+/);
        const lengths = words.map(word => word.length);
        const shortestLength = Math.min(...lengths);
        const shortestWords = words.filter(word => word.length === shortestLength);

        return {
            wordCount: words.length,
            shortestLength,
            shortestWords
        };
    };

    const analysis = analyzeSentence(sentence);

    return (
        <div className="sentence-analyzer">
            <h3>Sentence Analysis</h3>
            <p><strong>Original:</strong> {sentence}</p>
            <div className="stats">
                <p>Word count: {analysis.wordCount}</p>
                <p>Shortest word length: {analysis.shortestLength}</p>
                <p>Shortest words: {analysis.shortestWords.join(', ')}</p>
            </div>
        </div>
    );
}

// Usage
<SentenceAnalyzer sentence="The quick brown fox jumps over the lazy dog" />
```

### 2. **Text Input Validator**

```javascript
function TextValidator({ value, minWordLength = 3 }) {
    const [error, setError] = useState('');

    useEffect(() => {
        if (!value) return;

        const words = value.trim().split(/\s+/);
        const shortestLength = Math.min(...words.map(word => word.length));

        if (shortestLength < minWordLength) {
            setError(`All words must be at least ${minWordLength} characters long. Shortest word found: ${shortestLength} characters.`);
        } else {
            setError('');
        }
    }, [value, minWordLength]);

    return (
        <div className="text-validator">
            <textarea
                value={value}
                onChange={(e) => setValue(e.target.value)}
                placeholder="Enter your text..."
                className={error ? 'error' : ''}
            />
            {error && <p className="error-message">{error}</p>}
        </div>
    );
}

// Usage
<TextValidator minWordLength={3} />
```

### 3. **Reading Level Calculator**

```javascript
function ReadingLevelCalculator({ text }) {
    const calculateReadingLevel = (content) => {
        if (!content) return { level: 'N/A', shortestWord: 0 };

        const words = content.split(/\s+/);
        const wordLengths = words.map(word => word.length);
        const shortestWord = Math.min(...wordLengths);
        const averageLength = wordLengths.reduce((sum, len) => sum + len, 0) / wordLengths.length;

        // Simple reading level calculation
        let level = 'Beginner';
        if (averageLength > 5 && shortestWord >= 3) level = 'Intermediate';
        if (averageLength > 6 && shortestWord >= 4) level = 'Advanced';

        return { level, shortestWord, averageLength: averageLength.toFixed(1) };
    };

    const readingLevel = calculateReadingLevel(text);

    return (
        <div className="reading-level">
            <h3>Reading Level Analysis</h3>
            <div className="level-info">
                <p><strong>Level:</strong> {readingLevel.level}</p>
                <p><strong>Shortest word:</strong> {readingLevel.shortestWord} characters</p>
                <p><strong>Average word length:</strong> {readingLevel.averageLength} characters</p>
            </div>
        </div>
    );
}

// Usage
<ReadingLevelCalculator text="The quick brown fox jumps over the lazy dog" />
```

### 4. **Word Cloud Generator**

```javascript
function WordCloud({ text }) {
    const [words, setWords] = useState([]);

    useEffect(() => {
        if (!text) return;

        const wordArray = text.toLowerCase().split(/\s+/);
        const wordCounts = {};

        // Count word frequencies
        wordArray.forEach(word => {
            wordCounts[word] = (wordCounts[word] || 0) + 1;
        });

        // Convert to array and sort by frequency
        const wordList = Object.entries(wordCounts)
            .map(([word, count]) => ({ word, count, length: word.length }))
            .sort((a, b) => b.count - a.count);

        setWords(wordList);
    }, [text]);

    const shortestLength = words.length > 0 ? Math.min(...words.map(w => w.length)) : 0;

    return (
        <div className="word-cloud">
            <h3>Word Cloud</h3>
            <p>Shortest word length: {shortestLength}</p>
            <div className="cloud">
                {words.map(({ word, count, length }) => (
                    <span
                        key={word}
                        className="word"
                        style={{
                            fontSize: `${Math.max(12, count * 8)}px`,
                            opacity: length === shortestLength ? 0.7 : 1
                        }}
                    >
                        {word} ({count})
                    </span>
                ))}
            </div>
        </div>
    );
}

// Usage
<WordCloud text="the quick brown fox jumps over the lazy dog the fox is quick" />
```

### 5. **Search Query Analyzer**

```javascript
function SearchAnalyzer({ query }) {
    const analyzeQuery = (searchQuery) => {
        if (!searchQuery) return { words: [], shortestLength: 0, suggestions: [] };

        const words = searchQuery.trim().split(/\s+/);
        const shortestLength = Math.min(...words.map(word => word.length));

        // Generate suggestions based on word length
        const suggestions = words
            .filter(word => word.length >= shortestLength + 2)
            .map(word => word.substring(0, word.length - 1));

        return { words, shortestLength, suggestions };
    };

    const analysis = analyzeQuery(query);

    return (
        <div className="search-analyzer">
            <div className="query-info">
                <p><strong>Query:</strong> {query}</p>
                <p><strong>Words:</strong> {analysis.words.join(', ')}</p>
                <p><strong>Shortest word:</strong> {analysis.shortestLength} characters</p>
            </div>

            {analysis.suggestions.length > 0 && (
                <div className="suggestions">
                    <h4>Did you mean:</h4>
                    <ul>
                        {analysis.suggestions.map((suggestion, index) => (
                            <li key={index}>{suggestion}</li>
                        ))}
                    </ul>
                </div>
            )}
        </div>
    );
}

// Usage
<SearchAnalyzer query="javascript react development" />
```

### 6. **Text Summarizer**

```javascript
function TextSummarizer({ text, maxWords = 50 }) {
    const summarize = (content) => {
        if (!content) return { summary: '', wordCount: 0, shortestWord: 0 };

        const words = content.split(/\s+/);
        const shortestWord = Math.min(...words.map(word => word.length));

        // Simple summarization: take first maxWords
        const summaryWords = words.slice(0, maxWords);
        const summary = summaryWords.join(' ');

        return {
            summary: summaryWords.length < words.length ? summary + '...' : summary,
            wordCount: summaryWords.length,
            shortestWord
        };
    };

    const summary = summarize(text);

    return (
        <div className="text-summarizer">
            <h3>Text Summary</h3>
            <div className="summary">
                <p>{summary.summary}</p>
            </div>
            <div className="stats">
                <p>Words in summary: {summary.wordCount}</p>
                <p>Shortest word length: {summary.shortestWord}</p>
            </div>
        </div>
    );
}

// Usage
const longText = "This is a very long text that contains many words and sentences. It demonstrates how to create a text summarizer component in React. The component finds the shortest word and creates a summary.";
<TextSummarizer text={longText} maxWords={20} />
```

## Performance Considerations:

- **split() + map() + Math.min()**: Clean but creates intermediate arrays
- **reduce()**: Memory efficient, single pass through words
- **for...of loop**: Most memory efficient for large texts
- **Regex split**: Handles multiple spaces and punctuation better
- **Large texts**: Consider processing in chunks for very long content

## Common Patterns:

1. **Basic solution**: `Math.min(...sentence.split(/\s+/).map(w => w.length))`
2. **With validation**: Trim and check for empty strings
3. **Punctuation handling**: Remove punctuation before processing
4. **Case sensitivity**: Convert to lowercase for consistent results
5. **Multiple shortest words**: Return array of all shortest words

### Interview Tip:
*"Split the sentence using regex to handle multiple spaces, then use Math.min() with spread operator and map(). Handle edge cases like empty sentences, single words, and punctuation. For better performance with large texts, use reduce() instead of creating intermediate arrays."*