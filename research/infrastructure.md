# Infrastructure & Pipeline Research: CLI Web API Wrapper

**Research Date:** 2026-02-13  
**Purpose:** Building a Node.js CLI that wraps a third-party web app's API  
**Research Method:** Perplexity Pro (sonar-pro model)

---

## 1. Token Management Pipeline

### Secure Local Token Storage

**Recommendation: Use platform-native credential managers** like `keytar` (cross-platform) or `node-keychain` (macOS-specific), as they leverage OS keychains to store tokens encrypted and isolated from the filesystem, outperforming encrypted JSON files which remain vulnerable to key extraction attacks.

### Library Comparison

| Approach | Pros | Cons | Security Level | Best For |
|----------|------|------|----------------|----------|
| **keytar** | Cross-platform; tokens encrypted by OS with hardware-backed isolation (Secure Enclave); no app-managed keys needed; resistant to filesystem access | Requires native dependencies (`npm i keytar`); Linux setup may need `libsecret` tools | **Highest**: OS-level encryption, no plaintext exposure | Production CLIs needing broad OS support |
| **node-keychain** | Simple API (`keychain.setPassword()`/`getPassword()`); seamless macOS Keychain integration | macOS-only; no Windows/Linux support | **High**: Leverages Keychain Services directly | macOS-focused CLIs |
| **Encrypted JSON** | Portable; easy to implement with Node's `crypto` module or libs like `keyv`; works offline | Requires deriving/storing encryption key (e.g., from passphrase or env var); vulnerable to key theft; file inspectable if key compromised; no hardware protection | **Medium**: Depends on key strength; susceptible to brute-force attacks | Prototypes or air-gapped envs; avoid for sensitive prod tokens |

**Winner: keytar** for cross-platform reliability and superior isolation.

### Implementation Example (keytar)

```javascript
const keytar = require('keytar');

// Store token
await keytar.setPassword('myapp', 'user@example.com', accessToken);

// Retrieve token
const token = await keytar.getPassword('myapp', 'user@example.com');

// Delete token (logout)
await keytar.deletePassword('myapp', 'user@example.com');
```

### OAuth Token Refresh Flow

OAuth flows (Authorization Code with PKCE for CLIs) involve:
- **access_token** — short-lived (1h)
- **refresh_token** — longer-lived (days/weeks)

**Best Practices:**

1. **Storage**: Keep both in keytar/node-keychain under service-specific labels
2. **Refresh Logic**:
   - On API call, check `access_token` expiry (parse `exp` claim)
   - If expired, fetch refresh_token from store → POST to `/token` endpoint with `grant_type=refresh_token`
   - Store new **access_token** + rotated **refresh_token** (overwrite old one)
   - Handle 401s by triggering refresh automatically
3. **Error Handling**: If refresh fails (400 Invalid Grant), delete stored tokens and re-prompt for OAuth flow

**Libraries**: Use `openid-client` or `simple-oauth2` for flows

```javascript
const keytar = require('keytar');
const { TokenSet } = require('openid-client');

async function refreshIfNeeded(tokenSet) {
  if (tokenSet.expired()) {
    const refreshToken = await keytar.getPassword('mycli', 'refresh');
    const newSet = await client.refresh(new TokenSet({ refresh_token: refreshToken }));
    await keytar.setPassword('mycli', 'access', newSet.access_token);
    await keytar.setPassword('mycli', 'refresh', newSet.refresh_token);
    return newSet;
  }
  return tokenSet;
}
```

### JWT Token Rotation Patterns

Implement **Refresh Token Rotation (ROT)** to limit compromise impact:

- **ROT Flow**: On each refresh, server issues new **refresh_token** and invalidates the old one
- **Expiry Guidelines**: 
  - Access: 15-30min (`expiresIn: '15m'`)
  - Refresh: 7-14 days
- **Revocation**: On logout/suspicion, delete from keytar + notify server to blacklist

```javascript
// Rotation-aware refresh
async function rotateJWT(oldRefresh) {
  const res = await fetch('/refresh', { 
    body: JSON.stringify({ refresh_token: oldRefresh }),
    headers: { 'Content-Type': 'application/json' }
  });
  const { access_token, refresh_token } = await res.json();
  await keytar.setPassword('mycli', 'access', access_token);
  await keytar.setPassword('mycli', 'refresh', refresh_token); // Rotated token
}
```

**Security Additions**: Always use HTTPS; validate `aud`/`iss`; short expiries; no sensitive payload data.

---

## 2. Request Pipeline: Mimicking Browser Behavior

### HTTP Client Comparison

**undici** is the fastest option (18,000 req/sec vs 3,200 for Node's http module), achieving 3x acceleration compared to got, fetch, axios or node-fetch. Node.js v18+ includes a built-in fetch() implementation powered by undici.

### Setting Browser-Like Headers

All three libraries support custom headers to imitate Chrome-like requests:

#### got (recommended for modern defaults)

```javascript
const got = require('got');

const response = await got('https://example.com', {
  headers: {
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36',
    'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8',
    'Accept-Language': 'en-US,en;q=0.9',
    'Accept-Encoding': 'gzip, deflate, br',
    'Referer': 'https://google.com/',
    'Sec-Fetch-Mode': 'navigate',
    'Sec-Fetch-Site': 'none',
    'Sec-Fetch-Dest': 'document',
    'Sec-Ch-Ua': '"Not_A Brand";v="8", "Chromium";v="120", "Google Chrome";v="120"',
    'Sec-Ch-Ua-Mobile': '?0',
    'Sec-Ch-Ua-Platform': '"Windows"'
  }
});
```

#### axios

```javascript
const axios = require('axios');

const response = await axios.get('https://example.com', {
  headers: {
    'User-Agent': 'Mozilla/5.0 ...',
    'Accept': 'text/html...',
    'Referer': 'https://google.com/'
  }
});
```

#### undici (Node.js 18+, fastest)

```javascript
const { request } = require('undici');

const { body } = await request('https://example.com', {
  headers: {
    // Same browser-like headers as above
  }
});
```

### Cookie Jar Management

#### got with got-scraping (includes tough-cookie)

```javascript
const got = require('got-scraping');

const jar = got.jar(); // tough-cookie jar
const client = got.extend({ cookieJar: jar });

await client('https://example.com'); // Sets cookies
await client('https://example.com/profile'); // Sends cookies automatically
```

#### axios with tough-cookie + axios-cookiejar-support

```bash
npm i axios-cookiejar-support tough-cookie
```

```javascript
const axios = require('axios');
const { wrapper } = require('axios-cookiejar-support');
const { CookieJar } = require('tough-cookie');

const jar = new CookieJar();
const client = wrapper(axios.create({ jar }));

await client.get('https://example.com');
```

### Referer Handling

Set dynamically based on navigation chain:

```javascript
const referers = new Map();
referers.set('https://example.com/profile', 'https://example.com/');

const response = await got(url, {
  headers: {
    'Referer': referers.get(url) || 'https://google.com/'
  }
});
```

### TLS Client Hello Fingerprinting (Advanced)

Basic clients use Node.js TLS stack, easily detected. Use these libraries:

#### 1. curl-impersonate equivalent

```bash
npm i curl-impersonate-node
```

```javascript
const impersonate = require('curl-impersonate-node');
await impersonate('chrome120', 'https://example.com');
```

#### 2. undici-fingerprint (most advanced)

```bash
npm i @undici/fingerprint
```

Customizes TLS JA3 fingerprint to match Chrome.

#### 3. playwright or puppeteer-extra-plugin-stealth (full browser)

```javascript
const { chromium } = require('playwright-extra');
const stealth = require('puppeteer-extra-plugin-stealth')();
chromium.use(stealth);

const browser = await chromium.launch({ headless: true });
const page = await browser.newPage();
await page.goto('https://example.com');
```

Perfect TLS/JS/WebGL fingerprinting.

#### 4. node-fetch with undici dispatcher + custom TLS

```javascript
const fetch = require('node-fetch');
const { Agent } = require('undici');

const agent = new Agent({
  connect: {
    // Custom TLS ciphers matching Chrome
    ciphers: 'TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:...',
    sigalgs: 'ecdsa_secp256r1_sha256:rsa_pss_rsae_sha256:...'
  }
});

await fetch(url, { agent, headers: browserHeaders });
```

### Detection Evasion Comparison

| Technique | Basic Clients | curl-impersonate | Playwright Stealth |
|-----------|---------------|------------------|-------------------|
| **Headers** | ✅ Easy | ✅ Perfect | ✅ Perfect |
| **Cookies** | ✅ w/ jars | ✅ Automatic | ✅ Automatic |
| **TLS Fingerprint** | ❌ Node.js | ✅ Chrome | ✅ Chrome |
| **JA3** | ❌ Distinct | ✅ Matches | ✅ Matches |
| **HTTP/2** | ✅ undici | ✅ Chrome | ✅ Chrome |
| **JS Execution** | ❌ None | ❌ None | ✅ Full |

**Recommendation**: Use **got-scraping** + **undici-fingerprint** for HTTP-only scraping. Use **playwright-extra** for sites requiring JavaScript or advanced anti-bot (Cloudflare, PerimeterX).

---

## 3. Playwright for Authentication

### Launching in Headless Mode

Configure Chromium for headless execution to minimize resources while mimicking real user behavior:

```javascript
import { chromium } from 'playwright';

const browser = await chromium.launch({ headless: true });
```

This speeds up execution (2x-15x faster than headed) and suits non-interactive CLI/server environments. Add stealth flags like `--window-size` or plugins (e.g., playwright-extra) to evade detection.

### Persisting Browser Context and Cookies

Use **persistent contexts** with a user data directory to save/restore cookies, localStorage, and session state across runs—ideal for CLI tools avoiding repeated logins:

```javascript
const userDataDir = './user-data'; // Persist to disk
const context = await chromium.launchPersistentContext(userDataDir, {
  headless: true,
  viewport: { width: 1920, height: 1080 } // Consistent fingerprint
});
const page = await context.newPage();
```

**Workflow:**
- **First run**: Perform login; state saves automatically
- **Subsequent runs**: Reuse state from `./user-data` for instant authenticated sessions
- **Multi-session CLI**: Create isolated directories per user/profile

Share auth contexts across sessions to skip logins, but isolate per test for independence.

### Automating Login Flows

Follow Playwright's **auto-waiting** (no fixed delays needed) and Page Object Model for maintainable scripts:

```javascript
await page.goto('https://example.com/login');
await page.fill('#username', 'user');
await page.fill('#password', 'pass');
await page.click('#login-btn'); // Auto-waits for actionable state
```

**Best Practices:**
- Use stable locators (e.g., `getByRole`, `getByTestId`)
- Parallelize with workers/sharding for CLI scale
- Add logging/screenshots for headless debugging:

```javascript
page.on('console', msg => console.log('Console:', msg.text()));
await page.screenshot({ path: 'login.png', fullPage: true });
```

### Handling 2FA

Intercept network requests or wait for 2FA UI dynamically:

```javascript
// Wait for 2FA prompt
await page.waitForSelector('#totp-input', { timeout: 10000 });

// Option 1: User input via CLI (pause for manual entry)
const readline = require('readline').createInterface({
  input: process.stdin,
  output: process.stdout
});
const code = await new Promise(resolve => 
  readline.question('Enter 2FA code: ', resolve)
);
await page.fill('#totp-input', code);
await page.click('#verify-2fa');

// Option 2: Auto-generate from seed (e.g., via speakeasy lib)
const speakeasy = require('speakeasy');
const totp = speakeasy.totp({ secret: 'JBSWY3DPEHPK3PXP' });
await page.fill('#totp-input', totp);
```

Randomize delays/actions for human-like behavior to avoid rate limits.

### Detecting Successful Login

Check multiple indicators post-login:

```javascript
// URL change
await page.waitForURL('**/dashboard', { timeout: 10000 });

// Element presence
await expect(page.getByText('Welcome')).toBeVisible();

// Absence of login elements
await expect(page.locator('#login-form')).toBeHidden();
```

Combine with console logs or title checks for robustness.

### Extracting Authentication Tokens

Pull tokens immediately after login detection for CLI export:

| Method | Code Example | Use Case |
|--------|--------------|----------|
| **localStorage** | `const token = await page.evaluate(() => localStorage.getItem('authToken'));` | JWTs in SPA storage |
| **Cookies** | `const cookies = await context.cookies(); const token = cookies.find(c => c.name === 'sessionToken')?.value;` | Session cookies |
| **Network Requests** | ```page.on('response', async resp => { if (resp.url().includes('/auth')) { const headers = resp.headers(); console.log(headers['authorization']); } });``` | API tokens in headers |

**Export to CLI:**

```javascript
const authData = {
  token: extractedToken,
  expires: new Date(Date.now() + 3600000)
};
console.log(JSON.stringify(authData));
```

### Additional CLI Best Practices

- **Error Handling**: Retry with exponential backoff; trace on failure (`trace: 'on-first-retry'`)
- **Timeouts**: Default 30s; customize per action
- **Cleanup**: `await context.close();` but skip for persistent dirs
- Run locally headed (`headless: false`) for dev, headless for prod/CLI

---

## 4. Output Pipeline for Image Generation

### Async Generation Pattern

Image generation APIs often return a task ID for async processing, requiring polling until completion.

### Polling Task Status

Implement a polling loop with exponential backoff to avoid rate limits:

```javascript
const axios = require('axios');

async function pollTaskStatus(taskId, apiKey, baseUrl, maxRetries = 30, delay = 2000) {
  let retries = 0;
  while (retries < maxRetries) {
    try {
      const response = await axios.get(`${baseUrl}/tasks/${taskId}`, {
        headers: { Authorization: `Bearer ${apiKey}` }
      });
      const status = response.data.status;
      
      if (status === 'completed') return response.data.resultUrl;
      if (status === 'failed') throw new Error('Task failed');
      
      await new Promise(resolve => setTimeout(resolve, delay));
      delay = Math.min(delay * 1.5, 10000); // Exponential backoff, cap at 10s
      retries++;
    } catch (error) {
      if (retries === maxRetries - 1) throw error;
    }
  }
  throw new Error('Polling timeout');
}
```

This pattern ensures resilience; adjust `maxRetries` and `delay` based on API docs.

### Downloading Binary Results Efficiently

Stream downloads to avoid loading large images into memory:

```javascript
const fs = require('fs');
const https = require('https');

async function downloadImage(url, outputPath) {
  return new Promise((resolve, reject) => {
    const writer = fs.createWriteStream(outputPath);
    https.get(url, (response) => {
      response.pipe(writer);
      writer.on('finish', resolve);
      writer.on('error', reject);
    }).on('error', reject);
  });
}
```

This handles multi-GB images without OOM errors; add progress via `response.headers['content-length']`.

### Progress Indicators

#### Using ora for Spinners

```javascript
const ora = require('ora');

async function generateImageWithProgress(prompt, apiKey, baseUrl, outputPath) {
  const spinner = ora('Generating image...').start();
  try {
    const taskId = await submitGeneration(prompt, apiKey, baseUrl);
    spinner.text = 'Polling status...';
    const imageUrl = await pollTaskStatus(taskId, apiKey, baseUrl);
    spinner.text = 'Downloading image...';
    await downloadImage(imageUrl, outputPath);
    spinner.succeed(`Image saved to ${outputPath}`);
  } catch (error) {
    spinner.fail(error.message);
    throw error;
  }
}
```

#### Using cli-progress for Download Bars

```javascript
const cliProgress = require('cli-progress');
const axios = require('axios');
const fs = require('fs');

async function downloadWithProgress(url, outputPath) {
  const { data: stream, headers } = await axios({
    url,
    method: 'GET',
    responseType: 'stream'
  });
  
  const totalLength = headers['content-length'];
  const bar = new cliProgress.SingleBar({}, cliProgress.Presets.shades_classic);
  bar.start(totalLength, 0);
  
  const writer = fs.createWriteStream(outputPath);
  
  stream.on('data', (chunk) => bar.increment(chunk.length));
  stream.pipe(writer);
  
  return new Promise((resolve, reject) => {
    writer.on('finish', () => {
      bar.stop();
      resolve();
    });
    writer.on('error', reject);
  });
}
```

### Managing Parallel Requests

Limit concurrency to respect API quotas using **p-queue** or **p-limit**:

```bash
npm i p-queue
```

```javascript
const { Queue } = require('p-queue');
const queue = new Queue({ concurrency: 3 }); // API limit-dependent

async function generateBatch(prompts, apiKey, baseUrl) {
  const results = await Promise.all(
    prompts.map(prompt => 
      queue.add(() => generateImageWithProgress(prompt, apiKey, baseUrl, `output-${prompt}.png`))
    )
  );
  return results;
}
```

**p-limit** is similar but simpler for fixed limits. Set `concurrency` to your API's rate (e.g., 5-10 for most image APIs) to prevent bans.

**Full Example with Commander**:

```javascript
const { Command } = require('commander');
const program = new Command();

program
  .command('generate <prompt>')
  .option('-o, --output <path>', 'Output path')
  .option('-n, --count <number>', 'Number of variations', 1)
  .action(async (prompt, options) => {
    const prompts = Array(parseInt(options.count)).fill(prompt);
    await generateBatch(prompts, process.env.API_KEY, 'https://api.example.com');
  });

program.parse();
```

**Graceful Shutdown**:

```javascript
process.on('SIGINT', async () => {
  console.log('\nDraining queue...');
  await queue.onIdle();
  process.exit(0);
});
```

---

## 5. Error Handling & Retry Patterns

### Rate Limit Detection and Backoff

Monitor `429 Too Many Requests` status and `Retry-After` header:

```javascript
const axios = require('axios');

async function makeRequest(url, options = {}) {
  try {
    return await axios.get(url, options);
  } catch (error) {
    if (error.response?.status === 429) {
      const retryAfter = parseInt(error.response.headers['retry-after'] || '60', 10);
      console.log(`Rate limited. Retrying after ${retryAfter}s...`);
      await new Promise(resolve => setTimeout(resolve, retryAfter * 1000));
      return await makeRequest(url, options); // Retry after waiting
    }
    throw error;
  }
}
```

**Exponential Backoff** (for APIs without Retry-After):

```javascript
async function exponentialBackoff(fn, maxRetries = 5) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fn();
    } catch (error) {
      if (error.response?.status !== 429 || i === maxRetries - 1) throw error;
      const delay = Math.min(1000 * Math.pow(2, i), 32000); // Cap at 32s
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}
```

### Token Expiry Detection and Automatic Refresh

Intercept 401 errors and trigger refresh flow:

```javascript
class APIClient {
  #accessToken;
  #refreshToken;

  async request(url) {
    try {
      return await axios.get(url, { 
        headers: { Authorization: `Bearer ${this.#accessToken}` } 
      });
    } catch (error) {
      if (error.response?.status === 401) {
        await this.refresh();
        return await axios.get(url, { 
          headers: { Authorization: `Bearer ${this.#accessToken}` } 
        });
      }
      throw error;
    }
  }

  async refresh() {
    const response = await axios.post('/token', {
      grant_type: 'refresh_token',
      refresh_token: this.#refreshToken
    });
    this.#accessToken = response.data.access_token;
    this.#refreshToken = response.data.refresh_token;
    
    // Persist to keytar
    await keytar.setPassword('mycli', 'access', this.#accessToken);
    await keytar.setPassword('mycli', 'refresh', this.#refreshToken);
  }
}
```

**Use mutex** (e.g., `async-mutex`) for concurrent refresh to avoid race conditions:

```bash
npm i async-mutex
```

```javascript
const { Mutex } = require('async-mutex');

class APIClient {
  #mutex = new Mutex();
  
  async refresh() {
    const release = await this.#mutex.acquire();
    try {
      // Only one refresh at a time
      const response = await axios.post('/token', {
        grant_type: 'refresh_token',
        refresh_token: this.#refreshToken
      });
      this.#accessToken = response.data.access_token;
      this.#refreshToken = response.data.refresh_token;
    } finally {
      release();
    }
  }
}
```

### Graceful Retry Patterns

#### Using axios-retry

```javascript
import axiosRetry from 'axios-retry';

axiosRetry(axios, {
  retries: 3,
  retryDelay: axiosRetry.exponentialDelay,
  retryCondition: (error) => {
    // Retry on network errors or 5xx server errors
    return error.code === 'ECONNABORTED' || 
           (error.response?.status >= 500 && error.response?.status < 600) ||
           error.response?.data?.retryable;
  }
});

// Now all axios requests auto-retry
const response = await axios.get('/api/data');
```

#### Using p-retry (for non-axios promises)

```javascript
import pRetry from 'p-retry';

const result = await pRetry(async () => {
  const response = await fetch('/api/data');
  if (!response.ok) throw new Error(`HTTP ${response.status}`);
  return response.json();
}, {
  retries: 3,
  onFailedAttempt: (error) => {
    console.log(`Attempt ${error.attemptNumber} failed. ${error.retriesLeft} retries left.`);
  }
});
```

**CLI tip**: Add `--max-retries` flag; abort on user interrupt:

```javascript
process.on('SIGINT', () => {
  console.log('\nAborting...');
  process.exit(0);
});
```

### Network Failure Handling

Catch common network errors and set timeouts:

```javascript
// Set global timeout
axios.defaults.timeout = 10000; // 10 seconds

try {
  const response = await axios.get(url);
} catch (error) {
  if (error.code === 'ECONNABORTED') {
    console.error('Request timeout');
  } else if (error.code === 'ECONNRESET') {
    console.error('Connection reset');
  } else if (error.code === 'ETIMEDOUT') {
    console.error('Connection timed out');
  } else if (error.code === 'ENOTFOUND') {
    console.error('DNS lookup failed');
  } else {
    console.error('Network error:', error.message);
  }
}
```

**Fallback to offline mode or cached data**:

```javascript
const NodeCache = require('node-cache');
const cache = new NodeCache({ stdTTL: 3600 }); // 1 hour

async function fetchWithCache(url) {
  const cached = cache.get(url);
  if (cached) return cached;
  
  try {
    const response = await axios.get(url);
    cache.set(url, response.data);
    return response.data;
  } catch (error) {
    if (error.code?.startsWith('E')) {
      console.warn('Network error, using stale cache if available');
      return cache.get(url) || null;
    }
    throw error;
  }
}
```

### Centralized Error Handler

**Create a wrapper service** for all API calls to ensure consistency:

```javascript
class APIService {
  constructor(baseUrl, apiKey) {
    this.client = axios.create({
      baseURL: baseUrl,
      timeout: 10000,
      headers: { 'Authorization': `Bearer ${apiKey}` }
    });
    
    // Add retry logic
    axiosRetry(this.client, {
      retries: 3,
      retryDelay: axiosRetry.exponentialDelay,
      retryCondition: (error) => {
        return axiosRetry.isNetworkOrIdempotentRequestError(error) ||
               error.response?.status === 429;
      }
    });
    
    // Add response interceptor for token refresh
    this.client.interceptors.response.use(
      response => response,
      async error => {
        if (error.response?.status === 401 && !error.config._retry) {
          error.config._retry = true;
          await this.refreshToken();
          return this.client(error.config);
        }
        return Promise.reject(error);
      }
    );
  }
  
  async get(endpoint, options = {}) {
    return this.client.get(endpoint, options);
  }
  
  async post(endpoint, data, options = {}) {
    return this.client.post(endpoint, data, options);
  }
  
  async refreshToken() {
    // Implementation from earlier section
  }
}

// Usage
const api = new APIService('https://api.example.com', process.env.API_KEY);
const data = await api.get('/users/me');
```

### Testing with Mocked Failures

Use `nock` for testing error scenarios:

```bash
npm i -D nock
```

```javascript
const nock = require('nock');

// Mock rate limit
nock('https://api.example.com')
  .get('/data')
  .reply(429, { error: 'Rate limit exceeded' }, {
    'Retry-After': '5'
  });

// Mock network timeout
nock('https://api.example.com')
  .get('/data')
  .replyWithError({ code: 'ETIMEDOUT' });

// Your tests here
```

### Uncaught Exception Handling (CLI Safety)

```javascript
process.on('uncaughtException', (error) => {
  console.error('Uncaught exception:', error);
  // Cleanup (close files, flush logs, etc.)
  process.exit(1);
});

process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled rejection at:', promise, 'reason:', reason);
  process.exit(1);
});
```

---

## Summary & Recommendations

### Technology Stack

| Layer | Recommended Library | Rationale |
|-------|-------------------|-----------|
| **Token Storage** | `keytar` | Cross-platform OS keychain integration |
| **HTTP Client** | `got` or `undici` | Modern APIs, built-in retry, cookie support |
| **Browser Automation** | `playwright` | Headless auth flows, token extraction |
| **Progress UI** | `ora` + `cli-progress` | Spinners for tasks, bars for downloads |
| **Concurrency** | `p-queue` | Rate limit-friendly parallel requests |
| **Retry Logic** | `axios-retry` or `p-retry` | Automatic exponential backoff |
| **OAuth** | `openid-client` | Standard OAuth/OIDC flows |

### Architecture Pattern

```
CLI Command
    ↓
Token Manager (keytar)
    ↓
API Client Wrapper (got/undici + retry logic)
    ↓
Request Pipeline (headers, cookies, TLS)
    ↓
Response Handler
    ↓
Output Pipeline (polling, progress, download)
```

### Key Principles

1. **Security First**: Use OS keychains, never plaintext tokens
2. **Resilience**: Auto-retry with exponential backoff, handle rate limits gracefully
3. **UX**: Show progress for long operations, clear error messages
4. **Browser Fidelity**: Proper headers + TLS fingerprinting for anti-bot evasion
5. **Testability**: Mock network failures, test retry logic

### Next Steps

1. Set up project structure with TypeScript
2. Implement `APIClient` wrapper with token management
3. Add Playwright auth flow with persistent context
4. Build CLI commands using Commander.js
5. Add comprehensive error handling and logging
6. Test with real API endpoints

---

**Research completed:** 2026-02-13 06:39 UTC  
**Total research cost:** $0.018 (5 queries @ sonar-pro)  
**Budget remaining:** $8.49 / $14.00
