# ioBroker Adapter Development with GitHub Copilot

**Version:** 0.5.7
**Template Source:** https://github.com/DrozmotiX/ioBroker-Copilot-Instructions

This file contains instructions and best practices for GitHub Copilot when working on ioBroker adapter development.

---

## 📑 Table of Contents

1. [Project Context](#project-context)
2. [Code Quality & Standards](#code-quality--standards)
   - [Code Style Guidelines](#code-style-guidelines)
   - [ESLint Configuration](#eslint-configuration)
3. [Testing](#testing)
   - [Unit Testing](#unit-testing)
   - [Integration Testing](#integration-testing)
   - [API Testing with Credentials](#api-testing-with-credentials)
4. [Development Best Practices](#development-best-practices)
   - [Dependency Management](#dependency-management)
   - [HTTP Client Libraries](#http-client-libraries)
   - [Error Handling](#error-handling)
5. [Admin UI Configuration](#admin-ui-configuration)
   - [JSON-Config Setup](#json-config-setup)
   - [Translation Management](#translation-management)
6. [Documentation](#documentation)
   - [README Updates](#readme-updates)
   - [Changelog Management](#changelog-management)
7. [CI/CD & GitHub Actions](#cicd--github-actions)
   - [Workflow Configuration](#workflow-configuration)
   - [Testing Integration](#testing-integration)

---

## Project Context

You are working on an ioBroker adapter. ioBroker is an integration platform for the Internet of Things, focused on building smart home and industrial IoT solutions. Adapters are plugins that connect ioBroker to external systems, devices, or services.

## Adapter-Specific Context
- **Adapter Name**: omnicomm-lls
- **Primary Function**: Read data from Omnicomm LLS (Liquid Level Sensor) fuel sensors via serial communication
- **Target Devices**: Omnicomm LLS fuel sensors with RS-485/RS-232 interface
- **Communication Protocol**: Serial communication using Modbus-like protocol
- **Key Dependencies**: serialport, @serialport/parser-inter-byte-timeout
- **Hardware Requirements**: Serial port connection to Omnicomm LLS sensors
- **Data Types**: Fuel level readings, temperature, sensor status information
- **Configuration**: Serial port path, baud rate, sensor address, polling intervals

### Omnicomm LLS Specific Patterns
- Use SerialPort for RS-485/RS-232 communication with fuel sensors
- Implement proper timeout handling for sensor responses
- Parse binary data from sensor responses correctly
- Handle multiple sensor addresses on the same serial bus
- Implement proper error handling for communication failures
- Support configurable polling intervals to prevent sensor overload
- Convert raw sensor values to meaningful fuel level measurements

---

## Code Quality & Standards

### Code Style Guidelines

- Follow JavaScript/TypeScript best practices
- Use async/await for asynchronous operations
- Implement proper resource cleanup in `unload()` method
- Use semantic versioning for adapter releases
- Include proper JSDoc comments for public methods
- Handle Buffer operations safely for binary serial data
- Implement proper error boundaries for hardware communication

**Timer and Resource Cleanup Example:**
```javascript
private connectionTimer?: NodeJS.Timeout;

async onReady() {
  this.connectionTimer = setInterval(() => this.checkConnection(), 30000);
}

onUnload(callback) {
  try {
    if (this.connectionTimer) {
      clearInterval(this.connectionTimer);
      this.connectionTimer = undefined;
    }
    callback();
  } catch (e) {
    callback();
  }
}
```

### ESLint Configuration

**CRITICAL:** ESLint validation must run FIRST in your CI/CD pipeline, before any other tests. This "lint-first" approach catches code quality issues early.

#### Setup
```bash
npm install --save-dev eslint @iobroker/eslint-config
```

#### Configuration (.eslintrc.json)
```json
{
  "extends": "@iobroker/eslint-config",
  "rules": {
    // Add project-specific rule overrides here if needed
  }
}
```

#### Package.json Scripts
```json
{
  "scripts": {
    "lint": "eslint --max-warnings 0 .",
    "lint:fix": "eslint . --fix"
  }
}
```

#### Best Practices
1. ✅ Run ESLint before committing — fix ALL warnings, not just errors
2. ✅ Use `lint:fix` for auto-fixable issues
3. ✅ Don't disable rules without documentation
4. ✅ Lint all relevant files (main code, tests, build scripts)
5. ✅ Keep `@iobroker/eslint-config` up to date
6. ✅ **ESLint warnings are treated as errors in CI** (`--max-warnings 0`). The `lint` script above already includes this flag — run `npm run lint` to match CI behavior locally

#### Common Issues
- **Unused variables**: Remove or prefix with underscore (`_variable`)
- **Missing semicolons**: Run `npm run lint:fix`
- **Indentation**: Use 4 spaces (ioBroker standard)
- **console.log**: Replace with `adapter.log.debug()` or remove

---

## Testing

### Unit Testing
- Use Jest as the primary testing framework for ioBroker adapters
- Create tests for all adapter main functions and helper methods
- Test error handling scenarios and edge cases
- Mock external API calls and hardware dependencies
- For adapters connecting to APIs/devices not reachable by internet, provide example data files to allow testing of functionality without live connections
- Example test structure:
  ```javascript
  describe('AdapterName', () => {
    let adapter;
    
    beforeEach(() => {
      // Setup test adapter instance
    });
    
    test('should initialize correctly', () => {
      // Test adapter initialization
    });
  });
  ```

### Serial Communication Testing for Omnicomm LLS
- Mock SerialPort for unit tests to avoid hardware dependencies
- Create sample binary response data for different sensor states
- Test timeout scenarios and communication errors
- Validate data parsing from sensor binary responses
- Test multiple sensor address handling
- Example mock setup:
  ```javascript
  const mockSerialPort = {
    write: jest.fn(),
    on: jest.fn(),
    close: jest.fn(),
    isOpen: true
  };
  jest.mock('serialport', () => ({
    SerialPort: jest.fn(() => mockSerialPort)
  }));
  ```

### Integration Testing

**IMPORTANT**: Use the official `@iobroker/testing` framework for all integration tests. This is the ONLY correct way to test ioBroker adapters.

**Official Documentation**: https://github.com/ioBroker/testing

#### Framework Structure
Integration tests MUST follow this exact pattern:

```javascript
const path = require('path');
const { tests } = require('@iobroker/testing');

// Define test coordinates or configuration
const TEST_COORDINATES = '52.520008,13.404954'; // Berlin
const wait = ms => new Promise(resolve => setTimeout(resolve, ms));

// Use tests.integration() with defineAdditionalTests
tests.integration(path.join(__dirname, '..'), {
    defineAdditionalTests({ suite }) {
        suite('Test adapter with specific configuration', (getHarness) => {
            let harness;

            before(() => {
                harness = getHarness();
            });

            it('should configure and start adapter', function () {
                return new Promise(async (resolve, reject) => {
                    try {
                        harness = getHarness();
                        
                        // Get adapter object using promisified pattern
                        const obj = await new Promise((res, rej) => {
                            harness.objects.getObject('system.adapter.your-adapter.0', (err, o) => {
                                if (err) return rej(err);
                                res(o);
                            });
                        });
                        
                        if (!obj) {
                            return reject(new Error('Adapter object not found'));
                        }

                        // Configure adapter properties
                        Object.assign(obj.native, {
                            position: TEST_COORDINATES,
                            createCurrently: true,
                            createHourly: true,
                            createDaily: true,
                            // Add other configuration as needed
                        });

                        // Set the updated configuration
                        harness.objects.setObject(obj._id, obj);

                        console.log('✅ Step 1: Configuration written, starting adapter...');
                        
                        // Start adapter and wait
                        await harness.startAdapterAndWait();
                        
                        console.log('✅ Step 2: Adapter started');

                        // Wait for adapter to process data
                        const waitMs = 15000;
                        await wait(waitMs);

                        console.log('🔍 Step 3: Checking states after adapter run...');
                        
                        // Get all states created by adapter
                        const stateIds = await harness.dbConnection.getStateIDs('your-adapter.0.*');
                        
                        console.log(`📊 Found ${stateIds.length} states`);

                        if (stateIds.length > 0) {
                            console.log('✅ Adapter successfully created states');
                            
                            // Show sample of created states
                            const allStates = await new Promise((res, rej) => {
                                harness.states.getStates(stateIds, (err, states) => {
                                    if (err) return rej(err);
                                    res(states || []);
                                });
                            });
                            
                            console.log('📋 Sample states created:');
                            stateIds.slice(0, 5).forEach((stateId, index) => {
                                const state = allStates[index];
                                console.log(`   ${stateId}: ${state && state.val !== undefined ? state.val : 'undefined'}`);
                            });
                            
                            await harness.stopAdapter();
                            resolve(true);
                        } else {
                            console.log('❌ No states were created by the adapter');
                            reject(new Error('Adapter did not create any states'));
                        }
                    } catch (error) {
                        reject(error);
                    }
                });
            }).timeout(40000);
        });
    }
});
```

#### Testing Both Success AND Failure Scenarios

**IMPORTANT**: For every "it works" test, implement corresponding "it doesn't work and fails" tests. This ensures proper error handling and validates that your adapter fails gracefully when expected.

```javascript
// Example: Testing successful configuration
it('should configure and start adapter with valid configuration', function () {
    return new Promise(async (resolve, reject) => {
        // ... successful configuration test as shown above
    });
}).timeout(40000);

// Example: Testing failure scenarios
it('should NOT create daily states when daily is disabled', function () {
    return new Promise(async (resolve, reject) => {
        try {
            harness = getHarness();
            
            console.log('🔍 Step 1: Fetching adapter object...');
            const obj = await new Promise((res, rej) => {
                harness.objects.getObject('system.adapter.your-adapter.0', (err, o) => {
                    if (err) return rej(err);
                    res(o);
                });
            });
            
            if (!obj) return reject(new Error('Adapter object not found'));
            console.log('✅ Step 1.5: Adapter object loaded');

            console.log('🔍 Step 2: Updating adapter config...');
            Object.assign(obj.native, {
                position: TEST_COORDINATES,
                createCurrently: false,
                createHourly: true,
                createDaily: false, // Daily disabled for this test
            });

            await new Promise((res, rej) => {
                harness.objects.setObject(obj._id, obj, (err) => {
                    if (err) return rej(err);
                    console.log('✅ Step 2.5: Adapter object updated');
                    res(undefined);
                });
            });

            console.log('🔍 Step 3: Starting adapter...');
            await harness.startAdapterAndWait();
            console.log('✅ Step 4: Adapter started');

            console.log('⏳ Step 5: Waiting 20 seconds for states...');
            await new Promise((res) => setTimeout(res, 20000));

            console.log('🔍 Step 6: Fetching state IDs...');
            const stateIds = await harness.dbConnection.getStateIDs('your-adapter.0.*');

            console.log(`📊 Step 7: Found ${stateIds.length} total states`);

            const hourlyStates = stateIds.filter((key) => key.includes('hourly'));
            if (hourlyStates.length > 0) {
                console.log(`✅ Step 8: Correctly ${hourlyStates.length} hourly weather states created`);
            } else {
                console.log('❌ Step 8: No hourly states created (test failed)');
                return reject(new Error('Expected hourly states but found none'));
            }

            // Check daily states should NOT be present
            const dailyStates = stateIds.filter((key) => key.includes('daily'));
            if (dailyStates.length === 0) {
                console.log(`✅ Step 9: No daily states found as expected`);
            } else {
                console.log(`❌ Step 9: Daily states present (${dailyStates.length}) (test failed)`);
                return reject(new Error('Expected no daily states but found some'));
            }

            await harness.stopAdapter();
            console.log('🛑 Step 10: Adapter stopped');

            resolve(true);
        } catch (error) {
            reject(error);
        }
    });
}).timeout(40000);

// Example: Testing missing required configuration  
it('should handle missing required configuration properly', function () {
    return new Promise(async (resolve, reject) => {
        try {
            harness = getHarness();
            
            console.log('🔍 Step 1: Fetching adapter object...');
            const obj = await new Promise((res, rej) => {
                harness.objects.getObject('system.adapter.your-adapter.0', (err, o) => {
                    if (err) return rej(err);
                    res(o);
                });
            });
            
            if (!obj) return reject(new Error('Adapter object not found'));

            console.log('🔍 Step 2: Removing required configuration...');
            // Remove required configuration to test failure handling
            delete obj.native.position; // This should cause failure or graceful handling

            await new Promise((res, rej) => {
                harness.objects.setObject(obj._id, obj, (err) => {
                    if (err) return rej(err);
                    res(undefined);
                });
            });

            console.log('🔍 Step 3: Starting adapter...');
            await harness.startAdapterAndWait();

            console.log('⏳ Step 4: Waiting for adapter to process...');
            await new Promise((res) => setTimeout(res, 10000));

            console.log('🔍 Step 5: Checking adapter behavior...');
            const stateIds = await harness.dbConnection.getStateIDs('your-adapter.0.*');

            // Check if adapter handled missing configuration gracefully
            if (stateIds.length === 0) {
                console.log('✅ Adapter properly handled missing configuration - no invalid states created');
                resolve(true);
            } else {
                // If states were created, check if they're in error state
                const connectionState = await new Promise((res, rej) => {
                    harness.states.getState('your-adapter.0.info.connection', (err, state) => {
                        if (err) return rej(err);
                        res(state);
                    });
                });
                
                if (!connectionState || connectionState.val === false) {
                    console.log('✅ Adapter properly failed with missing configuration');
                    resolve(true);
                } else {
                    console.log('❌ Adapter should have failed or handled missing config gracefully');
                    reject(new Error('Adapter should have handled missing configuration'));
                }
            }

            await harness.stopAdapter();
        } catch (error) {
            console.log('✅ Adapter correctly threw error with missing configuration:', error.message);
            resolve(true);
        }
    });
}).timeout(40000);
```

#### Advanced State Access Patterns

For testing adapters that create multiple states, use bulk state access methods to efficiently verify large numbers of states:

```javascript
it('should create and verify multiple states', () => new Promise(async (resolve, reject) => {
    // Configure and start adapter first...
    harness.objects.getObject('system.adapter.tagesschau.0', async (err, obj) => {
        if (err) {
            console.error('Error getting adapter object:', err);
            reject(err);
            return;
        }

        // Configure adapter as needed
        obj.native.someConfig = 'test-value';
        harness.objects.setObject(obj._id, obj);

        await harness.startAdapterAndWait();

        // Wait for adapter to create states
        setTimeout(() => {
            // Access bulk states using pattern matching
            harness.dbConnection.getStateIDs('tagesschau.0.*').then(stateIds => {
                if (stateIds && stateIds.length > 0) {
                    harness.states.getStates(stateIds, (err, allStates) => {
                        if (err) {
                            console.error('❌ Error getting states:', err);
                            reject(err); // Properly fail the test instead of just resolving
                            return;
                        }

                        // Verify states were created and have expected values
                        const expectedStates = ['tagesschau.0.info.connection', 'tagesschau.0.articles.0.title'];
                        let foundStates = 0;
                        
                        for (const stateId of expectedStates) {
                            if (allStates[stateId]) {
                                foundStates++;
                                console.log(`✅ Found expected state: ${stateId}`);
                            } else {
                                console.log(`❌ Missing expected state: ${stateId}`);
                            }
                        }

                        if (foundStates === expectedStates.length) {
                            console.log('✅ All expected states were created successfully');
                            resolve();
                        } else {
                            reject(new Error(`Only ${foundStates}/${expectedStates.length} expected states were found`));
                        }
                    });
                } else {
                    reject(new Error('No states found matching pattern tagesschau.0.*'));
                }
            }).catch(reject);
        }, 20000); // Allow more time for multiple state creation
    });
})).timeout(45000);
```

#### Integration Testing Patterns for Omnicomm LLS (Serial Communication)

Integration tests specific to serial communication hardware:

```javascript
suite('Test omnicomm-lls adapter with mock serial data', (getHarness) => {
    let harness;

    before(() => {
        harness = getHarness();
    });

    it('should configure and start adapter with serial settings', function () {
        return new Promise(async (resolve, reject) => {
            try {
                harness = getHarness();
                
                // Get adapter object using promisified pattern
                const obj = await new Promise((res, rej) => {
                    harness.objects.getObject('system.adapter.omnicomm-lls.0', (err, o) => {
                        if (err) return rej(err);
                        res(o);
                    });
                });
                
                if (!obj) {
                    return reject(new Error('Adapter object not found'));
                }

                // Configure adapter properties for serial communication
                Object.assign(obj.native, {
                    comPort: '/dev/ttyUSB0',  // Mock port for testing
                    baud: 9600,
                    address: 1,
                    pollTime: 10000
                });

                // Set the updated configuration
                harness.objects.setObject(obj._id, obj);

                console.log('✅ Step 1: Serial configuration written, starting adapter...');
                
                // Start adapter and wait
                await harness.startAdapterAndWait();
                
                console.log('✅ Step 2: Adapter started');

                // Wait for adapter to process data (longer for serial communication)
                const waitMs = 15000;
                await wait(waitMs);

                console.log('🔍 Step 3: Checking states after sensor polling...');
                
                // Check if connection state was set
                const connectionState = await new Promise((res, rej) => {
                    harness.states.getState('omnicomm-lls.0.info.connection', (err, state) => {
                        if (err) return rej(err);
                        res(state);
                    });
                });

                if (!connectionState || connectionState.val !== true) {
                    console.log('ℹ️ Connection state not established (expected in test environment without real serial hardware)');
                }

                resolve();
            } catch (error) {
                reject(error);
            }
        });
    }).timeout(30000); // Longer timeout for serial operations
});

suite('Serial Communication Error Handling', (getHarness) => {
    it('should handle serial port connection failures gracefully', async () => {
        // Test adapter behavior when serial port is not available
        // Verify error logging and connection state updates
    });
    
    it('should recover from timeout errors', async () => {
        // Test timeout handling and retry logic
        // Verify adapter continues attempting communication
    });
    
    it('should parse sensor data correctly', async () => {
        // Test binary data parsing from fuel sensors
        // Verify fuel level calculations and unit conversions
    });
});
```

#### Key Rules

1. ✅ Use `@iobroker/testing` framework
2. ✅ Configure via `harness.objects.setObject()`
3. ✅ Start via `harness.startAdapterAndWait()`
4. ✅ Verify states via `harness.states.getState()`
5. ✅ Allow proper timeouts for async operations
6. ❌ NEVER test API URLs directly
7. ❌ NEVER bypass the harness system

#### Workflow Dependencies

Integration tests should run ONLY after lint and adapter tests pass:

```yaml
integration-tests:
  needs: [check-and-lint, adapter-tests]
  runs-on: ubuntu-22.04
```

### API Testing with Credentials

For adapters connecting to external APIs requiring authentication:

#### Password Encryption for Integration Tests

```javascript
async function encryptPassword(harness, password) {
    const systemConfig = await harness.objects.getObjectAsync("system.config");
    if (!systemConfig?.native?.secret) {
        throw new Error("Could not retrieve system secret for password encryption");
    }

    const secret = systemConfig.native.secret;
    let result = '';
    for (let i = 0; i < password.length; ++i) {
        result += String.fromCharCode(secret[i % secret.length].charCodeAt(0) ^ password.charCodeAt(i));
    }
    return result;
}
```

#### Demo Credentials Testing Pattern

- Use provider demo credentials when available (e.g., `demo@api-provider.com` / `demo`)
- Create separate test file: `test/integration-demo.js`
- Add npm script: `"test:integration-demo": "mocha test/integration-demo --exit"`
- Implement clear success/failure criteria

**Example Implementation:**
```javascript
it("Should connect to API with demo credentials", async () => {
    const encryptedPassword = await encryptPassword(harness, "demo_password");

    await harness.changeAdapterConfig("your-adapter", {
        native: {
            username: "demo@provider.com",
            password: encryptedPassword,
        }
    });

    await harness.startAdapter();
    await new Promise(resolve => setTimeout(resolve, 60000));

    const connectionState = await harness.states.getStateAsync("your-adapter.0.info.connection");

    if (connectionState?.val === true) {
        console.log("✅ SUCCESS: API connection established");
        return true;
    } else {
        throw new Error("API Test Failed: Expected API connection. Check logs for API errors.");
    }
}).timeout(120000);
```

### Performance Testing for Omnicomm LLS
- Test polling intervals don't overload the serial bus
- Verify memory usage doesn't increase over time with continuous polling
- Test multiple sensor handling on same serial connection
- Monitor adapter performance with different baud rates

---

## Development Best Practices

### Dependency Management

- Always use `npm` for dependency management
- Use `npm ci` for installing existing dependencies (respects package-lock.json)
- Use `npm install` only when adding or updating dependencies
- Keep dependencies minimal and focused
- Only update dependencies in separate Pull Requests

**When modifying package.json:**
1. Run `npm install` to sync package-lock.json
2. Commit both package.json and package-lock.json together

**Best Practices:**
- Prefer built-in Node.js modules when possible
- Use `@iobroker/adapter-core` for adapter base functionality
- Avoid deprecated packages
- Document specific version requirements

### HTTP Client Libraries

- **Preferred:** Use native `fetch` API (Node.js 20+ required)
- **Avoid:** `axios` unless specific features are required

**Example with fetch:**
```javascript
try {
  const response = await fetch('https://api.example.com/data');
  if (!response.ok) {
    throw new Error(`HTTP ${response.status}: ${response.statusText}`);
  }
  const data = await response.json();
} catch (error) {
  this.log.error(`API request failed: ${error.message}`);
}
```

**Other Recommendations:**
- **Logging:** Use adapter built-in logging (`this.log.*`)
- **Scheduling:** Use adapter built-in timers and intervals
- **File operations:** Use Node.js `fs/promises`
- **Configuration:** Use adapter config system

### Error Handling

- Always catch and log errors appropriately
- Use adapter log levels (error, warn, info, debug)
- Provide meaningful, user-friendly error messages
- Handle network failures gracefully
- Implement retry mechanisms where appropriate
- Always clean up timers, intervals, and resources in `unload()` method

**General Error Handling Example:**
```javascript
try {
  await this.connectToDevice();
} catch (error) {
  this.log.error(`Failed to connect to device: ${error.message}`);
  this.setState('info.connection', false, true);
  // Implement retry logic if needed
}
```

**Serial Communication Error Patterns for Omnicomm LLS:**

Handle these specific error scenarios for Omnicomm LLS sensors:
- Serial port connection failures
- Sensor response timeouts
- Invalid or corrupted sensor data
- Multiple sensor address conflicts
- Hardware disconnection/reconnection

```javascript
// Example error handling pattern
try {
    const response = await this.serialRequest(command);
    const data = this.parseOmnicommResponse(response);
    await this.setState('fuelLevel', data.level);
} catch (error) {
    if (error.code === 'ENOENT') {
        this.log.error('Serial port not found - check hardware connection');
        await this.setState('info.connection', false);
    } else if (error.message.includes('timeout')) {
        this.log.warn('Sensor response timeout - retrying...');
        // Implement retry logic
    } else {
        this.log.error(`Unexpected sensor error: ${error.message}`);
    }
}
```

**Resource Management for Serial Connections:**
```javascript
async unload(callback) {
  try {
    // Clear all polling timers
    if (this.pollTimer) {
      clearTimeout(this.pollTimer);
      this.pollTimer = undefined;
    }

    // Properly close serial port
    if (this.serialPort && this.serialPort.isOpen) {
      await new Promise((resolve) => {
        this.serialPort.close(resolve);
      });
    }

    this.serialPort = undefined;
    callback();
  } catch (e) {
    callback();
  }
}
```

---

## Admin UI Configuration

### JSON-Config Setup

Use JSON-Config format for modern ioBroker admin interfaces.

**Example Structure:**
```json
{
  "type": "panel",
  "items": {
    "host": {
      "type": "text",
      "label": "Host address",
      "help": "IP address or hostname of the device"
    }
  }
}
```

**Guidelines:**
- ✅ Use consistent naming conventions
- ✅ Provide sensible default values
- ✅ Include validation for required fields
- ✅ Add tooltips for complex options
- ✅ Ensure translations for all supported languages (minimum English and German)
- ✅ Write end-user friendly labels, avoid technical jargon

### JSON Config for Serial Parameters (Omnicomm LLS)

```json
{
  "type": "panel",
  "items": {
    "comPort": {
      "type": "text",
      "label": "Serial Port",
      "tooltip": "Path to serial port device (e.g., /dev/ttyUSB0, COM1)",
      "default": "/dev/ttyUSB0"
    },
    "baud": {
      "type": "select",
      "label": "Baud Rate",
      "options": [
        {"label": "9600", "value": 9600},
        {"label": "19200", "value": 19200},
        {"label": "38400", "value": 38400},
        {"label": "57600", "value": 57600},
        {"label": "115200", "value": 115200}
      ],
      "default": 9600
    },
    "address": {
      "type": "number",
      "label": "Sensor Address",
      "tooltip": "Modbus address of the Omnicomm LLS sensor",
      "min": 1,
      "max": 247,
      "default": 1
    },
    "pollTime": {
      "type": "number",
      "label": "Polling Interval (ms)",
      "tooltip": "Time between sensor readings in milliseconds",
      "min": 5000,
      "default": 10000
    }
  }
}
```

### Translation Management

**CRITICAL:** Translation files must stay synchronized with `admin/jsonConfig.json`. Orphaned keys or missing translations cause UI issues and PR review delays.

#### Overview
- **Location:** `admin/i18n/{lang}/translations.json` for 11 languages (de, en, es, fr, it, nl, pl, pt, ru, uk, zh-cn)
- **Source of truth:** `admin/jsonConfig.json` - all `label` and `help` properties must have translations
- **Command:** `npm run translate` - auto-generates translations but does NOT remove orphaned keys
- **Formatting:** English uses tabs, other languages use 4 spaces

#### Critical Rules
1. ✅ Keys must match exactly with jsonConfig.json
2. ✅ No orphaned keys in translation files
3. ✅ All translations must be in native language (no English fallbacks)
4. ✅ Keys must be sorted alphabetically

#### Workflow for Translation Updates

**When modifying admin/jsonConfig.json:**

1. Make your changes to labels/help texts
2. Run automatic translation: `npm run translate`
3. Run validation: `node scripts/validate-translations.js`
4. Remove orphaned keys manually from all translation files
5. Add missing translations in native languages
6. Run: `npm run lint && npm run test`

#### Translation Checklist

Before committing changes to admin UI or translations:
1. ✅ No orphaned keys in any translation file
2. ✅ All translations in native language
3. ✅ Keys alphabetically sorted
4. ✅ `npm run lint` passes
5. ✅ `npm run test` passes

---

## Documentation

### README Updates

#### Required Sections
1. **Installation** - Clear npm/ioBroker admin installation steps
2. **Configuration** - Detailed configuration options with examples
3. **Usage** - Practical examples and use cases
4. **Changelog** - Version history (use "## **WORK IN PROGRESS**" for ongoing changes)
5. **License** - License information (typically MIT for ioBroker adapters)
6. **Support** - Links to issues, discussions, community support

#### Documentation Standards
- Use clear, concise language
- Include code examples for configuration
- Add screenshots for admin interface when applicable
- Maintain multilingual support (minimum English and German)
- Always reference issues in commits and PRs (e.g., "fixes #xx")

#### Mandatory README Updates for PRs

For **every PR or new feature**, always add a user-friendly entry to README.md:

- Add entries under `## **WORK IN PROGRESS**` section
- Use format: `* (author) **TYPE**: Description of user-visible change`
- Types: **NEW** (features), **FIXED** (bugs), **ENHANCED** (improvements), **TESTING** (test additions), **CI/CD** (automation)
- Focus on user impact, not technical details

**Example:**
```markdown
## **WORK IN PROGRESS**

* (DutchmanNL) **FIXED**: Adapter now properly validates login credentials (fixes #25)
* (DutchmanNL) **NEW**: Added device discovery to simplify initial setup
```

### Changelog Management

Follow the [AlCalzone release-script](https://github.com/AlCalzone/release-script) standard.

#### Format Requirements

```markdown
# Changelog

<!--
  Placeholder for the next version (at the beginning of the line):
  ## **WORK IN PROGRESS**
-->

## **WORK IN PROGRESS**

- (author) **NEW**: Added new feature X
- (author) **FIXED**: Fixed bug Y (fixes #25)

## v0.1.0 (2023-01-01)
Initial release
```

#### Workflow Process
- **During Development:** All changes go under `## **WORK IN PROGRESS**`
- **For Every PR:** Add user-facing changes to WORK IN PROGRESS section
- **Before Merge:** Version number and date added when merging to main
- **Release Process:** Release-script automatically converts placeholder to actual version

#### Change Entry Format
- Format: `- (author) **TYPE**: User-friendly description`
- Types: **NEW**, **FIXED**, **ENHANCED**
- Focus on user impact, not technical implementation
- Reference issues: "fixes #XX" or "solves #XX"

---

## CI/CD & GitHub Actions

### Workflow Configuration

#### GitHub Actions Best Practices

**Must use ioBroker official testing actions:**
- `ioBroker/testing-action-check@v1` for lint and package validation
- `ioBroker/testing-action-adapter@v1` for adapter tests
- `ioBroker/testing-action-deploy@v1` for automated releases with Trusted Publishing (OIDC)

**Configuration:**
- **Node.js versions:** Test on 20.x, 22.x, 24.x
- **Platform:** Use ubuntu-22.04
- **Automated releases:** Deploy to npm on version tags (requires NPM Trusted Publishing)

#### Critical: Lint-First Validation Workflow

**ALWAYS run ESLint checks BEFORE other tests.** Benefits:
- Catches code quality issues immediately
- Prevents wasting CI resources on tests that would fail due to linting errors
- Provides faster feedback to developers

**Workflow Dependency Configuration:**
```yaml
jobs:
  check-and-lint:
    # Runs ESLint and package validation
    # Uses: ioBroker/testing-action-check@v1

  adapter-tests:
    needs: [check-and-lint]  # Wait for linting to pass

  integration-tests:
    needs: [check-and-lint, adapter-tests]  # Wait for both
```

### Testing Integration

#### GitHub Actions for Hardware Simulation

Since this adapter requires serial hardware, implement separate CI/CD jobs for hardware simulation:

```yaml
# Tests hardware simulation with mock serial data
hardware-simulation-tests:
  if: contains(github.event.head_commit.message, '[skip ci]') == false

  runs-on: ubuntu-22.04

  steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Use Node.js 20.x
      uses: actions/setup-node@v4
      with:
        node-version: 20.x
        cache: 'npm'

    - name: Install dependencies
      run: npm ci

    - name: Run hardware simulation tests
      run: npm run test:integration-hardware
```

#### CI/CD Best Practices for Serial Hardware
- Run hardware simulation tests separately from main test suite
- Use mock serial data for reproducible testing
- Don't make hardware tests required for deployment
- Provide clear failure messages for serial communication issues
- Use appropriate timeouts for serial operations (30+ seconds)
- Test with various sensor configurations and error scenarios

#### Package.json Integration
```json
{
  "scripts": {
    "test:integration-demo": "mocha test/integration-demo --exit",
    "test:integration-hardware": "mocha test/integration-hardware --exit --timeout 30000"
  }
}
```

#### Practical Example: Hardware Simulation Testing

```javascript
const path = require("path");
const { tests } = require("@iobroker/testing");

// Mock Omnicomm sensor responses
const MOCK_SENSOR_RESPONSES = {
    fuelLevel: Buffer.from([0x01, 0x03, 0x02, 0x13, 0x88, 0xB4, 0x32]), // 5000 mm fuel level
    temperature: Buffer.from([0x01, 0x03, 0x02, 0x00, 0x96, 0xB8, 0x7C]), // 150 = 15.0°C
    error: Buffer.from([0x01, 0x83, 0x02, 0xC0, 0xF1]) // Error response
};

tests.integration(path.join(__dirname, ".."), {
    defineAdditionalTests({ suite }) {
        suite("Hardware Simulation Testing", (getHarness) => {
            let harness;

            before(() => {
                harness = getHarness();
            });

            it("Should handle serial communication with mock sensor responses", async () => {
                console.log("Setting up mock serial communication...");

                if (harness.isAdapterRunning()) {
                    await harness.stopAdapter();
                }

                await harness.changeAdapterConfig("omnicomm-lls", {
                    native: {
                        comPort: "/dev/ttyUSB0",
                        baud: 9600,
                        address: 1,
                        pollTime: 5000,
                        testMode: true
                    }
                });

                console.log("Starting adapter with hardware simulation...");
                await harness.startAdapter();

                // Wait for serial initialization and first poll
                await new Promise(resolve => setTimeout(resolve, 15000));

                const fuelLevelState = await harness.states.getStateAsync("omnicomm-lls.0.fuelLevel");

                if (fuelLevelState && typeof fuelLevelState.val === 'number') {
                    console.log("✅ SUCCESS: Fuel level data received from simulated sensor");
                    return true;
                } else {
                    throw new Error("Hardware Simulation Test Failed: Expected fuel level data from mock sensor responses. " +
                        "Check serial communication mock implementation and data parsing logic.");
                }
            }).timeout(30000);
        });
    }
});
```

---

## Security Considerations for Serial Communication

### Safe Serial Port Handling
- Validate serial port paths to prevent directory traversal
- Implement rate limiting to prevent serial bus overload
- Handle unexpected data gracefully without crashes
- Sanitize sensor data before state updates

```javascript
// Example safe serial port validation
function validateSerialPort(portPath) {
    // Only allow standard serial port patterns
    const validPatterns = [
        /^\/dev\/ttyUSB\d+$/, // Linux USB serial
        /^\/dev\/ttyS\d+$/,   // Linux native serial
        /^COM\d+$/            // Windows COM ports
    ];

    return validPatterns.some(pattern => pattern.test(portPath));
}
```

---

## Development Patterns for Omnicomm LLS

### Logging Best Practices
- Use debug level for raw sensor data and communication details
- Use info level for successful sensor readings and connection status
- Use warn level for timeout or retry situations
- Use error level for hardware failures or configuration issues

```javascript
// Good logging examples for fuel sensors
this.log.debug(`Sending command to sensor ${address}: ${Buffer.from(command).toString('hex')}`);
this.log.info(`Fuel level reading from sensor ${address}: ${fuelLevel}L`);
this.log.warn(`Sensor ${address} response timeout, retrying in ${retryDelay}ms`);
this.log.error(`Failed to connect to serial port ${comPort}: ${error.message}`);
```

### Sensor Data Processing Pipeline
```javascript
async processOmnicommData(rawData) {
    try {
        // 1. Validate data integrity
        if (!this.validateCRC(rawData)) {
            throw new Error('Invalid CRC checksum');
        }

        // 2. Parse sensor-specific data format
        const parsedData = this.parseOmnicommFrame(rawData);

        // 3. Convert to meaningful units
        const fuelLevel = this.convertToFuelLevel(parsedData.rawValue);
        const temperature = this.convertToTemperature(parsedData.tempValue);

        // 4. Update states with proper error handling
        await this.setStateAsync('fuelLevel', fuelLevel, true);
        await this.setStateAsync('temperature', temperature, true);

        return { fuelLevel, temperature };
    } catch (error) {
        this.log.error(`Failed to process sensor data: ${error.message}`);
        throw error;
    }
}
```

### Multi-Sensor Management
```javascript
async pollAllSensors() {
    for (const sensorConfig of this.config.sensors) {
        try {
            await this.pollSensor(sensorConfig.address);
            // Add delay between sensors to prevent bus conflicts
            await new Promise(resolve => setTimeout(resolve, 100));
        } catch (error) {
            this.log.warn(`Failed to poll sensor ${sensorConfig.address}: ${error.message}`);
            // Continue with next sensor
        }
    }
}
```
