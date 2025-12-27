---
title: "Fetch API và Async/Await trong JavaScript"
date: 2025-12-08
description: "Làm chủ lập trình bất đồng bộ với Fetch API, Promises và async/await để giao tiếp với REST API hiệu quả..."
image: "https://images.unsplash.com/photo-1633356122544-f134324a6cee?w=800&h=500&fit=crop"
---

Fetch API là interface hiện đại để thực hiện HTTP requests trong JavaScript. Kết hợp với async/await, nó giúp code bất đồng bộ trở nên dễ đọc và maintain hơn.

## Promises Basics

### Promise States

- **Pending**: Đang chờ xử lý
- **Fulfilled**: Hoàn thành thành công
- **Rejected**: Bị từ chối (lỗi)

### Creating Promises

```javascript
const myPromise = new Promise((resolve, reject) => {
    setTimeout(() => {
        const success = true;
        if (success) {
            resolve('Operation successful!');
        } else {
            reject('Operation failed!');
        }
    }, 1000);
});

myPromise
    .then(result => console.log(result))
    .catch(error => console.error(error));
```

## Fetch API Basics

### Simple GET Request

```javascript
fetch('https://api.example.com/users')
    .then(response => response.json())
    .then(data => console.log(data))
    .catch(error => console.error('Error:', error));
```

### POST Request

```javascript
fetch('https://api.example.com/users', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
    },
    body: JSON.stringify({
        name: 'John Doe',
        email: 'john@example.com'
    })
})
.then(response => response.json())
.then(data => console.log('Created:', data))
.catch(error => console.error('Error:', error));
```

## Async/Await

### Converting to Async/Await

```javascript
// Traditional Promise chain
function getUsers() {
    return fetch('https://api.example.com/users')
        .then(response => response.json())
        .then(data => data)
        .catch(error => {
            console.error('Error:', error);
            throw error;
        });
}

// Using async/await
async function getUsers() {
    try {
        const response = await fetch('https://api.example.com/users');
        const data = await response.json();
        return data;
    } catch (error) {
        console.error('Error:', error);
        throw error;
    }
}
```

## Error Handling

### Checking Response Status

```javascript
async function fetchUser(id) {
    try {
        const response = await fetch(`https://api.example.com/users/${id}`);
        
        if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status}`);
        }
        
        const user = await response.json();
        return user;
    } catch (error) {
        console.error('Failed to fetch user:', error);
        throw error;
    }
}
```

## Request Configuration

### Full Fetch Options

```javascript
async function createUser(userData) {
    const response = await fetch('https://api.example.com/users', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${token}`,
            'Accept': 'application/json'
        },
        body: JSON.stringify(userData),
        mode: 'cors',
        credentials: 'include',
        cache: 'no-cache'
    });
    
    return response.json();
}
```

## Concurrent Requests

### Promise.all()

Chạy nhiều requests đồng thời:

```javascript
async function fetchMultipleUsers(ids) {
    try {
        const promises = ids.map(id => 
            fetch(`https://api.example.com/users/${id}`)
                .then(res => res.json())
        );
        
        const users = await Promise.all(promises);
        return users;
    } catch (error) {
        console.error('Error fetching users:', error);
        throw error;
    }
}

// Usage
const userIds = [1, 2, 3, 4, 5];
const users = await fetchMultipleUsers(userIds);
```

### Promise.allSettled()

Đợi tất cả promises hoàn thành, kể cả lỗi:

```javascript
async function fetchUsersWithFallback(ids) {
    const promises = ids.map(id => 
        fetch(`https://api.example.com/users/${id}`)
            .then(res => res.json())
    );
    
    const results = await Promise.allSettled(promises);
    
    const successful = results
        .filter(result => result.status === 'fulfilled')
        .map(result => result.value);
    
    const failed = results
        .filter(result => result.status === 'rejected')
        .map(result => result.reason);
    
    return { successful, failed };
}
```

## Building a RESTful API Client

### Complete API Client Class

```javascript
class APIClient {
    constructor(baseURL) {
        this.baseURL = baseURL;
        this.token = null;
    }
    
    setToken(token) {
        this.token = token;
    }
    
    async request(endpoint, options = {}) {
        const url = `${this.baseURL}${endpoint}`;
        const headers = {
            'Content-Type': 'application/json',
            ...options.headers
        };
        
        if (this.token) {
            headers.Authorization = `Bearer ${this.token}`;
        }
        
        const config = {
            ...options,
            headers
        };
        
        try {
            const response = await fetch(url, config);
            
            if (!response.ok) {
                const error = await response.json();
                throw new Error(error.message || 'Request failed');
            }
            
            return await response.json();
        } catch (error) {
            console.error('API request failed:', error);
            throw error;
        }
    }
    
    async get(endpoint) {
        return this.request(endpoint, { method: 'GET' });
    }
    
    async post(endpoint, data) {
        return this.request(endpoint, {
            method: 'POST',
            body: JSON.stringify(data)
        });
    }
    
    async put(endpoint, data) {
        return this.request(endpoint, {
            method: 'PUT',
            body: JSON.stringify(data)
        });
    }
    
    async delete(endpoint) {
        return this.request(endpoint, { method: 'DELETE' });
    }
}

// Usage
const api = new APIClient('https://api.example.com');
api.setToken('your-jwt-token');

const users = await api.get('/users');
const newUser = await api.post('/users', { name: 'John' });
```

## Retry Logic

### Automatic Retry on Failure

```javascript
async function fetchWithRetry(url, options = {}, retries = 3) {
    for (let i = 0; i < retries; i++) {
        try {
            const response = await fetch(url, options);
            if (response.ok) {
                return response;
            }
            
            if (i === retries - 1) {
                throw new Error(`Failed after ${retries} retries`);
            }
        } catch (error) {
            if (i === retries - 1) {
                throw error;
            }
            
            // Wait before retry (exponential backoff)
            await new Promise(resolve => 
                setTimeout(resolve, Math.pow(2, i) * 1000)
            );
        }
    }
}
```

## Timeout Handling

### Adding Request Timeout

```javascript
async function fetchWithTimeout(url, options = {}, timeout = 5000) {
    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort(), timeout);
    
    try {
        const response = await fetch(url, {
            ...options,
            signal: controller.signal
        });
        clearTimeout(timeoutId);
        return response;
    } catch (error) {
        clearTimeout(timeoutId);
        if (error.name === 'AbortError') {
            throw new Error('Request timeout');
        }
        throw error;
    }
}
```

## Best Practices

1. **Always Handle Errors**: Use try-catch with async/await
2. **Check Response Status**: Verify `response.ok` before parsing
3. **Use Async/Await**: More readable than promise chains
4. **Implement Retry Logic**: For network failures
5. **Add Timeouts**: Prevent hanging requests
6. **Validate Data**: Check data before sending
7. **Use TypeScript**: For better type safety
8. **Cache When Appropriate**: Reduce unnecessary requests

## Kết luận

Fetch API kết hợp với async/await là công cụ mạnh mẽ để làm việc với REST APIs. Hiểu rõ cách sử dụng chúng sẽ giúp bạn xây dựng được các ứng dụng web hiện đại và hiệu quả.
