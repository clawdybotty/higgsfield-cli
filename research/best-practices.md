# Best Practices for Reverse-Engineering Web App APIs

**Research Date:** February 13, 2026  
**Model:** Perplexity Sonar Pro (High Context)  
**Purpose:** Guide for creating CLI wrappers for undocumented/unofficial web APIs

---

## Table of Contents

1. [API Discovery & Endpoint Mapping](#1-api-discovery--endpoint-mapping)
2. [Authentication Flow Reverse Engineering](#2-authentication-flow-reverse-engineering)
3. [Anti-Bot Detection & Circumvention](#3-anti-bot-detection--circumvention)
4. [CLI Architecture Patterns](#4-cli-architecture-patterns)
5. [Legal & Terms of Service Considerations](#5-legal--terms-of-service-considerations)

---

## 1. API Discovery & Endpoint Mapping

### 1.1 Browser DevTools Network Analysis

The **Network tab** in Chrome/Firefox DevTools is the first-line tool for API discovery:

**Workflow:**
1. Open DevTools (F12) → Network tab
2. Filter by **XHR/Fetch** to isolate AJAX requests
3. Perform actions in the web app while monitoring requests
4. Right-click requests → **Copy as cURL** or **Copy as fetch**
5. Inspect request/response headers, payloads, and authentication

**Key Patterns to Look For:**
- **REST APIs**: Predictable URL structures like `/api/v1/resource/{id}` with standard HTTP methods (GET, POST, PUT, DELETE)
- **GraphQL APIs**: Single endpoint (e.g., `/graphql`) with POST requests containing `query` or `mutation` fields
- **JSON responses** with consistent schemas
- **Authentication headers**: `Authorization: Bearer <token>`, cookies, or custom headers

**HAR File Export:**
Export network traffic as HAR (HTTP Archive) files for offline analysis:
```bash
# DevTools → Network → Right-click → Save all as HAR with content
# Parse with tools like:
cat traffic.har | jq '.log.entries[] | {url: .request.url, method: .request.method}'
```

### 1.2 MITM Proxies (mitmproxy, Charles Proxy)

For **deeper inspection** and **traffic modification**, use MITM proxies:

**mitmproxy Setup:**
```bash
# Install
pip install mitmproxy

# Run interactive proxy
mitmproxy -p 8080

# Configure browser/system proxy to localhost:8080
# Install mitmproxy CA cert for HTTPS inspection

# Save traffic to file
mitmdump -w flows.mitm
# Replay later
mitmproxy -r flows.mitm
```

**Advantages over DevTools:**
- Captures traffic from **any application** (not just browsers)
- **Modify requests/responses** on-the-fly for testing
- **Replay traffic** exactly
- Script custom logic with Python addons

**Charles Proxy Alternative:**
- GUI-based, ideal for non-technical stakeholders
- Excellent for mobile app reverse engineering (configure phone proxy)
- Built-in JSON/XML tree viewers

### 1.3 Playwright/Puppeteer Programmatic Capture

Automate API discovery by intercepting requests programmatically:

**Playwright Example (Node.js):**
```javascript
const { chromium } = require('playwright');

(async () => {
  const browser = await chromium.launch();
  const context = await browser.newContext();
  const page = await context.newPage();

  // Intercept all requests
  const endpoints = new Set();
  page.on('request', request => {
    const url = request.url();
    if (url.includes('/api/') || url.includes('/graphql')) {
      endpoints.add(`${request.method()} ${url}`);
      console.log(`[${request.method()}] ${url}`);
      console.log('Headers:', request.headers());
      console.log('Body:', request.postData());
    }
  });

  // Perform actions
  await page.goto('https://example.com');
  await page.click('button#load-data');
  await page.waitForTimeout(2000);

  console.log('\nDiscovered endpoints:');
  endpoints.forEach(e => console.log(e));

  await browser.close();
})();
```

**Use Cases:**
- Automate repetitive endpoint discovery
- Test API changes during site updates
- Build automated scrapers that adapt to API evolution

### 1.4 Code-Based Endpoint Extraction

Search JavaScript source files for embedded API endpoints:

```bash
# Extract all URLs from JS files
find . -type f -name "*.js" | jsluice urls | jq -r '.url' | sort -u

# Search for API endpoint patterns
find . -type f -name "*.js" -exec grep -H "API_ENDPOINT\|fetch(\|axios\." {} \;

# Look for GraphQL schema definitions
grep -r "query\|mutation\|subscription" --include="*.js" .
```

**Tools:**
- **jsluice**: Extract URLs, secrets, and endpoints from JavaScript
- **LinkFinder**: Python tool to find endpoints in JS files
- **Burp Suite**: Professional-grade scanner with JS analysis

### 1.5 Practical Workflow Summary

**Complete Reverse-Engineering Workflow:**

1. **Initial Discovery** (Browser DevTools)
   - Perform all user actions while recording Network tab
   - Export as HAR file
   - Identify authentication mechanism

2. **Deep Capture** (MITM Proxy)
   - Route traffic through mitmproxy
   - Capture complete request/response cycles
   - Validate findings and discover hidden endpoints

3. **Documentation** (HAR Analysis)
   - Parse HAR with tools like Swagger/Postman
   - Generate API documentation/mocks
   - Extract authentication details and data schemas

4. **Automation** (Playwright)
   - Write scripts to programmatically capture traffic
   - Test against stub API in controlled environment
   - Validate before deploying

5. **Code Analysis** (JS Extraction)
   - Extract static endpoints from source
   - Cross-reference with captured traffic
   - Identify endpoint patterns and versioning

---

## 2. Authentication Flow Reverse Engineering

### 2.1 OAuth2 Flow Interception

OAuth2 flows involve multiple redirects and token exchanges. Intercept them using browser DevTools or proxies.

**Common OAuth2 Flow:**
```
1. User clicks "Login with X"
2. Redirect to provider: https://provider.com/oauth/authorize?client_id=...&redirect_uri=...
3. User authorizes
4. Redirect back: https://app.com/callback?code=AUTH_CODE
5. App exchanges code for token: POST /oauth/token with code + client_secret
6. Response: {"access_token": "...", "refresh_token": "...", "expires_in": 3600}
```

**Interception Strategy:**
```python
# In DevTools or mitmproxy, capture:
# 1. Initial authorization URL (client_id, redirect_uri, scope, state)
# 2. Callback with authorization code
# 3. Token exchange request (client_secret often visible!)

# Replicate in CLI:
import requests

auth_url = "https://provider.com/oauth/authorize"
params = {
    "client_id": "captured_client_id",
    "redirect_uri": "http://localhost:8080/callback",  # Start local server
    "scope": "read write",
    "response_type": "code",
    "state": "random_state"
}

# Open browser, capture code from redirect
print(f"Visit: {auth_url}?{urlencode(params)}")
code = input("Enter authorization code: ")

# Exchange for tokens
token_response = requests.post("https://provider.com/oauth/token", data={
    "grant_type": "authorization_code",
    "code": code,
    "redirect_uri": "http://localhost:8080/callback",
    "client_id": "captured_client_id",
    "client_secret": "captured_client_secret"  # Often in JS or network traffic
})

tokens = token_response.json()
print(f"Access Token: {tokens['access_token']}")
print(f"Refresh Token: {tokens['refresh_token']}")
```

**Finding client_secret:**
- Check browser network traffic (sometimes sent in plain text)
- Search JavaScript bundles for hardcoded values
- Use mitmproxy to intercept HTTPS traffic
- Note: Many modern SPAs use PKCE (Proof Key for Code Exchange) to avoid client_secret

### 2.2 JWT Token Extraction & Refresh

**JWT Structure:**
JWTs have three parts: `header.payload.signature` (Base64-encoded)

**Extraction from Browser:**
```python
# Method 1: From localStorage/sessionStorage
# In browser console:
# localStorage.getItem('access_token')
# sessionStorage.getItem('jwt')

# Method 2: From cookies
import browser_cookie3

cookies = browser_cookie3.chrome(domain_name='example.com')
for cookie in cookies:
    if 'token' in cookie.name.lower():
        print(f"{cookie.name}: {cookie.value}")

# Method 3: Intercept from Network tab
# Look for Set-Cookie headers or JSON responses with tokens
```

**Token Refresh Logic:**
```python
import requests
import json
import time
from pathlib import Path

class TokenManager:
    def __init__(self, token_file='~/.config/app/tokens.json'):
        self.token_file = Path(token_file).expanduser()
        self.tokens = self._load_tokens()
    
    def _load_tokens(self):
        if self.token_file.exists():
            return json.loads(self.token_file.read_text())
        return {}
    
    def _save_tokens(self):
        self.token_file.parent.mkdir(parents=True, exist_ok=True)
        self.token_file.write_text(json.dumps(self.tokens, indent=2))
        self.token_file.chmod(0o600)  # Secure permissions
    
    def get_access_token(self):
        # Check if token expired
        if time.time() >= self.tokens.get('expires_at', 0):
            self._refresh_token()
        return self.tokens['access_token']
    
    def _refresh_token(self):
        response = requests.post('https://api.example.com/oauth/token', data={
            'grant_type': 'refresh_token',
            'refresh_token': self.tokens['refresh_token'],
            'client_id': 'CLIENT_ID'
        })
        
        data = response.json()
        self.tokens = {
            'access_token': data['access_token'],
            'refresh_token': data.get('refresh_token', self.tokens['refresh_token']),
            'expires_at': time.time() + data['expires_in']
        }
        self._save_tokens()

# Usage
token_mgr = TokenManager()
headers = {'Authorization': f'Bearer {token_mgr.get_access_token()}'}
response = requests.get('https://api.example.com/data', headers=headers)
```

### 2.3 Session Cookie Management

For cookie-based authentication (non-token):

**Using requests.Session:**
```python
import requests

session = requests.Session()

# Login
login_data = {"username": "user", "password": "pass"}  # From intercepted form
session.post("https://example.com/login", data=login_data)

# Session automatically handles cookies
response = session.get("https://example.com/profile")  # Cookies auto-sent
print(response.text)

# Save session for later
import pickle
with open('session.pkl', 'wb') as f:
    pickle.dump(session.cookies, f)

# Load session
with open('session.pkl', 'rb') as f:
    session.cookies.update(pickle.load(f))
```

### 2.4 CSRF Token Handling

CSRF tokens (e.g., `_csrf`, `csrfmiddlewaretoken`) appear in forms or headers:

**Extraction Strategy:**
```python
from bs4 import BeautifulSoup
import requests

session = requests.Session()

# Fetch page with CSRF token
response = session.get("https://example.com/login")
soup = BeautifulSoup(response.text, 'html.parser')

# Extract from hidden input
csrf_token = soup.find('input', {'name': '_csrf'})['value']

# Or from meta tag
# csrf_token = soup.find('meta', {'name': 'csrf-token'})['content']

# Submit with CSRF
login_data = {
    "username": "user",
    "password": "pass",
    "_csrf": csrf_token
}
session.post("https://example.com/login", data=login_data)
```

**Header-Based CSRF:**
```python
# Some APIs expect CSRF in headers
headers = {
    'X-CSRF-Token': csrf_token,
    'X-Requested-With': 'XMLHttpRequest'  # Often required
}
session.post("https://example.com/api/action", headers=headers, json=data)
```

### 2.5 Secure Credential Storage

**Never store credentials in plaintext.** Use OS keychains:

**Python (keyring library):**
```python
import keyring
import getpass

SERVICE_NAME = "example-app"

# Store once (prompts for password)
def store_credentials():
    username = input("Username: ")
    password = getpass.getpass("Password: ")
    keyring.set_password(SERVICE_NAME, username, password)
    print("Credentials stored securely")

# Retrieve securely
def get_credentials():
    username = input("Username: ")
    password = keyring.get_password(SERVICE_NAME, username)
    if password is None:
        raise ValueError("No credentials found. Run store_credentials() first.")
    return {"username": username, "password": password}

# Use in CLI
creds = get_credentials()
session.post("https://example.com/login", data=creds)
```

**For tokens/JWTs:**
```python
# Store encrypted in config file with restricted permissions
import json
from pathlib import Path
from cryptography.fernet import Fernet

class SecureTokenStorage:
    def __init__(self, config_dir='~/.config/app'):
        self.config_dir = Path(config_dir).expanduser()
        self.config_dir.mkdir(parents=True, exist_ok=True)
        self.key_file = self.config_dir / '.key'
        self.token_file = self.config_dir / 'tokens.enc'
        self._ensure_key()
    
    def _ensure_key(self):
        if not self.key_file.exists():
            key = Fernet.generate_key()
            self.key_file.write_bytes(key)
            self.key_file.chmod(0o600)
    
    def save_tokens(self, tokens):
        key = self.key_file.read_bytes()
        cipher = Fernet(key)
        encrypted = cipher.encrypt(json.dumps(tokens).encode())
        self.token_file.write_bytes(encrypted)
        self.token_file.chmod(0o600)
    
    def load_tokens(self):
        if not self.token_file.exists():
            return None
        key = self.key_file.read_bytes()
        cipher = Fernet(key)
        decrypted = cipher.decrypt(self.token_file.read_bytes())
        return json.loads(decrypted)
```

**Environment Variables (for CI/automation):**
```bash
# Never commit to git
export APP_ACCESS_TOKEN="token_here"

# In Python
import os
token = os.environ.get('APP_ACCESS_TOKEN')
if not token:
    raise ValueError("APP_ACCESS_TOKEN not set")
```

---

## 3. Anti-Bot Detection & Circumvention

### 3.1 Modern Anti-Bot Detection Techniques

Websites use layered defenses targeting specific automation fingerprints:

#### Browser Fingerprinting
Collects unique identifiers:
- **Canvas/WebGL rendering**: Generates images that vary by GPU/driver
- **Audio context**: AudioContext fingerprinting based on hardware
- **Font enumeration**: Installed fonts leak OS/user profile
- **Screen resolution, color depth, timezone**
- **API availability**: Check for navigator.webdriver, missing APIs
- **Hardware info**: GPU, audio devices, CPU cores

**Detection Method:**
```javascript
// Sites check for:
if (navigator.webdriver) {
  console.log("Automation detected!");
}

// Headless browser artifacts
if (!window.chrome || !window.chrome.runtime) {
  // Likely headless Chrome
}
```

**AI-Based Fleet Detection:**
Modern systems (DataDome, PerimeterX) analyze patterns across sessions to detect shared fingerprints from automation fleets.

#### TLS Fingerprinting (JA3)
Inspects SSL/TLS handshake signatures unique to HTTP libraries:
- **Python requests**: Non-browser-like TLS fingerprint
- **curl**: Different from browsers
- **Real browsers**: Chrome, Firefox have distinct signatures

**Detection occurs before HTTP request reaches server**, making it extremely effective.

#### Header Analysis
Validates:
- **User-Agent strings**: Check for default automation strings (e.g., "HeadlessChrome")
- **Header order**: Browsers send headers in specific order
- **Header presence**: Missing `Accept-Language`, `Referer`, etc.
- **Header combinations**: Mismatched UA with incompatible headers

#### Behavioral Detection
Monitors:
- **Mouse movements**: Natural curves vs. straight lines
- **Scroll patterns**: Human scrolling is erratic
- **Typing cadence**: Natural pauses vs. instant input
- **Timing between actions**: Bots are too fast or too uniform
- **Interaction sequences**: Humans don't click in perfect patterns

**AI models** score these signals for human-like variability vs. robotic predictability.

### 3.2 Anti-Bot Systems (Examples)

| System | Techniques | Strength |
|--------|-----------|----------|
| **Cloudflare Turnstile** | Browser challenges, TLS fingerprinting, behavioral analysis | High (fewer false positives than CAPTCHA) |
| **DataDome** | Real-time AI, device fingerprinting, bot fleet detection | Very High |
| **HUMAN (PerimeterX)** | ML models, behavioral biometrics, network analysis | Very High |
| **Reblaze** | Multi-layer defense, signature analysis | High |
| **Netacea** | Intent analysis, threat intelligence | High |

### 3.3 Circumvention Tools (Ethical Use Only)

⚠️ **Use these tools only with permission or for authorized testing.**

#### curl-impersonate / curl_cffi
Spoofs TLS handshakes and headers to match real browsers at the network level:

```bash
# Install curl-impersonate
brew install curl-impersonate  # macOS

# Impersonate Chrome
curl_chrome116 https://example.com

# Python with curl_cffi
pip install curl_cffi

from curl_cffi import requests
response = requests.get("https://example.com", impersonate="chrome116")
```

**Targets:** TLS fingerprinting, header analysis

#### playwright-extra with stealth plugin
Patches Playwright APIs to hide automation flags:

```javascript
const { chromium } = require('playwright-extra');
const stealth = require('puppeteer-extra-plugin-stealth')();

(async () => {
  const browser = await chromium.launch({
    headless: false  // Many sites detect headless
  });
  
  const page = await browser.newPage();
  
  // Stealth plugin removes navigator.webdriver, patches APIs
  await stealth.onPageCreated(page);
  
  await page.goto('https://example.com');
  
  // Behave like human
  await page.mouse.move(100, 200, { steps: 10 });
  await page.waitForTimeout(Math.random() * 1000 + 500);
  await page.click('button');
  
  await browser.close();
})();
```

**Targets:** Browser fingerprinting, JS challenges

#### undetected-chromedriver
Modifies ChromeDriver to remove WebDriver properties:

```python
import undetected_chromedriver as uc

options = uc.ChromeOptions()
options.add_argument('--disable-blink-features=AutomationControlled')

driver = uc.Chrome(options=options)
driver.get('https://example.com')

# navigator.webdriver is now undefined
# ChromeDriver automation flags are obfuscated
```

**Targets:** Browser fingerprinting, automation markers

#### Other Tools
- **Camoufox**: C++-level Firefox spoofing for fingerprints
- **Nodriver**: Chromium-based with human behavior simulation
- **Residential proxies**: Rotate IPs to avoid IP-based blocking

### 3.4 Legitimate Automation vs. Bot Behavior

| Aspect | Legitimate Automation | Bot Behavior |
|--------|----------------------|--------------|
| **Speed/Timing** | Variable delays, natural pauses, erratic scrolls | Uniform, rapid actions without hesitation |
| **Fingerprint** | Diverse, consistent signals (e.g., real OS/browser combos) | Shared/static fingerprints (e.g., Linux-only fleets) |
| **Network** | Browser-like TLS/headers, geolocation-matched IPs | Datacenter TLS, mismatched headers/IPs |
| **Behavior** | Realistic mouse/typing patterns via simulation | Predictable, non-interactive sequences |
| **Scale** | Low volume, session persistence | High-volume bursts, fingerprint reuse across IPs |

### 3.5 Ethical Approaches

**Always prioritize these practices:**

1. **Transparency**
   - Disclose automation in User-Agent or custom headers
   - Respect `robots.txt`
   - Seek explicit permission for scraping

2. **Rate Limiting & Variability**
   - Introduce human-like delays (random 0.5-3 seconds)
   - Random paths through site
   - Low request volumes to avoid server overload

3. **Residential Proxies & Real Browsers**
   - Use ethical proxy providers (not compromised devices)
   - Rotate IPs naturally
   - Use real browsers with anti-detect browsers (GoLogin, Multilogin) only for permitted tasks

4. **CAPTCHA/Challenge Handling**
   - Solve challenges ethically or reduce triggers via legitimate patterns
   - Avoid black-box CAPTCHA solvers (often violate ToS)

5. **Prefer Official APIs**
   - Use official APIs when available
   - Request partnerships or API access
   - Use human-verified data over evasion

**Remember:** Violating ToS risks legal action and permanent bans, even if technically successful.

---

## 4. CLI Architecture Patterns

### 4.1 Token Storage Strategies

**Hierarchical Configuration:**
```
Priority: CLI flags > Environment vars > Config file > Defaults
```

**Example (Python with click):**
```python
import click
import json
from pathlib import Path

CONFIG_FILE = Path.home() / '.config' / 'app' / 'config.json'

def load_config():
    if CONFIG_FILE.exists():
        return json.loads(CONFIG_FILE.read_text())
    return {}

@click.command()
@click.option('--token', envvar='APP_TOKEN', help='API token')
@click.option('--config', type=click.Path(), help='Config file path')
def main(token, config):
    # Priority: CLI flag > env var > config file
    if not token:
        cfg = load_config() if not config else json.loads(Path(config).read_text())
        token = cfg.get('token')
    
    if not token:
        raise click.ClickException("No token provided. Use --token, APP_TOKEN env, or config file.")
    
    # Use token
    click.echo(f"Using token: {token[:10]}...")

if __name__ == '__main__':
    main()
```

**Secure Storage (repeat from section 2.5):**
- Use `keyring` for passwords
- Encrypt tokens in config files
- Set file permissions to `0o600`
- Never commit secrets to git

### 4.2 Retry Logic with Exponential Backoff

**Using tenacity library:**
```python
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type
import requests

class APIError(Exception):
    pass

@retry(
    stop=stop_after_attempt(5),
    wait=wait_exponential(multiplier=1, min=1, max=60),
    retry=retry_if_exception_type((requests.exceptions.RequestException, APIError)),
    reraise=True
)
def api_request(url, **kwargs):
    response = requests.get(url, **kwargs)
    
    if response.status_code == 429:  # Rate limited
        retry_after = int(response.headers.get('Retry-After', 60))
        raise APIError(f"Rate limited. Retry after {retry_after}s")
    
    if response.status_code >= 500:  # Server error
        raise APIError(f"Server error: {response.status_code}")
    
    response.raise_for_status()
    return response.json()

# Usage
try:
    data = api_request('https://api.example.com/data', headers=headers)
except Exception as e:
    print(f"Failed after retries: {e}")
```

**Manual Implementation:**
```python
import time
import requests

def api_request_with_retry(url, max_retries=5, **kwargs):
    for attempt in range(max_retries):
        try:
            response = requests.get(url, **kwargs)
            
            if response.status_code == 429:
                retry_after = int(response.headers.get('Retry-After', 2 ** attempt))
                print(f"Rate limited. Waiting {retry_after}s...")
                time.sleep(retry_after)
                continue
            
            if response.status_code >= 500:
                wait_time = min(2 ** attempt, 60)  # Exponential backoff, max 60s
                print(f"Server error. Retrying in {wait_time}s...")
                time.sleep(wait_time)
                continue
            
            response.raise_for_status()
            return response.json()
            
        except requests.exceptions.RequestException as e:
            if attempt == max_retries - 1:
                raise
            wait_time = min(2 ** attempt, 60)
            print(f"Request failed: {e}. Retrying in {wait_time}s...")
            time.sleep(wait_time)
    
    raise Exception("Max retries exceeded")
```

### 4.3 Rate Limiting and Throttling

**Using ratelimit library:**
```python
from ratelimit import limits, sleep_and_retry
import requests

# Allow 10 calls per minute
@sleep_and_retry
@limits(calls=10, period=60)
def api_call(url):
    return requests.get(url).json()

# Usage
for i in range(100):
    data = api_call('https://api.example.com/data')  # Automatically throttled
```

**Token Bucket Algorithm:**
```python
import time

class TokenBucket:
    def __init__(self, rate, capacity):
        self.rate = rate  # tokens per second
        self.capacity = capacity
        self.tokens = capacity
        self.last_update = time.time()
    
    def consume(self, tokens=1):
        now = time.time()
        elapsed = now - self.last_update
        self.tokens = min(self.capacity, self.tokens + elapsed * self.rate)
        self.last_update = now
        
        if self.tokens >= tokens:
            self.tokens -= tokens
            return True
        
        # Wait until enough tokens
        wait_time = (tokens - self.tokens) / self.rate
        time.sleep(wait_time)
        self.tokens = 0
        self.last_update = time.time()
        return True

# Usage: 5 requests per second
bucket = TokenBucket(rate=5, capacity=10)

for i in range(100):
    bucket.consume()
    requests.get('https://api.example.com/data')
```

### 4.4 Error Handling

**Robust error handling pattern:**
```python
import requests
import sys
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class APIClient:
    def __init__(self, base_url, token):
        self.base_url = base_url
        self.session = requests.Session()
        self.session.headers.update({'Authorization': f'Bearer {token}'})
    
    def request(self, method, endpoint, **kwargs):
        url = f"{self.base_url}/{endpoint}"
        
        try:
            response = self.session.request(method, url, **kwargs)
            
            # Handle specific status codes
            if response.status_code == 401:
                logger.error("Authentication failed. Check your token.")
                sys.exit(1)
            elif response.status_code == 403:
                logger.error("Access forbidden. Insufficient permissions.")
                sys.exit(1)
            elif response.status_code == 404:
                logger.error(f"Endpoint not found: {endpoint}")
                return None
            elif response.status_code == 429:
                retry_after = response.headers.get('Retry-After', '60')
                logger.warning(f"Rate limited. Retry after {retry_after}s")
                raise RateLimitError(retry_after)
            elif response.status_code >= 500:
                logger.error(f"Server error: {response.status_code}")
                raise ServerError(response.status_code)
            
            response.raise_for_status()
            return response.json()
            
        except requests.exceptions.ConnectionError:
            logger.error("Connection failed. Check your internet connection.")
            sys.exit(1)
        except requests.exceptions.Timeout:
            logger.error("Request timed out.")
            raise
        except requests.exceptions.JSONDecodeError:
            logger.error("Invalid JSON response")
            return response.text

class RateLimitError(Exception):
    pass

class ServerError(Exception):
    pass
```

### 4.5 Session Persistence

**Save/restore sessions:**
```python
import pickle
import requests
from pathlib import Path

SESSION_FILE = Path.home() / '.config' / 'app' / 'session.pkl'

class PersistentSession:
    def __init__(self):
        self.session = self._load_session()
    
    def _load_session(self):
        if SESSION_FILE.exists():
            with open(SESSION_FILE, 'rb') as f:
                return pickle.load(f)
        return requests.Session()
    
    def save(self):
        SESSION_FILE.parent.mkdir(parents=True, exist_ok=True)
        with open(SESSION_FILE, 'wb') as f:
            pickle.dump(self.session, f)
        SESSION_FILE.chmod(0o600)
    
    def login(self, username, password):
        response = self.session.post('https://example.com/login', data={
            'username': username,
            'password': password
        })
        response.raise_for_status()
        self.save()
        return response
    
    def get(self, url, **kwargs):
        return self.session.get(url, **kwargs)

# Usage
session = PersistentSession()
# Login once
# session.login('user', 'pass')

# Subsequent runs reuse cookies
response = session.get('https://example.com/protected')
```

### 4.6 Configuration Management

**TOML config example:**
```toml
# ~/.config/app/config.toml
[auth]
token = "your_token_here"

[api]
base_url = "https://api.example.com"
timeout = 30
retry_attempts = 5

[rate_limit]
requests_per_minute = 60
```

**Loading with tomli/tomllib:**
```python
import tomllib  # Python 3.11+ (or tomli for older versions)
from pathlib import Path
import click

CONFIG_FILE = Path.home() / '.config' / 'app' / 'config.toml'

def load_config():
    if CONFIG_FILE.exists():
        with open(CONFIG_FILE, 'rb') as f:
            return tomllib.load(f)
    return {
        'api': {'base_url': 'https://api.example.com', 'timeout': 30},
        'rate_limit': {'requests_per_minute': 60}
    }

@click.group()
@click.pass_context
def cli(ctx):
    ctx.obj = load_config()

@cli.command()
@click.pass_obj
def status(config):
    click.echo(f"API: {config['api']['base_url']}")
    click.echo(f"Rate limit: {config['rate_limit']['requests_per_minute']}/min")

if __name__ == '__main__':
    cli()
```

### 4.7 Testing Strategies

**Unit Tests with Mock Responses:**
```python
import pytest
from unittest.mock import Mock, patch
import requests

@pytest.fixture
def mock_response():
    mock = Mock()
    mock.status_code = 200
    mock.json.return_value = {'data': 'test'}
    return mock

def test_api_request(mock_response):
    with patch('requests.get', return_value=mock_response):
        result = api_request('https://api.example.com/data')
        assert result == {'data': 'test'}

def test_rate_limit_handling():
    mock = Mock()
    mock.status_code = 429
    mock.headers = {'Retry-After': '2'}
    
    with patch('requests.get', return_value=mock):
        with pytest.raises(RateLimitError):
            api_request('https://api.example.com/data')
```

**Integration Tests with VCR (Record/Replay):**
```python
import pytest
import vcr

my_vcr = vcr.VCR(
    cassette_library_dir='tests/cassettes',
    record_mode='once',  # Record once, then replay
    match_on=['uri', 'method']
)

@my_vcr.use_cassette('get_data.yaml')
def test_get_data():
    # First run: records real API response
    # Subsequent runs: replays from cassette
    response = requests.get('https://api.example.com/data')
    assert response.status_code == 200
    assert 'data' in response.json()
```

### 4.8 Architectural Patterns

**Modular Layer Design:**
```
cli_tool/
├── cli/                 # CLI interface (Click/Typer)
│   ├── __init__.py
│   └── commands.py
├── core/                # Business logic
│   ├── __init__.py
│   ├── api_client.py    # API wrapper
│   └── models.py        # Data models
├── auth/                # Authentication
│   ├── __init__.py
│   ├── token_manager.py
│   └── storage.py
├── utils/               # Utilities
│   ├── __init__.py
│   ├── retry.py
│   └── rate_limit.py
└── tests/               # Tests
    ├── test_api.py
    └── cassettes/
```

**Example from yt-dlp:**
- **Extractor pattern**: Plugin-based architecture for different sites
- Each extractor handles site-specific API quirks
- Fallback mechanisms for API changes
- Extensive unit tests with mocked responses

**Key Principles:**
1. **Separation of concerns**: CLI ≠ Business Logic ≠ API Client
2. **Dependency injection**: Pass clients/configs, don't hardcode
3. **Resilience**: Compose retry/backoff/rate-limit as decorators
4. **Testability**: Mock external APIs, test commands separately
5. **Documentation**: Include usage examples and API change notes

---

## 5. Legal & Terms of Service Considerations

### 5.1 Key Legal Frameworks

Reverse-engineering web APIs carries legal risks under multiple frameworks:

#### DMCA Section 1201 (Anti-Circumvention)

**Prohibition:**
- Bypassing **Technological Protection Measures (TPMs)** that control access to copyrighted works
- Examples: Authentication handshakes, encryption, obfuscation

**Exception (Section 1201(f) - Interoperability):**
Allows circumvention to identify and analyze elements necessary for **interoperability** with independently created programs, provided:
1. Information is used **only for interoperability**
2. No disclosure enabling infringement
3. Lawful possession of the program
4. Non-infringing software result

**Case Law:**
- **MDY Industries v. Blizzard (BnetD)**: Programmers emulated Battle.net authentication server. Court found violation because emulation went beyond minimal needs for interoperability and enabled infringing play.

**Takeaway:**
- **Legal**: Extracting protocol specs for compatibility (e.g., building a CLI client)
- **Risky**: Emulating authentication servers, bypassing DRM

#### CFAA (Computer Fraud and Abuse Act)

**Prohibition:**
Criminalizes unauthorized access to computers.

**Key Case - Van Buren v. United States (2021):**
Supreme Court narrowed CFAA, holding that "exceeds authorized access" requires **breaching technical barriers** (e.g., hacking past login gates), not merely violating use restrictions like ToS.

**Implications:**
- **Legal**: Inspecting publicly accessible APIs without credentials
- **Risky**: Using scraped/stolen credentials, bypassing rate limits, accessing private APIs without permission

**Note:** Post-*Van Buren*, simple ToS violations don't trigger CFAA, but using unauthorized credentials still does.

#### Copyright Law & Fair Use (17 U.S.C. § 107)

**Fair Use Factors:**
1. Purpose (transformative? commercial?)
2. Nature of copyrighted work
3. Amount used
4. Market effect

**Case Law:**
- **Atari Games v. Nintendo**: Microscopic analysis and disassembly to extract unprotected ideas (protocol specs) = **fair use**
- Creating substantially similar derivatives or clones harming the market = **not fair use**

**Takeaway:**
- **Legal**: Extracting API specs for compatibility
- **Risky**: Copying substantial code, creating competitive clones

#### Contract Law & Terms of Service

**Enforceability:**
ToS/EULAs are contracts if users "click-agree." Many explicitly ban reverse engineering.

**Risks:**
- Breach of contract claims → damages or injunctions
- Even if DMCA/CFAA don't apply, ToS violations can trigger civil liability
- Fourth Circuit rulings favor narrow ToS definitions (e.g., "reverse engineering" = decompiling only)

**Takeaway:**
- Read ToS carefully
- "Click-wrap" agreements are generally enforceable
- Violations = civil liability, even if no criminal charges

#### Trade Secrets

**Protection:**
APIs may be trade secrets if not publicly disclosed.

**Exception:**
Reverse engineering is a **lawful means of discovery** under trade secret law, not misappropriation—unless via:
- NDAs/confidentiality agreements
- Stolen credentials or improper access

**Takeaway:**
- **Legal**: Independent discovery via public observation
- **Risky**: Using leaked docs, stolen credentials, or violating NDAs

### 5.2 Scraping Case Law: hiQ Labs v. LinkedIn

**Facts:**
- hiQ scraped publicly accessible LinkedIn profiles
- LinkedIn claimed CFAA violation and sent cease-and-desist
- Ninth Circuit (affirmed by Supreme Court denial, 2022): Scraping **publicly accessible** data does NOT violate CFAA

**Key Holdings:**
1. No "access" barrier = no CFAA violation
2. Public data scraping is lawful (under CFAA)
3. ToS violations alone don't create CFAA liability

**Implications:**
- **Legal**: Reverse-engineering public APIs without authentication
- **Risky**: Private APIs requiring login, even if ToS forbids it

**Limitations:**
- Commercial competition may invite unfair competition claims
- Other jurisdictions may differ (EU, state laws)

### 5.3 What Makes It Legally Defensible vs. Risky?

| Factor | Defensible (Low Risk) | Risky (High Risk) |
|--------|----------------------|-------------------|
| **Access Method** | Public endpoints, no TPM bypass | Private APIs with login, circumventing encryption/rate limits |
| **Purpose** | Interoperability with independent CLI tool | Cloning features, competitive substitutes |
| **Contracts** | No ToS agreement, or public data (per hiQ) | Violates clicked ToS/EULA banning reverse engineering |
| **Output** | Non-infringing wrapper, no infringement enablement | Shares internals, tools enabling infringement, derivative works |
| **Intent** | Compatibility, research, personal use | Commercial harm, market substitution |
| **Examples** | hiQ public scraping, Atari fair use analysis | MDY BnetD server emulation, DRM bypass clones |

### 5.4 Ethical Considerations

Even if legally defensible, consider:

1. **Transparency**
   - Disclose wrapper as unofficial
   - Don't misrepresent as official client

2. **Server Load**
   - Respect robots.txt
   - Implement rate limiting
   - Avoid DDoS-like behavior

3. **Competitive Harm**
   - Are you enabling spam/abuse?
   - Is this a competitive substitute harming the provider?

4. **Reputational Risk**
   - Providers may pursue costly litigation regardless of merits
   - Public backlash possible

5. **Terms of Service**
   - Even if not legally enforceable in all cases, violating ToS damages trust
   - Consider seeking permission or API partnership

### 5.5 Practical Recommendations

**Before Building:**
1. **Read the ToS** - Does it explicitly ban reverse engineering?
2. **Check for official API** - Use it if available
3. **Assess risk** - Is the data public? Does your tool compete?
4. **Consult counsel** - For commercial projects or high-stakes use

**Design Choices:**
1. **Minimize circumvention** - Avoid bypassing technical protections
2. **Respect rate limits** - Don't overload servers
3. **Use public data** - Prefer publicly accessible endpoints
4. **Document independently** - Don't copy proprietary docs

**If Challenged:**
1. **Cease immediately** - Upon receiving cease-and-desist
2. **Consult lawyer** - Don't respond without legal advice
3. **Consider settlement** - Litigation is expensive, even if you're right

### 5.6 International Considerations

**Note:** Laws vary by jurisdiction:
- **EU**: Database Directive provides additional protections
- **UK**: Different fair dealing exceptions
- **Other countries**: Varying ToS enforceability and computer fraud laws

Always consult local counsel for non-US projects.

---

## Summary & Best Practices Checklist

### ✅ API Discovery
- [ ] Use browser DevTools for initial endpoint mapping
- [ ] Capture traffic with mitmproxy/Charles for deep analysis
- [ ] Export HAR files for documentation
- [ ] Automate with Playwright for ongoing monitoring
- [ ] Extract static endpoints from JavaScript source

### ✅ Authentication
- [ ] Intercept OAuth flows to capture tokens
- [ ] Implement automatic token refresh logic
- [ ] Use `requests.Session` for cookie-based auth
- [ ] Handle CSRF tokens from forms/headers
- [ ] Store credentials securely (keyring, encrypted files, never plaintext)

### ✅ Anti-Bot Evasion (Ethical Use Only)
- [ ] Understand fingerprinting techniques (browser, TLS, behavioral)
- [ ] Use tools like curl-impersonate only with permission
- [ ] Implement human-like behavior (random delays, mouse movements)
- [ ] Respect robots.txt and rate limits
- [ ] Prefer official APIs over circumvention

### ✅ CLI Architecture
- [ ] Implement hierarchical configuration (CLI > env > config > defaults)
- [ ] Add retry logic with exponential backoff
- [ ] Implement rate limiting (token bucket or library)
- [ ] Handle errors gracefully with specific status code logic
- [ ] Persist sessions securely
- [ ] Write unit tests with mocked responses
- [ ] Use VCR/cassettes for integration tests

### ✅ Legal Compliance
- [ ] Read and understand ToS before building
- [ ] Prefer public APIs and official access methods
- [ ] Ensure purpose is interoperability, not competitive substitution
- [ ] Avoid circumventing technical protection measures
- [ ] Document independently (don't copy proprietary docs)
- [ ] Implement respectful rate limiting
- [ ] Disclose tool as unofficial
- [ ] Consult legal counsel for commercial projects

---

## References & Further Reading

### Tools
- **mitmproxy**: https://mitmproxy.org/
- **Playwright**: https://playwright.dev/
- **curl-impersonate**: https://github.com/lwthiker/curl-impersonate
- **undetected-chromedriver**: https://github.com/ultrafunkamsterdam/undetected-chromedriver

### Libraries (Python)
- **requests**: HTTP client
- **keyring**: Secure credential storage
- **tenacity**: Retry logic
- **ratelimit**: Rate limiting
- **click/typer**: CLI frameworks
- **pytest-vcr**: Record/replay HTTP interactions

### Legal Resources
- **EFF**: https://www.eff.org/ (digital rights, legal analysis)
- **Van Buren v. United States**: Supreme Court CFAA decision
- **hiQ Labs v. LinkedIn**: Ninth Circuit scraping case

### Case Studies
- **yt-dlp**: https://github.com/yt-dlp/yt-dlp (video download CLI)
- **gallery-dl**: https://github.com/mikf/gallery-dl (image scraping)
- **instaloader**: https://github.com/instaloader/instaloader (Instagram CLI)

---

**Document compiled from Perplexity Sonar Pro research**  
**Total research cost: ~$0.03**  
**Date: February 13, 2026**
