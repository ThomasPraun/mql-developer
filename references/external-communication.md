# External & Internal Communication

Reference for WebRequest, REST API integration, JSON handling, Node.js patterns, and inter-program communication in MQL4/MQL5.

## Table of Contents

- [WebRequest - REST API Communication](#webrequest---rest-api-communication)
- [JSON Handling](#json-handling)
- [Node.js Integration Patterns](#nodejs-integration-patterns)
- [Network Error Handling](#network-error-handling)
- [Internal Communication (Between MQL Programs)](#internal-communication-between-mql-programs)
- [MQL5 Sockets (Advanced)](#mql5-sockets-advanced)
- [Communication Method Comparison](#communication-method-comparison)

---

## WebRequest - REST API Communication

### Setup Requirements

- **Whitelist URLs**: Tools > Options > Expert Advisors > "Allow WebRequest for listed URL"
- Only available in **EAs and Scripts** (NOT indicators)
- **Not available** during backtesting in Strategy Tester
- **Synchronous/blocking**: freezes EA execution until response received or timeout expires
- Each call blocks the entire EA thread; keep timeouts reasonable (5000-10000 ms)

### MQL4 WebRequest Signatures

MQL4 provides two overloaded variants:

```mql4
// Variant 1: cookie/referer (simpler, limited)
int WebRequest(
    const string method,           // "GET", "POST", "PUT", "DELETE"
    const string url,              // Full URL
    const string cookie,           // Cookie string (can be "")
    const string referer,          // Referer header (can be "")
    int timeout,                   // Timeout in milliseconds
    const char &data[],            // Request body (char array)
    int data_size,                 // Size of data array
    char &result[],                // Response body (output)
    string &result_headers         // Response headers (output)
);

// Variant 2: custom headers (preferred for REST APIs)
int WebRequest(
    const string method,           // "GET", "POST", "PUT", "DELETE"
    const string url,              // Full URL
    const string headers,          // Custom headers, each ending with \r\n
    int timeout,                   // Timeout in milliseconds
    const char &data[],            // Request body (char array)
    char &result[],                // Response body (output)
    string &result_headers         // Response headers (output)
);
```

### MQL4 GET Request Example

```mql4
string HttpGet(string url, int timeout = 5000)
{
    char   data[];
    char   result[];
    string resultHeaders;
    string headers = "Content-Type: application/json\r\n";

    ResetLastError();
    int statusCode = WebRequest("GET", url, headers, timeout, data, result, resultHeaders);

    if(statusCode == -1)
    {
        int error = GetLastError();
        if(error == 4014)
            Print("ERROR: URL not allowed. Add to Tools > Options > Expert Advisors: ", url);
        else if(error == 4060)
            Print("ERROR: WebRequest not allowed in this context (indicator or tester)");
        else
            Print("ERROR: WebRequest failed. Error code: ", error);
        return "";
    }

    if(statusCode != 200)
    {
        Print("HTTP Error: ", statusCode, " Response: ", CharArrayToString(result));
        return "";
    }

    return CharArrayToString(result);
}
```

### MQL4 POST Request Example

```mql4
string HttpPost(string url, string jsonBody, int timeout = 5000)
{
    char   data[];
    char   result[];
    string resultHeaders;
    string headers = "Content-Type: application/json\r\n";

    // CRITICAL: Use StringLen() to avoid including the null terminator
    StringToCharArray(jsonBody, data, 0, StringLen(jsonBody));

    ResetLastError();
    int statusCode = WebRequest("POST", url, headers, timeout, data, result, resultHeaders);

    if(statusCode == -1)
    {
        int error = GetLastError();
        Print("ERROR: WebRequest POST failed. Error: ", error);
        return "";
    }

    if(statusCode != 200 && statusCode != 201)
    {
        Print("HTTP Error: ", statusCode, " Response: ", CharArrayToString(result));
        return "";
    }

    return CharArrayToString(result);
}
```

### MQL5 WebRequest

MQL5 uses the same `WebRequest()` function with identical signatures. The key difference is encoding support:

```mql5
// MQL5 POST with explicit UTF-8 encoding
string HttpPostMQL5(string url, string jsonBody, int timeout = 5000)
{
    char   data[];
    char   result[];
    string resultHeaders;
    string headers = "Content-Type: application/json\r\n";

    // Use CP_UTF8 for proper Unicode handling
    // CRITICAL: Use StringLen() to avoid null terminator in body
    StringToCharArray(jsonBody, data, 0, StringLen(jsonBody), CP_UTF8);

    ResetLastError();
    int statusCode = WebRequest("POST", url, headers, timeout, data, result, resultHeaders);

    if(statusCode == -1)
    {
        int error = GetLastError();
        PrintFormat("WebRequest failed: error %d", error);
        return "";
    }

    // Decode response with UTF-8
    string response = CharArrayToString(result, 0, WHOLE_ARRAY, CP_UTF8);
    return response;
}
```

### CHttpClient Class (MQL5)

Complete reusable HTTP client class for REST API communication:

```mql5
//+------------------------------------------------------------------+
//| CHttpClient - Reusable HTTP client for REST APIs                  |
//+------------------------------------------------------------------+
class CHttpClient
{
private:
    string m_baseUrl;
    int    m_timeout;
    string m_authHeader;    // Optional auth header

    string DoRequest(string method, string endpoint, string body = "")
    {
        char   data[];
        char   result[];
        string resultHeaders;

        string url = m_baseUrl + endpoint;
        string headers = "Content-Type: application/json\r\n";
        if(m_authHeader != "")
            headers += m_authHeader + "\r\n";

        if(body != "")
        {
            // CRITICAL: Use StringLen() to avoid null terminator byte in request body
            StringToCharArray(body, data, 0, StringLen(body), CP_UTF8);
        }

        ResetLastError();
        int statusCode = WebRequest(method, url, headers, m_timeout, data, result, resultHeaders);

        if(statusCode == -1)
        {
            int error = GetLastError();
            if(error == 4014)
                PrintFormat("ERROR: URL not whitelisted: %s", url);
            else if(error == 4060)
                PrintFormat("ERROR: WebRequest not allowed in this context");
            else
                PrintFormat("ERROR: WebRequest failed, error: %d", error);
            return "";
        }

        string response = CharArrayToString(result, 0, WHOLE_ARRAY, CP_UTF8);

        if(statusCode < 200 || statusCode >= 300)
        {
            PrintFormat("HTTP %d: %s %s -> %s", statusCode, method, endpoint, response);
            return "";
        }

        return response;
    }

public:
    void Init(string baseUrl, int timeoutMs = 5000)
    {
        m_baseUrl = baseUrl;
        m_timeout = timeoutMs;
        m_authHeader = "";
    }

    void SetAuth(string token)
    {
        m_authHeader = "Authorization: Bearer " + token;
    }

    string Get(string endpoint)
    {
        return DoRequest("GET", endpoint);
    }

    string Post(string endpoint, string jsonBody)
    {
        return DoRequest("POST", endpoint, jsonBody);
    }

    string Put(string endpoint, string jsonBody)
    {
        return DoRequest("PUT", endpoint, jsonBody);
    }

    string Delete(string endpoint)
    {
        return DoRequest("DELETE", endpoint);
    }
};
```

**Usage:**

```mql5
CHttpClient httpClient;

int OnInit()
{
    httpClient.Init("https://api.example.com", 5000);
    httpClient.SetAuth(InpApiToken);
    return INIT_SUCCEEDED;
}

void OnTick()
{
    string response = httpClient.Get("/api/signals?symbol=" + Symbol());
    if(response != "")
    {
        // Process response
    }
}
```

---

## JSON Handling

MQL has no native JSON parser. For simple payloads, manual string building and parsing works well. For complex nested structures, consider the `CJAVal` library from MQL5 CodeBase.

### Building JSON Manually

```mql5
string BuildTradeJSON(ulong ticket, string symbol, string action,
                      double lots, double price, double sl, double tp)
{
    string json = "{";
    json += "\"ticket\":" + IntegerToString(ticket) + ",";
    json += "\"symbol\":\"" + symbol + "\",";
    json += "\"action\":\"" + action + "\",";
    json += "\"lots\":" + DoubleToString(lots, 2) + ",";
    json += "\"price\":" + DoubleToString(price, (int)SymbolInfoInteger(symbol, SYMBOL_DIGITS)) + ",";
    json += "\"sl\":" + DoubleToString(sl, (int)SymbolInfoInteger(symbol, SYMBOL_DIGITS)) + ",";
    json += "\"tp\":" + DoubleToString(tp, (int)SymbolInfoInteger(symbol, SYMBOL_DIGITS)) + ",";
    json += "\"account\":" + IntegerToString(AccountInfoInteger(ACCOUNT_LOGIN)) + ",";
    json += "\"broker\":\"" + AccountInfoString(ACCOUNT_COMPANY) + "\",";
    json += "\"balance\":" + DoubleToString(AccountInfoDouble(ACCOUNT_BALANCE), 2) + ",";
    json += "\"equity\":" + DoubleToString(AccountInfoDouble(ACCOUNT_EQUITY), 2) + ",";
    json += "\"timestamp\":\"" + TimeToString(TimeCurrent(), TIME_DATE | TIME_SECONDS) + "\"";
    json += "}";
    return json;
}
```

**MQL4 variant** (uses `AccountBalance()`, `AccountCompany()`, etc. instead of `AccountInfoXxx()`):

```mql4
string BuildTradeJSON_MQL4(int ticket, string symbol, string action,
                           double lots, double price, double sl, double tp)
{
    string json = "{";
    json += "\"ticket\":" + IntegerToString(ticket) + ",";
    json += "\"symbol\":\"" + symbol + "\",";
    json += "\"action\":\"" + action + "\",";
    json += "\"lots\":" + DoubleToString(lots, 2) + ",";
    json += "\"price\":" + DoubleToString(price, Digits) + ",";
    json += "\"sl\":" + DoubleToString(sl, Digits) + ",";
    json += "\"tp\":" + DoubleToString(tp, Digits) + ",";
    json += "\"account\":" + IntegerToString(AccountNumber()) + ",";
    json += "\"broker\":\"" + AccountCompany() + "\",";
    json += "\"balance\":" + DoubleToString(AccountBalance(), 2) + ",";
    json += "\"equity\":" + DoubleToString(AccountEquity(), 2) + ",";
    json += "\"timestamp\":\"" + TimeToString(TimeCurrent(), TIME_DATE | TIME_SECONDS) + "\"";
    json += "}";
    return json;
}
```

### Parsing JSON Manually

Helper functions for extracting values from flat JSON strings:

```mql5
//+------------------------------------------------------------------+
//| Extract string value for a given key from JSON                    |
//+------------------------------------------------------------------+
string JsonGetString(const string json, const string key)
{
    string searchKey = "\"" + key + "\"";
    int keyPos = StringFind(json, searchKey);
    if(keyPos == -1) return "";

    // Find the colon after the key
    int colonPos = StringFind(json, ":", keyPos + StringLen(searchKey));
    if(colonPos == -1) return "";

    // Find opening quote of value
    int startQuote = StringFind(json, "\"", colonPos + 1);
    if(startQuote == -1) return "";

    // Find closing quote of value
    int endQuote = StringFind(json, "\"", startQuote + 1);
    if(endQuote == -1) return "";

    return StringSubstr(json, startQuote + 1, endQuote - startQuote - 1);
}

//+------------------------------------------------------------------+
//| Extract double value for a given key from JSON                    |
//+------------------------------------------------------------------+
double JsonGetDouble(const string json, const string key)
{
    string searchKey = "\"" + key + "\"";
    int keyPos = StringFind(json, searchKey);
    if(keyPos == -1) return 0.0;

    int colonPos = StringFind(json, ":", keyPos + StringLen(searchKey));
    if(colonPos == -1) return 0.0;

    // Skip whitespace after colon
    int valueStart = colonPos + 1;
    while(valueStart < StringLen(json) &&
          (StringGetCharacter(json, valueStart) == ' ' ||
           StringGetCharacter(json, valueStart) == '\t'))
        valueStart++;

    // Find end of number (comma, closing brace, closing bracket, or end of string)
    int valueEnd = valueStart;
    while(valueEnd < StringLen(json))
    {
        ushort ch = StringGetCharacter(json, valueEnd);
        if(ch == ',' || ch == '}' || ch == ']' || ch == ' ' || ch == '\n' || ch == '\r')
            break;
        valueEnd++;
    }

    string valueStr = StringSubstr(json, valueStart, valueEnd - valueStart);
    return StringToDouble(valueStr);
}

//+------------------------------------------------------------------+
//| Extract boolean value for a given key from JSON                   |
//+------------------------------------------------------------------+
bool JsonGetBool(const string json, const string key)
{
    string searchKey = "\"" + key + "\"";
    int keyPos = StringFind(json, searchKey);
    if(keyPos == -1) return false;

    int colonPos = StringFind(json, ":", keyPos + StringLen(searchKey));
    if(colonPos == -1) return false;

    // Check if "true" appears after colon (before next comma/brace)
    int truePos = StringFind(json, "true", colonPos);
    int commaPos = StringFind(json, ",", colonPos);
    int bracePos = StringFind(json, "}", colonPos);

    if(truePos == -1) return false;
    if(commaPos != -1 && truePos > commaPos) return false;
    if(bracePos != -1 && truePos > bracePos) return false;

    return true;
}

//+------------------------------------------------------------------+
//| Extract integer value for a given key from JSON                   |
//+------------------------------------------------------------------+
int JsonGetInt(const string json, const string key)
{
    return (int)JsonGetDouble(json, key);
}

//+------------------------------------------------------------------+
//| Extract long value for a given key from JSON                      |
//+------------------------------------------------------------------+
long JsonGetLong(const string json, const string key)
{
    string searchKey = "\"" + key + "\"";
    int keyPos = StringFind(json, searchKey);
    if(keyPos == -1) return 0;

    int colonPos = StringFind(json, ":", keyPos + StringLen(searchKey));
    if(colonPos == -1) return 0;

    int valueStart = colonPos + 1;
    while(valueStart < StringLen(json) &&
          (StringGetCharacter(json, valueStart) == ' ' ||
           StringGetCharacter(json, valueStart) == '\t'))
        valueStart++;

    int valueEnd = valueStart;
    while(valueEnd < StringLen(json))
    {
        ushort ch = StringGetCharacter(json, valueEnd);
        if(ch == ',' || ch == '}' || ch == ']' || ch == ' ')
            break;
        valueEnd++;
    }

    string valueStr = StringSubstr(json, valueStart, valueEnd - valueStart);
    return StringToInteger(valueStr);
}
```

**Note:** These helpers work for flat (non-nested) JSON. For production use with nested objects or arrays, consider the **CJAVal library** from MQL5 CodeBase, which provides proper recursive JSON parsing with object/array traversal.

---

## Node.js Integration Patterns

### EA -> Node.js: Send Trade Data

Report executed trades to a Node.js backend:

```mql5
CHttpClient httpClient;

void NotifyServer(string action, string symbol, double lots, double price, ulong ticket)
{
    string json = BuildTradeJSON(ticket, symbol, action, lots, price, 0, 0);
    string response = httpClient.Post("/api/trades", json);
    if(response == "")
        PrintFormat("WARNING: Failed to notify server about %s %s", action, symbol);
}

// Call after trade execution:
void OnTradeTransaction(const MqlTradeTransaction &trans,
                        const MqlTradeRequest &request,
                        const MqlTradeResult &result)
{
    if(trans.type == TRADE_TRANSACTION_DEAL_ADD)
    {
        // New deal executed
        NotifyServer("BUY", trans.symbol, trans.volume, trans.price, trans.deal);
    }
}
```

### Node.js -> EA: Receive Commands (Polling)

Timer-based polling pattern to receive trading signals from a Node.js server:

```mql5
CHttpClient httpClient;

int OnInit()
{
    httpClient.Init("https://api.example.com", 5000);
    httpClient.SetAuth(InpApiToken);
    EventSetTimer(5);  // Poll every 5 seconds
    return INIT_SUCCEEDED;
}

void OnTimer()
{
    string endpoint = "/api/signals?account=" +
                      IntegerToString(AccountInfoInteger(ACCOUNT_LOGIN)) +
                      "&symbol=" + Symbol();
    string response = httpClient.Get(endpoint);
    if(response == "") return;

    string action = JsonGetString(response, "action");
    if(action == "") return;

    double lots   = JsonGetDouble(response, "lots");
    double sl     = JsonGetDouble(response, "sl");
    double tp     = JsonGetDouble(response, "tp");
    string symbol = JsonGetString(response, "symbol");
    if(symbol == "") symbol = Symbol();

    if(action == "BUY")
        ExecuteBuy(symbol, lots, sl, tp);
    else if(action == "SELL")
        ExecuteSell(symbol, lots, sl, tp);
    else if(action == "CLOSE")
        ClosePosition(symbol);
}

void OnDeinit(const int reason)
{
    EventKillTimer();
}
```

### Send Account Status Updates

Periodically report account state to the server:

```mql5
void SendAccountStatus()
{
    string json = "{";
    json += "\"account\":" + IntegerToString(AccountInfoInteger(ACCOUNT_LOGIN)) + ",";
    json += "\"balance\":" + DoubleToString(AccountInfoDouble(ACCOUNT_BALANCE), 2) + ",";
    json += "\"equity\":" + DoubleToString(AccountInfoDouble(ACCOUNT_EQUITY), 2) + ",";
    json += "\"margin\":" + DoubleToString(AccountInfoDouble(ACCOUNT_MARGIN), 2) + ",";
    json += "\"freeMargin\":" + DoubleToString(AccountInfoDouble(ACCOUNT_MARGIN_FREE), 2) + ",";
    json += "\"openPositions\":" + IntegerToString(PositionsTotal()) + ",";
    json += "\"server\":\"" + AccountInfoString(ACCOUNT_SERVER) + "\",";
    json += "\"timestamp\":\"" + TimeToString(TimeCurrent(), TIME_DATE | TIME_SECONDS) + "\"";
    json += "}";

    httpClient.Post("/api/status", json);
}
```

### Common API Endpoints Pattern

| Method | Endpoint | Purpose |
|--------|----------|---------|
| POST | `/api/trades` | Report executed trades |
| GET | `/api/signals` | Poll for trading signals |
| POST | `/api/status` | Send account status (balance, equity, positions) |
| GET | `/api/config` | Fetch EA configuration parameters |
| POST | `/api/errors` | Report errors and alerts |
| POST | `/api/logs` | Send EA log entries |

### Authentication

Include an authentication token in every request via headers:

```mql5
input string InpApiToken = "";  // API Bearer Token

// In CHttpClient or raw WebRequest:
string headers = "Content-Type: application/json\r\n"
               + "Authorization: Bearer " + InpApiToken + "\r\n";
```

For the `CHttpClient` class, use `httpClient.SetAuth(InpApiToken)` after `Init()`.

---

## Network Error Handling

### Retry Pattern with Progressive Backoff

```mql5
string RequestWithRetry(string method, string url, string headers,
                        string body, int maxRetries = 3)
{
    char   data[];
    char   result[];
    string resultHeaders;

    if(body != "")
        StringToCharArray(body, data, 0, StringLen(body), CP_UTF8);

    for(int attempt = 0; attempt < maxRetries; attempt++)
    {
        if(attempt > 0)
        {
            int delayMs = 1000 * (attempt + 1);  // 2s, 3s progressive backoff
            PrintFormat("Retry %d/%d after %d ms delay...", attempt + 1, maxRetries, delayMs);
            Sleep(delayMs);
        }

        ResetLastError();
        int statusCode = WebRequest(method, url, headers, 5000, data, result, resultHeaders);

        if(statusCode == -1)
        {
            int error = GetLastError();
            PrintFormat("WebRequest attempt %d failed: error %d", attempt + 1, error);
            // Don't retry non-recoverable errors
            if(error == 4014 || error == 4060) return "";
            continue;
        }

        if(statusCode >= 500)
        {
            PrintFormat("Server error %d, attempt %d", statusCode, attempt + 1);
            continue;  // Retry on server errors
        }

        // Return response for any non-server-error status
        return CharArrayToString(result, 0, WHOLE_ARRAY, CP_UTF8);
    }

    PrintFormat("All %d attempts failed for %s %s", maxRetries, method, url);
    return "";
}
```

### Common Error Codes

| Status Code | GetLastError() | Meaning |
|-------------|---------------|---------|
| -1 | 4014 | URL not in allowed list (whitelist in terminal settings) |
| -1 | 4060 | Function not allowed (called from indicator or Strategy Tester) |
| -1 | 5203 | No connection to server / network error |
| -1 | 5200-5299 | Various network/internet errors |
| HTTP 0 | - | Timeout / no response from server |
| HTTP 400 | - | Bad request (check JSON format) |
| HTTP 401 | - | Authentication failed (check token) |
| HTTP 403 | - | Forbidden (check permissions) |
| HTTP 404 | - | Endpoint not found (check URL) |
| HTTP 429 | - | Rate limited (add delays between requests) |
| HTTP 500 | - | Internal server error (server-side issue) |
| HTTP 502/503 | - | Server unavailable (retry later) |

### Error Reporting to Server

```mql5
void ReportErrorToServer(string source, int errorCode, string details)
{
    string json = "{";
    json += "\"source\":\"" + source + "\",";
    json += "\"errorCode\":" + IntegerToString(errorCode) + ",";
    json += "\"details\":\"" + details + "\",";
    json += "\"account\":" + IntegerToString(AccountInfoInteger(ACCOUNT_LOGIN)) + ",";
    json += "\"timestamp\":\"" + TimeToString(TimeCurrent(), TIME_DATE | TIME_SECONDS) + "\"";
    json += "}";

    // Don't retry error reporting itself to avoid infinite loops
    httpClient.Post("/api/errors", json);
}
```

---

## Internal Communication (Between MQL Programs)

### Global Variables of the Terminal

Global variables are shared across all MQL programs running in a single terminal instance. They store `double` values and persist across program restarts (saved to disk by the terminal).

```mql5
// Write a value
GlobalVariableSet("EA_Signal_EURUSD", 1.0);

// Read a value
double signal = GlobalVariableGet("EA_Signal_EURUSD");

// Check existence
bool exists = GlobalVariableCheck("EA_Signal_EURUSD");

// Delete
GlobalVariableDel("EA_Signal_EURUSD");

// Create temporary (auto-deleted when terminal closes, not saved to disk)
GlobalVariableTemp("EA_Temp_Signal");

// Set only if doesn't exist (atomic check-and-set, useful for locking)
bool wasCreated = GlobalVariableSetOnCondition("EA_Lock", 1.0, 0.0);

// Get value and set time simultaneously
datetime lastAccess;
double value = GlobalVariableGet("EA_Signal", lastAccess);

// List and iterate all global variables
int total = GlobalVariablesTotal();
for(int i = 0; i < total; i++)
{
    string name = GlobalVariableName(i);
    double val = GlobalVariableGet(name);
    PrintFormat("GVar: %s = %f", name, val);
}
```

**Constraints:**
- Name maximum: **63 characters**
- Value type: always **double** (encode other types as double or use naming conventions)
- Shared across **all programs in the same terminal**
- Persisted to disk on terminal shutdown (except `GlobalVariableTemp`)

**Use cases:**
- Signal passing between EAs and indicators on different charts
- Simple locking mechanism with `GlobalVariableSetOnCondition()`
- State persistence across EA restarts
- Coordination between multiple EAs (e.g., portfolio-level risk)

### Custom Events (MQL5 Only)

Custom events allow one MQL5 program to send events to a chart's `OnChartEvent()` handler:

```mql5
// --- Define custom event IDs ---
#define EVENT_SIGNAL_BUY    (CHARTEVENT_CUSTOM + 1)
#define EVENT_SIGNAL_SELL   (CHARTEVENT_CUSTOM + 2)
#define EVENT_UPDATE_PANEL  (CHARTEVENT_CUSTOM + 3)
#define EVENT_CLOSE_ALL     (CHARTEVENT_CUSTOM + 4)

// --- Sender (indicator, script, or another EA on the same chart): ---
// Send to current chart
EventChartCustom(ChartID(), EVENT_SIGNAL_BUY, 0, 1.23456, "EURUSD");

// Send to a specific chart by chart ID
long targetChart = ChartFirst();
EventChartCustom(targetChart, EVENT_SIGNAL_BUY, 12345, 1.5, "EURUSD");

// --- Receiver (EA with OnChartEvent handler): ---
void OnChartEvent(const int id, const long &lparam,
                  const double &dparam, const string &sparam)
{
    if(id == EVENT_SIGNAL_BUY)
    {
        long   ticket = lparam;    // Custom long parameter
        double price  = dparam;    // Custom double parameter
        string symbol = sparam;    // Custom string parameter
        PrintFormat("BUY signal received: %s at %f", symbol, price);
    }
    else if(id == EVENT_SIGNAL_SELL)
    {
        PrintFormat("SELL signal received: %s at %f", sparam, dparam);
    }
    else if(id == EVENT_CLOSE_ALL)
    {
        // Close all positions
    }
}
```

**Parameters per event:**
- `lparam` (long): one integer/long value
- `dparam` (double): one double value
- `sparam` (string): one string value (can encode JSON for more data)

**Constraints:**
- Can only send to charts in the same terminal
- The receiving chart must have an EA or indicator with `OnChartEvent()`
- Custom event IDs range: `CHARTEVENT_CUSTOM` to `CHARTEVENT_CUSTOM + 65535`

### File-Based Communication

Programs can communicate by reading and writing files in the terminal data folder (`MQL4/Files` or `MQL5/Files`):

```mql5
// --- Writer (EA or Script) ---
void WriteSignalFile(string signal, string symbol, double price)
{
    string filename = "signals_" + symbol + ".csv";
    int handle = FileOpen(filename, FILE_WRITE | FILE_CSV | FILE_ANSI, ',');
    if(handle == INVALID_HANDLE)
    {
        PrintFormat("Failed to open file: %s, error: %d", filename, GetLastError());
        return;
    }
    FileWrite(handle, signal, symbol, DoubleToString(price, 5),
              TimeToString(TimeCurrent()));
    FileClose(handle);
}

// --- Reader (another EA or Indicator) ---
string ReadSignalFile(string symbol)
{
    string filename = "signals_" + symbol + ".csv";
    if(!FileIsExist(filename)) return "";

    int handle = FileOpen(filename, FILE_READ | FILE_CSV | FILE_ANSI, ',');
    if(handle == INVALID_HANDLE) return "";

    string signal = FileReadString(handle);
    FileClose(handle);

    // Delete after reading (one-time signal)
    FileDelete(filename);
    return signal;
}
```

**Cross-terminal file sharing** using `FILE_COMMON` flag:

```mql5
// Write to common folder (shared across all terminal instances)
int handle = FileOpen("shared_signal.txt", FILE_WRITE | FILE_TXT | FILE_COMMON);
FileWriteString(handle, "BUY EURUSD 1.12345");
FileClose(handle);

// Read from common folder in another terminal
int handle2 = FileOpen("shared_signal.txt", FILE_READ | FILE_TXT | FILE_COMMON);
string data = FileReadString(handle2);
FileClose(handle2);
```

**File locking pattern** (prevent concurrent read/write corruption):

```mql5
bool AcquireFileLock(string lockName, int timeoutMs = 5000)
{
    string lockFile = lockName + ".lock";
    datetime start = TimeLocal();
    while(FileIsExist(lockFile))
    {
        if((TimeLocal() - start) * 1000 > timeoutMs)
            return false;  // Timeout
        Sleep(50);
    }
    // Create lock file
    int h = FileOpen(lockFile, FILE_WRITE | FILE_TXT);
    if(h == INVALID_HANDLE) return false;
    FileClose(h);
    return true;
}

void ReleaseFileLock(string lockName)
{
    FileDelete(lockName + ".lock");
}
```

### Named Pipes (MQL4, Windows Only)

Named pipes provide fast IPC for Windows-based communication (e.g., between MetaTrader and a C#/Python application):

```mql4
#import "kernel32.dll"
int CreateFileW(string name, uint access, uint share, int security,
                uint creation, uint flags, int template);
int WriteFile(int handle, const uchar &buffer[], int bytes,
              int &written[], int overlapped);
int ReadFile(int handle, uchar &buffer[], int bytes,
             int &read[], int overlapped);
int CloseHandle(int handle);
int FlushFileBuffers(int handle);
#import

#define GENERIC_READ       0x80000000
#define GENERIC_WRITE      0x40000000
#define OPEN_EXISTING      3
#define INVALID_HANDLE_VALUE -1

// Connect to a named pipe server
int ConnectToPipe(string pipeName)
{
    string fullName = "\\\\.\\pipe\\" + pipeName;
    int pipe = CreateFileW(fullName,
                           GENERIC_READ | GENERIC_WRITE,
                           0, 0, OPEN_EXISTING, 0, 0);
    if(pipe == INVALID_HANDLE_VALUE)
    {
        Print("Failed to connect to pipe: ", pipeName);
        return INVALID_HANDLE_VALUE;
    }
    return pipe;
}

// Send message through pipe
bool SendPipeMessage(int pipe, string message)
{
    uchar data[];
    StringToCharArray(message, data);
    int written[];
    ArrayResize(written, 1);
    return WriteFile(pipe, data, ArraySize(data), written, 0) != 0;
}

// Read message from pipe
string ReadPipeMessage(int pipe, int bufferSize = 4096)
{
    uchar buffer[];
    ArrayResize(buffer, bufferSize);
    int bytesRead[];
    ArrayResize(bytesRead, 1);
    if(ReadFile(pipe, buffer, bufferSize, bytesRead, 0))
        return CharArrayToString(buffer, 0, bytesRead[0]);
    return "";
}

// Clean up
void ClosePipe(int pipe)
{
    FlushFileBuffers(pipe);
    CloseHandle(pipe);
}
```

---

## MQL5 Sockets (Advanced)

MQL5 provides built-in TCP socket support for real-time bidirectional communication. This is more efficient than WebRequest polling for scenarios requiring low-latency or persistent connections.

### Plain TCP Socket

```mql5
int g_socket = INVALID_HANDLE;

bool SocketConnectToServer(string host, int port, int timeoutMs = 5000)
{
    g_socket = SocketCreate();
    if(g_socket == INVALID_HANDLE)
    {
        PrintFormat("SocketCreate failed: %d", GetLastError());
        return false;
    }

    if(!SocketConnect(g_socket, host, port, timeoutMs))
    {
        PrintFormat("SocketConnect failed: %d", GetLastError());
        SocketClose(g_socket);
        g_socket = INVALID_HANDLE;
        return false;
    }

    PrintFormat("Connected to %s:%d", host, port);
    return true;
}

bool SocketSendMessage(string message)
{
    if(g_socket == INVALID_HANDLE) return false;

    uchar data[];
    int len = StringToCharArray(message, data, 0, StringLen(message), CP_UTF8);

    int sent = SocketSend(g_socket, data, len);
    if(sent == -1)
    {
        PrintFormat("SocketSend failed: %d", GetLastError());
        return false;
    }
    return true;
}

string SocketReceiveMessage(int timeoutMs = 1000)
{
    if(g_socket == INVALID_HANDLE) return "";

    uchar response[];
    int received = SocketRead(g_socket, response, 4096, timeoutMs);
    if(received <= 0) return "";

    return CharArrayToString(response, 0, received, CP_UTF8);
}

void SocketDisconnect()
{
    if(g_socket != INVALID_HANDLE)
    {
        SocketClose(g_socket);
        g_socket = INVALID_HANDLE;
    }
}
```

### TLS/SSL Encrypted Socket

For secure communication (HTTPS servers, encrypted APIs):

```mql5
bool SocketConnectTLS(string host, int port, int timeoutMs = 5000)
{
    g_socket = SocketCreate();
    if(g_socket == INVALID_HANDLE) return false;

    if(!SocketConnect(g_socket, host, port, timeoutMs))
    {
        SocketClose(g_socket);
        g_socket = INVALID_HANDLE;
        return false;
    }

    // Perform TLS handshake
    if(!SocketTlsHandshake(g_socket, host))
    {
        PrintFormat("TLS handshake failed: %d", GetLastError());
        SocketClose(g_socket);
        g_socket = INVALID_HANDLE;
        return false;
    }

    return true;
}

// For TLS, use SocketTlsSend / SocketTlsRead instead:
bool SocketSendTLS(string message)
{
    if(g_socket == INVALID_HANDLE) return false;

    uchar data[];
    int len = StringToCharArray(message, data, 0, StringLen(message), CP_UTF8);
    int sent = SocketTlsSend(g_socket, data, len);
    return (sent > 0);
}

string SocketReceiveTLS(int timeoutMs = 1000)
{
    if(g_socket == INVALID_HANDLE) return "";

    uchar response[];
    int received = SocketTlsRead(g_socket, response, 4096, timeoutMs);
    if(received <= 0) return "";

    return CharArrayToString(response, 0, received, CP_UTF8);
}
```

### Socket Constraints

- Only available in **EAs and Scripts** (NOT indicators)
- Maximum **128 sockets per program**
- Server URLs/IPs must be **whitelisted** in terminal settings (same as WebRequest)
- Non-blocking reads with timeout; blocking sends
- MQL5 does not support acting as a socket server (client only)
- For WebSocket protocol, you must implement the handshake and framing manually or use a library

---

## Communication Method Comparison

| Method | Direction | Latency | Complexity | MQL4 | MQL5 | Notes |
|--------|-----------|---------|------------|------|------|-------|
| WebRequest | EA -> Server | High (blocking) | Low | Yes | Yes | Simple REST, synchronous |
| Sockets | Bidirectional | Low | Medium | No | Yes | Persistent connection |
| Global Variables | Internal | Very low | Very low | Yes | Yes | Double values only |
| Custom Events | Internal | Very low | Low | No | Yes | Same terminal only |
| Files | Both | Medium | Low | Yes | Yes | FILE_COMMON for cross-terminal |
| Named Pipes | Bidirectional | Low | High | Yes (DLL) | Yes (DLL) | Windows only, requires DLL import |
