# Week 11: Advanced Async Patterns - Weather Dashboard
## Master Production-Ready Asynchronous JavaScript

### 🎯 Learning Objectives
By completing this project, you will:
- Implement retry logic with exponential backoff for resilient API calls
- Use Promise.race, Promise.allSettled, and Promise.any for advanced coordination
- Build request caching and deduplication to optimize performance
- Handle race conditions with proper debouncing and cancellation
- Create production-ready error boundaries and recovery strategies
- Implement polling patterns for real-time updates

### 📚 Prerequisites
Before starting, you should have:
- Completed Week 11 lecture on Advanced JavaScript
- Reviewed prep work Chapter 10: Promises and Async
- Understanding of async/await from Week 10
- Basic knowledge of fetch API and Promise methods

### 🏗️ Project Overview
Transform a broken weather dashboard into a production-ready application that gracefully handles network issues, coordinates multiple APIs, and provides a smooth user experience even when things go wrong. You'll implement the same patterns used by Netflix, Spotify, and other major applications.

### 🚀 Getting Started

**First:** Accept the GitHub Classroom assignment link provided by your instructor. This will create a new repository with the Week 11 starter code.

1. **Clone your new repository:**
   ```bash
   git clone [your-github-classroom-repo-url]
   cd [your-repo-name]
   ```

2. **Install dependencies and start the development server:**
   ```bash
   npm install
   npm run dev
   ```

3. **In a separate terminal, start the mock API server:**
   ```bash
   npm run api
   ```

4. **Open your browser** to `http://localhost:5173`

You'll see a weather dashboard that's slow, unreliable, and prone to errors. Let's fix it!

### 📝 Step-by-Step Instructions

#### Part 1: Understanding the Problem (5 minutes)

Open the dashboard and try these actions:
1. **Search for a city** - Notice how it's slow and sometimes fails
2. **Click "Refresh All"** - Watch how one failing API blocks everything
3. **Type quickly in search** - See how results arrive out of order
4. **Open Network tab** - Observe duplicate requests for the same data

**The Issues:**
- No retry logic when APIs fail
- No caching (same requests made repeatedly)
- Race conditions in search
- Poor error handling
- No request timeouts

#### Part 2: Implementing Retry Logic with Exponential Backoff (10 minutes)

**Step 1: Create the retry utility**

Open `src/utils/asyncPatterns.js` and implement the retry function:

```javascript
// Retry with exponential backoff
export async function fetchWithRetry(url, options = {}, maxRetries = 3) {
  const { timeout = 5000, ...fetchOptions } = options;

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      // Add timeout to prevent hanging requests
      const controller = new AbortController();
      const timeoutId = setTimeout(() => controller.abort(), timeout);

      const response = await fetch(url, {
        ...fetchOptions,
        signal: controller.signal
      });

      clearTimeout(timeoutId);

      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`);
      }

      return response;
    } catch (error) {
      const isLastAttempt = attempt === maxRetries;

      if (isLastAttempt) {
        console.error(`Failed after ${maxRetries + 1} attempts:`, error);
        throw error;
      }

      // Exponential backoff: 1s, 2s, 4s
      const delay = Math.pow(2, attempt) * 1000;
      console.log(`Retry attempt ${attempt + 1} after ${delay}ms`);

      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}
```

**What's happening here:**
- Each retry waits exponentially longer (1s, 2s, 4s)
- Includes timeout to prevent hanging requests
- Logs attempts for debugging
- Throws error after max retries exhausted

**Step 2: Update weather service to use retry logic**

In `src/services/weatherService.js`, replace the basic fetch with your retry function:

```javascript
import { fetchWithRetry } from '../utils/asyncPatterns';

export async function getWeatherData(city) {
  try {
    const response = await fetchWithRetry(
      `http://localhost:3001/weather?city=${encodeURIComponent(city)}`,
      { timeout: 3000 },
      2 // Max 3 attempts total
    );

    const data = await response.json();
    return { success: true, data };
  } catch (error) {
    return {
      success: false,
      error: error.message,
      fallback: getWeatherFallback(city)
    };
  }
}

// Fallback data when API fails
function getWeatherFallback(city) {
  return {
    city,
    temp: '--',
    condition: 'Data temporarily unavailable',
    humidity: '--',
    wind: '--'
  };
}
```

#### Part 3: Implementing Request Cache and Deduplication (10 minutes)

**Step 1: Create the cache manager**

Add to `src/utils/asyncPatterns.js`:

```javascript
// Simple cache with TTL (time-to-live)
class RequestCache {
  constructor(ttl = 60000) { // Default 1 minute
    this.cache = new Map();
    this.ttl = ttl;
  }

  set(key, value) {
    this.cache.set(key, {
      value,
      timestamp: Date.now()
    });
  }

  get(key) {
    const entry = this.cache.get(key);
    if (!entry) return null;

    const age = Date.now() - entry.timestamp;
    if (age > this.ttl) {
      this.cache.delete(key);
      return null;
    }

    return entry.value;
  }

  clear() {
    this.cache.clear();
  }
}

// Request deduplication
const pendingRequests = new Map();
const cache = new RequestCache(300000); // 5 minute cache

export async function cachedFetch(url, options = {}) {
  const cacheKey = `${url}${JSON.stringify(options)}`;

  // Check cache first
  const cached = cache.get(cacheKey);
  if (cached) {
    console.log('Cache hit:', url);
    return cached;
  }

  // Check if request already pending
  if (pendingRequests.has(cacheKey)) {
    console.log('Request deduped:', url);
    return pendingRequests.get(cacheKey);
  }

  // Make the request
  const requestPromise = fetchWithRetry(url, options)
    .then(async response => {
      const data = await response.json();
      cache.set(cacheKey, data);
      return data;
    })
    .finally(() => {
      pendingRequests.delete(cacheKey);
    });

  pendingRequests.set(cacheKey, requestPromise);
  return requestPromise;
}
```

**What's happening here:**
- Cache stores successful responses with timestamps
- Automatic expiration after TTL
- Deduplication prevents multiple identical requests
- Returns cached data when available

#### Part 4: Using Promise Combinators for Parallel Requests (10 minutes)

**Step 1: Implement parallel data fetching with resilience**

Create `src/services/dashboardService.js`:

```javascript
import { cachedFetch } from '../utils/asyncPatterns';

export async function fetchDashboardData(city) {
  // Define all data sources
  const requests = [
    {
      name: 'weather',
      promise: cachedFetch(`http://localhost:3001/weather?city=${city}`)
        .catch(err => ({ error: err.message, fallback: true }))
    },
    {
      name: 'forecast',
      promise: cachedFetch(`http://localhost:3001/forecast?city=${city}`)
        .catch(err => ({ error: err.message, fallback: true }))
    },
    {
      name: 'news',
      promise: cachedFetch(`http://localhost:3001/news?city=${city}`)
        .catch(err => ({ error: err.message, fallback: true }))
    },
    {
      name: 'alerts',
      promise: cachedFetch(`http://localhost:3001/alerts?city=${city}`)
        .catch(err => ({ error: err.message, fallback: true }))
    }
  ];

  // Use allSettled to get all results, even if some fail
  const results = await Promise.allSettled(
    requests.map(r => r.promise)
  );

  // Process results
  const data = {};
  results.forEach((result, index) => {
    const { name } = requests[index];

    if (result.status === 'fulfilled') {
      data[name] = result.value;
    } else {
      data[name] = {
        error: true,
        message: 'Service temporarily unavailable'
      };
    }
  });

  return data;
}

// Race pattern for fastest response
export async function getFastestWeatherSource(city) {
  const sources = [
    cachedFetch(`http://localhost:3001/weather-source1?city=${city}`),
    cachedFetch(`http://localhost:3001/weather-source2?city=${city}`),
    cachedFetch(`http://localhost:3001/weather-source3?city=${city}`)
  ];

  try {
    // Promise.race returns first to resolve
    const result = await Promise.race(sources);
    return result;
  } catch (error) {
    // All sources failed
    console.error('All weather sources failed');
    throw error;
  }
}

// Get any successful response
export async function getAnyWeatherData(city) {
  const sources = [
    cachedFetch(`http://localhost:3001/weather-primary?city=${city}`),
    cachedFetch(`http://localhost:3001/weather-backup1?city=${city}`),
    cachedFetch(`http://localhost:3001/weather-backup2?city=${city}`)
  ];

  try {
    // Promise.any returns first successful (ES2021)
    const result = await Promise.any(sources);
    return result;
  } catch (error) {
    // AggregateError when all promises reject
    console.error('No weather sources available');
    throw error;
  }
}
```

**What's happening here:**
- `Promise.allSettled` gets all results regardless of failures
- `Promise.race` uses fastest responding service
- `Promise.any` provides automatic failover
- Each pattern serves different resilience needs

#### Part 5: Handling Search with Debouncing and Cancellation (10 minutes)

**Step 1: Implement search with race condition prevention**

Create `src/hooks/useSearch.js`:

```javascript
import { useState, useCallback, useRef } from 'react';

export function useSearch(searchFn, delay = 500) {
  const [results, setResults] = useState(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  const abortControllerRef = useRef(null);
  const timeoutRef = useRef(null);

  const search = useCallback(async (query) => {
    // Clear previous timeout
    if (timeoutRef.current) {
      clearTimeout(timeoutRef.current);
    }

    // Cancel previous request
    if (abortControllerRef.current) {
      abortControllerRef.current.abort();
    }

    if (!query) {
      setResults(null);
      return;
    }

    // Debounce the search
    timeoutRef.current = setTimeout(async () => {
      setLoading(true);
      setError(null);

      // Create new abort controller for this request
      abortControllerRef.current = new AbortController();

      try {
        const data = await searchFn(query, abortControllerRef.current.signal);

        // Only update if not aborted
        if (!abortControllerRef.current.signal.aborted) {
          setResults(data);
          setError(null);
        }
      } catch (err) {
        if (err.name !== 'AbortError') {
          setError(err.message);
          setResults(null);
        }
      } finally {
        setLoading(false);
      }
    }, delay);
  }, [searchFn, delay]);

  // Cleanup function
  const cancel = useCallback(() => {
    if (timeoutRef.current) {
      clearTimeout(timeoutRef.current);
    }
    if (abortControllerRef.current) {
      abortControllerRef.current.abort();
    }
  }, []);

  return { results, loading, error, search, cancel };
}
```

**What's happening here:**
- Debounces input to reduce API calls
- Cancels in-flight requests when new search starts
- Prevents race conditions with AbortController
- Provides loading and error states

#### Part 6: Implementing Polling for Live Updates (5 minutes)

Add to `src/hooks/usePolling.js`:

```javascript
import { useEffect, useRef } from 'react';

export function usePolling(callback, interval = 30000, enabled = true) {
  const savedCallback = useRef();
  const intervalIdRef = useRef();

  useEffect(() => {
    savedCallback.current = callback;
  }, [callback]);

  useEffect(() => {
    if (!enabled) {
      if (intervalIdRef.current) {
        clearInterval(intervalIdRef.current);
      }
      return;
    }

    // Immediate first call
    savedCallback.current();

    // Set up polling
    intervalIdRef.current = setInterval(() => {
      savedCallback.current();
    }, interval);

    // Cleanup
    return () => {
      if (intervalIdRef.current) {
        clearInterval(intervalIdRef.current);
      }
    };
  }, [interval, enabled]);

  // Manual refresh function
  const refresh = () => {
    savedCallback.current();
  };

  return { refresh };
}
```

### 🎨 Customization Ideas

- **Visual Feedback**: Add loading skeletons for each data section
- **Error Recovery UI**: Show "Retry" buttons for failed sections
- **Performance Metrics**: Display cache hit rates and response times
- **Smart Polling**: Increase interval when tab is inactive
- **Offline Mode**: Store last successful results in localStorage

### 🏆 Challenge Extensions

**Intermediate: Circuit Breaker Pattern**
Implement a circuit breaker that stops calling failing services:
```javascript
class CircuitBreaker {
  constructor(threshold = 5, timeout = 60000) {
    this.failureCount = 0;
    this.threshold = threshold;
    this.timeout = timeout;
    this.state = 'CLOSED'; // CLOSED, OPEN, HALF_OPEN
    this.nextAttempt = Date.now();
  }

  async call(fn) {
    if (this.state === 'OPEN') {
      if (Date.now() < this.nextAttempt) {
        throw new Error('Circuit breaker is OPEN');
      }
      this.state = 'HALF_OPEN';
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  onSuccess() {
    this.failureCount = 0;
    this.state = 'CLOSED';
  }

  onFailure() {
    this.failureCount++;
    if (this.failureCount >= this.threshold) {
      this.state = 'OPEN';
      this.nextAttempt = Date.now() + this.timeout;
    }
  }
}
```

**Advanced: Priority Queue for Requests**
Manage request priority during high load:
```javascript
class PriorityQueue {
  constructor(maxConcurrent = 3) {
    this.queue = [];
    this.running = 0;
    this.maxConcurrent = maxConcurrent;
  }

  async add(fn, priority = 0) {
    return new Promise((resolve, reject) => {
      this.queue.push({ fn, priority, resolve, reject });
      this.queue.sort((a, b) => b.priority - a.priority);
      this.process();
    });
  }

  async process() {
    if (this.running >= this.maxConcurrent || this.queue.length === 0) {
      return;
    }

    this.running++;
    const { fn, resolve, reject } = this.queue.shift();

    try {
      const result = await fn();
      resolve(result);
    } catch (error) {
      reject(error);
    } finally {
      this.running--;
      this.process();
    }
  }
}
```

**Graduate (253A): Adaptive Retry Strategy**
Implement smart retry that adapts based on error type:
```javascript
function getRetryStrategy(error) {
  // Rate limit: wait longer
  if (error.status === 429) {
    const retryAfter = error.headers?.get('Retry-After');
    return {
      maxRetries: 5,
      delay: retryAfter ? parseInt(retryAfter) * 1000 : 60000
    };
  }

  // Server errors: aggressive retry
  if (error.status >= 500) {
    return {
      maxRetries: 3,
      delay: 1000,
      backoff: 2
    };
  }

  // Network errors: moderate retry
  if (!error.status) {
    return {
      maxRetries: 2,
      delay: 2000,
      backoff: 1.5
    };
  }

  // Client errors: no retry
  return {
    maxRetries: 0
  };
}
```

### 🐛 Troubleshooting Guide

**Issue: "Network request failed"**
- Solution: Ensure mock API is running (`npm run api`)
- Check if port 3001 is available

**Issue: "Results arrive out of order"**
- Solution: Implement AbortController cancellation
- Verify debounce delay is appropriate

**Issue: "Memory leak warning"**
- Solution: Clean up timeouts and intervals in useEffect
- Cancel pending requests on unmount

**Issue: "Too many API calls"**
- Solution: Increase cache TTL
- Add debounce to search input
- Check for duplicate event handlers

### 📖 Resources & References

- [MDN: Promise Methods](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise)
- [MDN: AbortController](https://developer.mozilla.org/en-US/docs/Web/API/AbortController)
- [Exponential Backoff Explained](https://en.wikipedia.org/wiki/Exponential_backoff)
- [Circuit Breaker Pattern](https://martinfowler.com/bliki/CircuitBreaker.html)
- [Web Workers for Heavy Computation](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API)

### ✅ Project Checklist

Before considering your project complete, ensure:

- [ ] **Retry Logic Works**: Failed requests retry with increasing delays
- [ ] **Cache Prevents Duplicates**: Same request within 5 minutes uses cache
- [ ] **Search Handles Race Conditions**: Fast typing doesn't cause wrong results
- [ ] **Parallel Requests Complete**: Dashboard loads even if some APIs fail
- [ ] **Error Messages Are Helpful**: Users understand what went wrong
- [ ] **Loading States Are Accurate**: Each section shows its own loading state
- [ ] **Polling Updates Data**: Weather refreshes every 30 seconds
- [ ] **Memory Leaks Prevented**: No console warnings about unmounted components
- [ ] **Network Tab Shows Optimization**: Fewer requests than before
- [ ] **Dashboard Remains Responsive**: UI doesn't freeze during operations

### 💡 Key Takeaways

By completing this project, you've learned:
- How to make unreliable APIs reliable with retry logic
- Why caching and deduplication matter for performance
- How Promise combinators solve different coordination problems
- Why race conditions happen and how to prevent them
- How production apps handle errors gracefully

These patterns appear in every production JavaScript application. Whether you're building with React, Vue, Angular, or vanilla JS, you'll use these techniques to create robust, performant applications that users can depend on.