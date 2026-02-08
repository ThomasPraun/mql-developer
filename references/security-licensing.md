# Security & Licensing

## Table of Contents

- [Account-Based Licensing](#account-based-licensing)
- [Server-Side License Validation](#server-side-license-validation)
- [Expiration Date Checks](#expiration-date-checks)
- [Anti-Decompilation Techniques](#anti-decompilation-techniques)
- [Distribution Best Practices](#distribution-best-practices)
- [Node.js License Server Example](#nodejs-license-server-example)
- [MQL4 Specific Considerations](#mql4-specific-considerations)
- [Complete License System Architecture](#complete-license-system-architecture)

---

## Account-Based Licensing

### Simple Multi-Account Check

```mql5
bool CheckAccountLicense() {
    long currentAccount = AccountInfoInteger(ACCOUNT_LOGIN);
    long authorizedAccounts[] = {12345678, 87654321, 11223344};
    for(int i = 0; i < ArraySize(authorizedAccounts); i++) {
        if(currentAccount == authorizedAccounts[i]) return true;
    }
    Alert("Account ", currentAccount, " is not authorized.");
    return false;
}
```

### Account + Broker Hash Check (More Secure)

Don't store plain account numbers in code. Hash account + broker + salt:

```mql5
ulong HashString(string s) {
    ulong hash = 5381;
    for(int i = 0; i < StringLen(s); i++)
        hash = ((hash << 5) + hash) + StringGetCharacter(s, i);
    return hash;
}

bool ValidateLicense() {
    long account = AccountInfoInteger(ACCOUNT_LOGIN);
    string server = AccountInfoString(ACCOUNT_SERVER);
    string raw = IntegerToString(account) + "|" + server + "|SECRET_SALT";
    ulong hash = HashString(raw);
    ulong validHashes[] = {0xA1B2C3D4E5F6, 0x1234567890AB};
    for(int i = 0; i < ArraySize(validHashes); i++)
        if(hash == validHashes[i]) return true;
    return false;
}
```

---

## Server-Side License Validation

### Implementation Pattern

```mql5
input string InpLicenseKey = "";

bool ValidateLicenseOnServer() {
    long account = AccountInfoInteger(ACCOUNT_LOGIN);
    string broker = AccountInfoString(ACCOUNT_SERVER);
    string eaName = MQLInfoString(MQL_PROGRAM_NAME);

    string json = "{";
    json += "\"license_key\":\"" + InpLicenseKey + "\",";
    json += "\"account\":" + IntegerToString(account) + ",";
    json += "\"broker\":\"" + broker + "\",";
    json += "\"ea\":\"" + eaName + "\"";
    json += "}";

    // Send to server, check response
    // Handle network failures (fail-open vs fail-closed decision)
}
```

### Fail-Open vs Fail-Closed

- **Fail-open:** Allow trading if server unreachable (better UX, weaker security)
- **Fail-closed:** Block trading if server unreachable (stronger security, risk of false blocks)
- **Recommended:** Fail-open with cached expiry and offline grace period

### Caching License Locally

Use `GlobalVariableSet` to cache server expiry:

```mql5
GlobalVariableSet("EA_LICENSE_EXPIRY", (double)StringToTime(expiry));
```

On next check, verify cached expiry first, re-validate with server periodically.

---

## Expiration Date Checks

### Hardcoded Expiration

```mql5
datetime expiryDate = D'2025.12.31 23:59:59';
if(TimeCurrent() > expiryDate) {
    Alert("EA has expired. Please renew.");
    return INIT_FAILED;
}
```

### Server-Cached Expiration

Check `GlobalVariable` cache first, re-validate on timer.

### Demo Mode with Limited Features

After trial period, restrict to demo accounts or limited lot sizes.

### Integration in OnInit

```mql5
int OnInit() {
    if(!CheckAccountLicense()) return INIT_FAILED;
    if(!CheckExpiration()) return INIT_FAILED;
    EventSetTimer(3600); // Re-check every hour
    return INIT_SUCCEEDED;
}

void OnTimer() {
    if(!CheckExpiration()) ExpertRemove();
}
```

---

## Anti-Decompilation Techniques

### Protection Layers Table

| Layer | Technique | Protection Level | Notes |
|-------|-----------|-----------------|-------|
| 1 | Compile to .ex4/.ex5 | Basic | Strips variable/function names |
| 2 | Code obfuscation | Low | Complex expressions, dummy code |
| 3 | String encryption | Medium | Hide URLs, keys, constants |
| 4 | MQL5 Cloud Protector | High | Asymmetric encryption, native code |
| 5 | DLL offloading | High | Core logic in C++ DLL |
| 6 | Server-side logic | Highest | Core signals never leave server |

### String Obfuscation Example

```mql5
string DecryptString(int key) {
    uchar encrypted[] = {104,116,116,112,115,58,47,47};
    string result = "";
    for(int i = 0; i < ArraySize(encrypted); i++)
        result += CharToString((uchar)(encrypted[i] ^ (key % 256)));
    return result;
}
```

### MQL5 Cloud Protector

- Available in MetaEditor: Tools > MQL5 Cloud Protector
- Sends compiled .ex5 to MetaQuotes cloud
- Applies asymmetric encryption + unique key signing
- Source code never leaves your machine
- Same protection as MQL5 Market store
- Files NOT bound to specific computer (unlike Market)
- Free to use

---

## Distribution Best Practices

### Do's

- Distribute only .ex4/.ex5 compiled files
- Use MQL5 Market for built-in DRM (hardware + account binding)
- Combine account binding + server validation + Cloud Protector
- Include version checking in licensing server to force updates
- Use unique magic numbers or comments to identify your EA

### Don'ts

- Never distribute .mq4/.mq5 source files
- Never hardcode API keys or server passwords in source
- Don't rely on a single protection layer
- Don't use simple string comparisons for license keys

---

## Node.js License Server Example

### Endpoint: POST /api/validate

```javascript
app.post('/api/validate', (req, res) => {
    const { license_key, account, broker, ea, version } = req.body;

    // Look up license in database
    const license = db.findLicense(license_key);

    if(!license) return res.json({ valid: false, message: "Invalid key" });
    if(license.expired) return res.json({ valid: false, message: "License expired" });
    if(license.maxAccounts && license.accounts.length >= license.maxAccounts) {
        if(!license.accounts.includes(account))
            return res.json({ valid: false, message: "Max accounts reached" });
    }

    // Register account if new
    if(!license.accounts.includes(account)) {
        license.accounts.push(account);
        db.updateLicense(license);
    }

    res.json({
        valid: true,
        expiry: license.expiryDate,
        features: license.features
    });
});
```

---

## MQL4 Specific Considerations

### Account Number Check (MQL4)

```mql4
bool CheckLicense() {
    int account = AccountNumber();  // MQL4 function
    // ... same logic but with int instead of long
}
```

### Hardware ID (MQL4 via DLL)

Can use Windows API via DLL to get hardware identifiers for machine-specific licensing.

---

## Complete License System Architecture

```
EA (MQL5)                    License Server (Node.js)
    |                              |
    |-- POST /validate ----------->|
    |   {key, account, broker}     |-- Check DB
    |                              |-- Validate
    |<-- {valid, expiry, features}-|
    |                              |
    |-- Cache expiry locally       |
    |-- Re-check every hour        |
    |                              |
    |-- POST /heartbeat ---------->| (optional)
    |   {account, balance, status} |-- Track usage
```
