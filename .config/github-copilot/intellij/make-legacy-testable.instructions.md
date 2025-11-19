---
applyTo: '**'
description: 'Instructions for introducing seams to make untestable legacy code testable'
---

# Making Legacy Code Testable

Some legacy code is **fundamentally untestable** due to hard-coded dependencies (filesystem, database, network, time, randomness). Before you can write characterization tests, you must introduce **seams** - minimal changes that enable dependency injection without altering behavior.

## The Paradox and Its Solution

**The Problem**: You can't change untested code, but some code can't be tested without changes.

**The Solution**: Introduce seams using the **"Preserve Signature, Wrap and Extract"** pattern:
1. Keep the original function signature unchanged (behavior preserved for callers)
2. Extract hard dependencies to parameters with default values (backward compatible)
3. Now you can test by injecting test doubles

This is the **only acceptable exception** to the "never change untested code" rule, because:
- The original signature remains unchanged
- Default parameters preserve existing behavior
- The change is mechanical and verifiable by inspection
- Tests can now be written

## Core Principles for Introducing Seams

### What is a Seam?

A **seam** is a place where you can alter behavior without editing the code itself - by injecting different dependencies during tests.

**Bad seam** (requires code changes to test):
```javascript
// Legacy ES3/ES5 code
function processFile() {
    var data = fs.readFileSync('data.txt', 'utf8');  // Hard-coded
    return data.toUpperCase();
}
```

**Good seam** (testable via injection):
```javascript
// Compatible with ES3/ES5
function processFile(readFile) {
    readFile = readFile || fs.readFileSync;
    var data = readFile('data.txt', 'utf8');
    return data.toUpperCase();
}
```

### Observable Test Doubles

**Seams should be observable**: test doubles should capture arguments passed to dependencies, enabling characterization tests to create snapshots/golden masters of the interaction flow.

**Best seam** (testable and observable):
```javascript
// Test double captures arguments for golden master
var capturedCalls = [];
var fakeReadFile = function(path, encoding) {
    capturedCalls.push({ path: path, encoding: encoding });
    return 'file content';
};

var result = processFile(fakeReadFile);

// Now we have a snapshot of the interaction
expect(capturedCalls).toEqual([
    { path: 'data.txt', encoding: 'utf8' }
]);
```

**Why observable test doubles matter:**
- Document the interaction flow between components
- Create golden masters that protect against behavioral drift
- Verify not just what is returned, but how dependencies are called
- Alert you immediately when dependency contracts change

### The Seam Introduction Strategy

1. **Identify hard dependencies**: Filesystem, database, network, time, randomness, globals
2. **Extract to parameters with defaults**: Add parameters with current implementation as default
3. **Use version-compatible syntax**: Match the language version of the legacy code
4. **Follow naming priority rules**: Avoid shadowing, then match existing names, then use descriptive names
5. **Preserve exact behavior**: Existing callers get identical behavior
6. **Verify by inspection**: The change is mechanical and obviously behavior-preserving
7. **Write characterization tests with observable test doubles**: Capture interaction flow
8. **Refactor with confidence**: Tests protect you during further cleanup

### Handling Multiple Hard Dependencies

When a function has multiple hard dependencies (filesystem + database + network), introduce seams **one at a time** in separate commits.

**Priority order** (most intrusive first):
1. **Network** (slowest, most fragile, hardest to control)
2. **Filesystem** (I/O bound, state dependent)
3. **Database** (stateful, setup intensive)
4. **Time** (non-deterministic)
5. **Random** (non-deterministic)

**Workflow:**
1. Introduce first seam (e.g., network)
2. Write characterization tests with observable test double
3. Commit seam + tests together
4. Introduce second seam (e.g., filesystem)
5. Update characterization tests to inject both dependencies
6. Commit seam + updated tests
7. Repeat for remaining dependencies

**Example progression:**

```javascript
// Step 0: Original untestable code
function syncUsers() {
    var response = httpClient.get('/api/users');
    var users = JSON.parse(response.body);
    fs.writeFileSync('users.json', JSON.stringify(users));
    db.execute('UPDATE sync_log SET last_sync = NOW()');
    return users.length;
}

// Step 1: Introduce network seam (most intrusive)
function syncUsers(httpClient) {
    httpClient = httpClient || defaultHttpClient;
    var response = httpClient.get('/api/users');
    var users = JSON.parse(response.body);
    fs.writeFileSync('users.json', JSON.stringify(users));
    db.execute('UPDATE sync_log SET last_sync = NOW()');
    return users.length;
}

// Characterization test after Step 1
it('should fetch users and write to file and db', function() {
    var httpCalls = [];
    var fakeHttp = {
        get: function(url) {
            httpCalls.push({ method: 'get', url: url });
            return { body: '[{"id":1,"name":"Alice"}]' };
        }
    };
    
    var result = syncUsers(fakeHttp);
    
    expect(result).toBe(1);
    expect(httpCalls).toEqual([{ method: 'get', url: '/api/users' }]);
    // Note: filesystem and db still real - tested in next iteration
});

// Step 2: Introduce filesystem seam
function syncUsers(httpClient, fileSystem) {
    httpClient = httpClient || defaultHttpClient;
    fileSystem = fileSystem || fs;
    var response = httpClient.get('/api/users');
    var users = JSON.parse(response.body);
    fileSystem.writeFileSync('users.json', JSON.stringify(users));
    db.execute('UPDATE sync_log SET last_sync = NOW()');
    return users.length;
}

// Updated characterization test after Step 2
it('should fetch users and write to file and db', function() {
    var httpCalls = [];
    var fakeHttp = {
        get: function(url) {
            httpCalls.push({ method: 'get', url: url });
            return { body: '[{"id":1,"name":"Alice"}]' };
        }
    };
    
    var fsCalls = [];
    var fakeFs = {
        writeFileSync: function(path, content) {
            fsCalls.push({ method: 'writeFileSync', path: path, content: content });
        }
    };
    
    var result = syncUsers(fakeHttp, fakeFs);
    
    expect(result).toBe(1);
    expect(httpCalls).toEqual([{ method: 'get', url: '/api/users' }]);
    expect(fsCalls).toEqual([
        { method: 'writeFileSync', path: 'users.json', content: '[{"id":1,"name":"Alice"}]' }
    ]);
    // Note: db still real - tested in next iteration
});

// Step 3: Introduce database seam
function syncUsers(httpClient, fileSystem, database) {
    httpClient = httpClient || defaultHttpClient;
    fileSystem = fileSystem || fs;
    database = database || db;
    var response = httpClient.get('/api/users');
    var users = JSON.parse(response.body);
    fileSystem.writeFileSync('users.json', JSON.stringify(users));
    database.execute('UPDATE sync_log SET last_sync = NOW()');
    return users.length;
}

// Final characterization test with all seams
it('should fetch users and write to file and db', function() {
    var httpCalls = [];
    var fakeHttp = {
        get: function(url) {
            httpCalls.push({ method: 'get', url: url });
            return { body: '[{"id":1,"name":"Alice"}]' };
        }
    };
    
    var fsCalls = [];
    var fakeFs = {
        writeFileSync: function(path, content) {
            fsCalls.push({ method: 'writeFileSync', path: path, content: content });
        }
    };
    
    var dbCalls = [];
    var fakeDb = {
        execute: function(query) {
            dbCalls.push({ method: 'execute', query: query });
        }
    };
    
    var result = syncUsers(fakeHttp, fakeFs, fakeDb);
    
    expect(result).toBe(1);
    expect(httpCalls).toEqual([{ method: 'get', url: '/api/users' }]);
    expect(fsCalls).toEqual([
        { method: 'writeFileSync', path: 'users.json', content: '[{"id":1,"name":"Alice"}]' }
    ]);
    expect(dbCalls).toEqual([
        { method: 'execute', query: 'UPDATE sync_log SET last_sync = NOW()' }
    ]);
});
```

**Why one seam at a time:**
- Smaller, reviewable commits
- Each step is independently verifiable
- Easier to rollback if issues arise
- Tests evolve incrementally with the code
- Reduces cognitive load

## Naming Seams: Priority Rules

When naming seam parameters, follow these priorities **in order**:

### Priority 1: Don't Break Scope or Shadow

**Never introduce a parameter name that:**
- Shadows an imported module
- Shadows a built-in type or function
- Creates scope confusion
- Breaks existing variable resolution

### Priority 2: Match Existing Names

**When Priority 1 is satisfied**, match the original variable/dependency name to minimize git diff.

### Priority 3: Use Descriptive Names

**When Priorities 1-2 cannot be satisfied**, choose a clear, descriptive name.

### Examples

#### ✅ Priority 1 Violation Avoided

```javascript
// Before: uses fs module
var fs = require('fs');

function loadConfig() {
    var content = fs.readFileSync('config.json', 'utf8');
    return JSON.parse(content);
}

// WRONG: Shadows the fs module (Priority 1 violation)
function loadConfig(fs) {  // ❌ Shadows require('fs')
    fs = fs || require('fs');
    var content = fs.readFileSync('config.json', 'utf8');
    return JSON.parse(content);
}

// CORRECT: Different name avoids shadowing
function loadConfig(fileSystem) {  // ✅ Priority 3: descriptive name
    fileSystem = fileSystem || fs;
    var content = fileSystem.readFileSync('config.json', 'utf8');
    return JSON.parse(content);
}
```

#### ✅ Priority 2: Match Existing Name

```javascript
// Before: uses custom variable myHttpClient
function fetchData(url) {
    var response = myHttpClient.get(url);
    return response.data;
}

// CORRECT: Match existing name (no shadowing)
function fetchData(url, myHttpClient) {  // ✅ Priority 2: matched name
    myHttpClient = myHttpClient || defaultHttpClient;
    var response = myHttpClient.get(url);
    return response.data;
}
```

#### ✅ Priority 1 Violation Avoided (Built-in)

```javascript
// Before: Date is a built-in
function isExpired(expiryDate) {
    var now = new Date();
    return expiryDate < now;
}

// WRONG: Shadows built-in Date
function isExpired(expiryDate, Date) {  // ❌ Shadows built-in
    Date = Date || Date;  // ❌ Infinite loop!
    var now = new Date();
    return expiryDate < now;
}

// CORRECT: Use descriptive name
function isExpired(expiryDate, dateProvider) {  // ✅ Priority 3: descriptive name
    dateProvider = dateProvider || Date;
    var now = new dateProvider();
    return expiryDate < now;
}
```

## Naming Patterns by Dependency Type

### File System (ES3/ES5)

```javascript
// When fs is imported module, use fileSystem to avoid shadowing
var fs = require('fs');

function processFile(fileSystem) {
    fileSystem = fileSystem || fs;
    var data = fileSystem.readFileSync('data.txt', 'utf8');
    return data.toUpperCase();
}
```

### Database (PHP 5.x)

```php
// PHP 5.x requires explicit null checks with === operator
// because the ?? null coalescing operator doesn't exist until PHP 7+
function get_user($id, $db = null) {
    if ($db === null) {  // Explicit check required in PHP 5.x
        global $db;
    }
    $result = mysqli_query($db, "SELECT * FROM users WHERE id = $id");
    return mysqli_fetch_assoc($result);
}
```

### HTTP Client (ES3/ES5)

```javascript
// When using built-in XMLHttpRequest, use descriptive name
function fetchUser(id, callback, httpClient) {
    httpClient = httpClient || defaultHttpClient;
    var response = httpClient.get('/api/users/' + id);
    callback(response);
}
```

### Time Provider (ES3/ES5)

```javascript
// Date is a built-in, use dateProvider to avoid shadowing
function isExpired(expiryDate, dateProvider) {
    dateProvider = dateProvider || Date;
    var now = new dateProvider();
    return expiryDate < now;
}
```

### Random Generator (ES3/ES5)

```javascript
// Math is a built-in, use randomProvider to avoid shadowing
function generateId(randomProvider) {
    randomProvider = randomProvider || Math.random;
    return randomProvider().toString(36).substring(7);
}
```

## Seam Introduction Guide

### Before: Verification Checklist

Before introducing a seam, verify:

- [ ] **Matches language version**: Does the seam use syntax compatible with the codebase's language version?
- [ ] **Doesn't break scope**: Does the parameter name avoid shadowing modules, built-ins, or creating scope conflicts?
- [ ] **Identifies real untestability**: Is the dependency truly preventing tests?
- [ ] **Preserves signature compatibility**: Can existing callers continue without changes?
- [ ] **One dependency at a time**: Are you introducing only one seam per commit?

### During: What to Do

- ✅ Match the exact language version and syntax of the legacy code
- ✅ Add one parameter with null/undefined check and fallback
- ✅ Follow naming priority rules (no shadowing → match existing → descriptive)
- ✅ Replace direct dependency call with parameter call
- ✅ Keep changes minimal (as few lines as possible)
- ✅ Verify existing callers still work without modification
- ✅ Document the seam's purpose in a comment if not obvious

### During: What NOT to Do

- ❌ Do not use modern syntax (arrow functions, default params, etc.) if code is ES3/ES5, PHP 5.x, Python 2.x
- ❌ Do not refactor beyond adding the seam
- ❌ Do not change the function's algorithm or logic
- ❌ Do not rename variables or restructure code
- ❌ Do not add multiple seams at once (one dependency at a time)
- ❌ Do not change return types or error handling
- ❌ Do not inject abstractions (interfaces) before writing tests
- ❌ Do not create new classes/modules just for injection
- ❌ Do not change existing callers

### After: Verification Checklist

Once the seam is introduced:

- [ ] **Write characterization tests immediately**: Use observable test doubles
- [ ] **Capture interaction flow**: Assert on arguments passed to dependencies
- [ ] **Commit seam and tests together**: Proves behavior preservation
- [ ] **Tests pass**: Characterization tests verify behavior is unchanged
- [ ] **Ready for next seam**: If multiple dependencies exist, repeat for next one

## Example: Complete Seam Introduction (ES3/ES5)

### Before: Untestable Code

```javascript
// Legacy ES3/ES5 code
function importUsers() {
    var content = fs.readFileSync('users.csv', 'utf8');
    var lines = content.split('\n');
    var users = [];
    
    for (var i = 1; i < lines.length; i++) {
        var parts = lines[i].split(',');
        var name = parts[0];
        var email = parts[1];
        var query = "INSERT INTO users (name, email) VALUES ('" + name + "', '" + email + "')";
        db.execute(query);
        users.push({ name: name, email: email });
    }
    
    return users;
}
```

### Step 1: Introduce Seams (Minimal Change, ES3/ES5 Compatible)

```javascript
// Add seams for file I/O and database (ES3/ES5 compatible)
function importUsers(fileSystem, database) {
    fileSystem = fileSystem || fs;
    database = database || db;
    
    var content = fileSystem.readFileSync('users.csv', 'utf8');
    var lines = content.split('\n');
    var users = [];
    
    for (var i = 1; i < lines.length; i++) {
        var parts = lines[i].split(',');
        var name = parts[0];
        var email = parts[1];
        var query = "INSERT INTO users (name, email) VALUES ('" + name + "', '" + email + "')";
        database.execute(query);
        users.push({ name: name, email: email });
    }
    
    return users;
}
```

### Step 2: Write Characterization Tests with Observable Test Doubles (ES3/ES5)

```javascript
// Characterization test with observable test doubles (ES3/ES5)
describe('importUsers - characterization', function() {
    it('should parse CSV and insert users', function() {
        // Observable fake filesystem
        var fsCalls = [];
        var fakeFs = {
            readFileSync: function(path, encoding) {
                fsCalls.push({ method: 'readFileSync', path: path, encoding: encoding });
                return 'name,email\nAlice,alice@test.com\nBob,bob@test.com';
            }
        };
        
        // Observable fake database
        var dbCalls = [];
        var fakeDb = {
            execute: function(query) {
                dbCalls.push({ method: 'execute', query: query });
            }
        };
        
        var result = importUsers(fakeFs, fakeDb);
        
        // Assert on result
        expect(result).toEqual([
            { name: 'Alice', email: 'alice@test.com' },
            { name: 'Bob', email: 'bob@test.com' }
        ]);
        
        // Golden master: filesystem interactions
        expect(fsCalls).toEqual([
            { method: 'readFileSync', path: 'users.csv', encoding: 'utf8' }
        ]);
        
        // Golden master: database interactions
        expect(dbCalls).toEqual([
            { method: 'execute', query: "INSERT INTO users (name, email) VALUES ('Alice', 'alice@test.com')" },
            { method: 'execute', query: "INSERT INTO users (name, email) VALUES ('Bob', 'bob@test.com')" }
        ]);
    });
});
```

### Step 3: Refactor with Test Protection

Now that you have tests with golden masters, you can safely refactor:

```javascript
// Extract CSV parsing (safe because tests exist)
function parseCSV(content) {
    var lines = content.split('\n');
    var users = [];
    for (var i = 1; i < lines.length; i++) {
        var parts = lines[i].split(',');
        users.push({ name: parts[0], email: parts[1] });
    }
    return users;
}

function importUsers(fileSystem, database) {
    fileSystem = fileSystem || fs;
    database = database || db;
    
    var content = fileSystem.readFileSync('users.csv', 'utf8');
    var users = parseCSV(content);
    
    for (var i = 0; i < users.length; i++) {
        var user = users[i];
        var query = "INSERT INTO users (name, email) VALUES ('" + user.name + "', '" + user.email + "')";
        database.execute(query);
    }
    
    return users;
}
```

**The golden master test still passes**: the filesystem and database interactions remain identical, proving behavior preservation.

## Creating Reusable Observable Test Doubles

To avoid repeating capture logic, create reusable observable test double helpers:

### JavaScript/TypeScript (ES3/ES5)

```javascript
// Reusable observable spy function
function createSpy(name, returnValue) {
    var calls = [];
    var spy = function() {
        var args = Array.prototype.slice.call(arguments);
        calls.push({ name: name, args: args });
        return returnValue;
    };
    spy.calls = calls;
    return spy;
}

// Usage
var readFileSpy = createSpy('readFileSync', 'file content');
processFile(readFileSpy);
expect(readFileSpy.calls).toEqual([
    { name: 'readFileSync', args: ['data.txt', 'utf8'] }
]);
```

### PHP (5.x)

```php
// Reusable observable spy class
class Spy {
    public $name;
    public $calls = array();
    private $return_value;
    
    public function __construct($name, $return_value = null) {
        $this->name = $name;
        $this->return_value = $return_value;
    }
    
    public function __invoke() {
        $args = func_get_args();
        $this->calls[] = array('name' => $this->name, 'args' => $args);
        return $this->return_value;
    }
}

// Usage
$read_file_spy = new Spy('file_get_contents', '{"key":"value"}');
$result = load_data('data.json', $read_file_spy);
$this->assertEquals(
    array(array('name' => 'file_get_contents', 'args' => array('data.json'))),
    $read_file_spy->calls
);
```

### Python (2.x)

```python
# Reusable observable spy class
class Spy(object):
    def __init__(self, name, return_value=None):
        self.name = name
        self.return_value = return_value
        self.calls = []
    
    def __call__(self, *args, **kwargs):
        self.calls.append({'name': self.name, 'args': args, 'kwargs': kwargs})
        return self.return_value

# Usage
read_file_spy = Spy('open', StringIO.StringIO('{"key":"value"}'))
result = load_data('data.json', read_file_spy)
self.assertEqual(
    [{'name': 'open', 'args': ('data.json', 'r'), 'kwargs': {}}],
    read_file_spy.calls
)
```

## Snapshot Testing

For golden master testing, use snapshot testing libraries to automatically manage captured interactions.

### General Workflow

1. **First run**: Snapshot is created automatically
2. **Subsequent runs**: Current output is compared against the snapshot
3. **Intentional changes**: Update snapshots with a flag (e.g., `--update-snapshots`)
4. **Version control**: Snapshots are committed alongside tests

### Language-Specific Tools

#### PHP: Spatie Snapshots

**If the project uses `spatie/phpunit-snapshot-assertions`:**

```php
use PHPUnit\Framework\TestCase;
use Spatie\Snapshots\MatchesSnapshots;

class LoadDataTest extends TestCase {
    use MatchesSnapshots;
    
    public function test_should_parse_json_from_file_content() {
        $captured_calls = array();
        $fake_reader = function($path) use (&$captured_calls) {
            $captured_calls[] = array('path' => $path);
            return '{"name": "test", "value": 42}';
        };
        
        $result = load_data('data.json', $fake_reader);
        
        $this->assertEquals(array('name' => 'test', 'value' => 42), $result);
        $this->assertMatchesJsonSnapshot($captured_calls);
    }
}
```

Update snapshots: `phpunit --update-snapshots`

#### JavaScript/TypeScript: Jest/Vitest Snapshots

```javascript
// Using Jest or Vitest
describe('importUsers', () => {
    it('should parse CSV and insert users', () => {
        const fsCalls = [];
        const fakeFs = {
            readFileSync: (path, encoding) => {
                fsCalls.push({ method: 'readFileSync', path, encoding });
                return 'name,email\nAlice,alice@test.com';
            }
        };
        
        const result = importUsers(fakeFs, fakeDb);
        
        expect(fsCalls).toMatchSnapshot();
    });
});
```

Update snapshots: `jest --updateSnapshot` or `vitest -u`

#### Python: pytest-snapshot

**If the project uses `pytest-snapshot`:**

```python
def test_should_parse_json_from_file_content(snapshot):
    captured_calls = []
    
    def fake_reader(path):
        captured_calls.append({'path': path})
        return '{"name": "test", "value": 42}'
    
    result = load_data('data.json', fake_reader)
    
    assert result == {'name': 'test', 'value': 42}
    snapshot.assert_match(captured_calls)
```

Update snapshots: `pytest --snapshot-update`

### Advantages of Snapshot Testing

- Automatic snapshot management
- Cleaner test code (no manual array comparisons)
- Version control friendly (snapshots are committed)
- Easy to review changes in PRs
- Detects unintended behavioral changes immediately

## When to Introduce Seams

**Introduce seams when:**
- Code has hard-coded dependencies (file, network, DB, time, random)
- You need to write characterization tests but can't
- The seam is minimal (one parameter with null/undefined check)
- The seam uses syntax compatible with the codebase's language version
- Existing callers won't need changes
- The change is mechanical and obviously safe

**Don't introduce seams when:**
- Code is already testable
- You can write characterization tests without changes
- The seam would require changing many callers
- You'd need to use modern syntax in legacy code
- You'd need to create complex abstractions
- You're not about to write tests immediately

## After Introducing Seams

Once seams are in place:

1. **Write characterization tests** immediately with observable test doubles
2. **Commit seam and tests together** (proves behavior preservation)
3. **Refactor with confidence** (tests protect you)
4. **Extract pure functions** (move logic away from I/O)
5. **Add unit tests** for extracted logic
6. **Consider better abstractions** (now that you understand the code)

## Remember

**Introducing seams is the only acceptable pre-test code change** because it's:
- Mechanical and verifiable by inspection
- Backward compatible (null/undefined checks with fallbacks)
- Version-compatible (uses same syntax as existing code)
- Minimal (one parameter at a time)
- Scope-safe (doesn't break imports or shadowing rules)
- Observable (enables golden master testing)
- Enables testing (the whole point)

After adding seams, **immediately write characterization tests with observable test doubles**. The seam isn't done until tests exist that capture the interaction flow as a golden master. This three-step dance - seam, observable test doubles, golden master assertions - is the key to safely modernizing untestable legacy code, regardless of the language version.

**Golden masters protect you**: any change that alters the arguments passed to dependencies will fail the characterization test, alerting you immediately to potential behavior changes.