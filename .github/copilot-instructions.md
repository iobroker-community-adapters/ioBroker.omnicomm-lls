# ioBroker Adapter Development with GitHub Copilot

**Version:** 0.4.2
**Template Source:** https://github.com/DrozmotiX/ioBroker-Copilot-Instructions

This file contains instructions and best practices for GitHub Copilot when working on ioBroker adapter development.

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

#### Key Integration Testing Rules

1. **NEVER test API URLs directly** - Let the adapter handle API calls
2. **ALWAYS use the harness** - `getHarness()` provides the testing environment  
3. **Configure via objects** - Use `harness.objects.setObject()` to set adapter configuration
4. **Start properly** - Use `harness.startAdapterAndWait()` to start the adapter
5. **Check states** - Use `harness.states.getState()` to verify results
6. **Use timeouts** - Allow time for async operations with appropriate timeouts
7. **Test real workflow** - Initialize → Configure → Start → Verify States

#### Workflow Dependencies
Integration tests should run ONLY after lint and adapter tests pass:

```yaml
integration-tests:
  needs: [check-and-lint, adapter-tests]
  runs-on: ubuntu-latest
  steps:
    - name: Run integration tests
      run: npx mocha test/integration-*.js --exit
```

#### What NOT to Do
❌ Direct API testing: `axios.get('https://api.example.com')`
❌ Mock adapters: `new MockAdapter()`  
❌ Direct internet calls in tests
❌ Bypassing the harness system

#### What TO Do
✅ Use `@iobroker/testing` framework
✅ Configure via `harness.objects.setObject()`
✅ Start via `harness.startAdapterAndWait()`
✅ Test complete adapter lifecycle
✅ Verify states via `harness.states.getState()`
✅ Allow proper timeouts for async operations

### API Testing with Credentials
For adapters that connect to external APIs requiring authentication, implement comprehensive credential testing:

#### Password Encryption for Integration Tests
When creating integration tests that need encrypted passwords (like those marked as `encryptedNative` in io-package.json):

1. **Read system secret**: Use `harness.objects.getObjectAsync("system.config")` to get `obj.native.secret`
2. **Apply XOR encryption**: Implement the encryption algorithm:
   ```javascript
   async function encryptPassword(harness, password) {
       const systemConfig = await harness.objects.getObjectAsync("system.config");
       if (!systemConfig || !systemConfig.native || !systemConfig.native.secret) {
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
3. **Store encrypted password**: Set the encrypted result in adapter config, not the plain text
4. **Result**: Adapter will properly decrypt and use credentials, enabling full API connectivity testing

#### Demo Credentials Testing Pattern
- Use provider demo credentials when available (e.g., `demo@api-provider.com` / `demo`)
- Create separate test file (e.g., `test/integration-demo.js`) for credential-based tests
- Add npm script: `"test:integration-demo": "mocha test/integration-demo --exit"`
- Implement clear success/failure criteria with recognizable log messages
- Expected success pattern: Look for specific adapter initialization messages
- Test should fail clearly with actionable error messages for debugging

#### Enhanced Test Failure Handling
```javascript
it("Should connect to API with demo credentials", async () => {
    // ... setup and encryption logic ...
    
    const connectionState = await harness.states.getStateAsync("adapter.0.info.connection");
    
    if (connectionState && connectionState.val === true) {
        console.log("✅ SUCCESS: API connection established");
        return true;
    } else {
        throw new Error("API Test Failed: Expected API connection to be established with demo credentials. " +
            "Check logs above for specific API errors (DNS resolution, 401 Unauthorized, network issues, etc.)");
    }
}).timeout(120000); // Extended timeout for API calls
```

### Performance Testing for Omnicomm LLS
- Test polling intervals don't overload the serial bus
- Verify memory usage doesn't increase over time with continuous polling
- Test multiple sensor handling on same serial connection
- Monitor adapter performance with different baud rates

## README Updates

### Required Sections
When updating README.md files, ensure these sections are present and well-documented:

1. **Installation** - Clear npm/ioBroker admin installation steps
2. **Configuration** - Detailed configuration options with examples
3. **Usage** - Practical examples and use cases
4. **Changelog** - Version history and changes (use "## **WORK IN PROGRESS**" section for ongoing changes following AlCalzone release-script standard)
5. **License** - License information (typically MIT for ioBroker adapters)
6. **Support** - Links to issues, discussions, and community support

### Documentation Standards
- Use clear, concise language
- Include code examples for configuration
- Add screenshots for admin interface when applicable
- Maintain multilingual support (at minimum English and German)
- When creating PRs, add entries to README under "## **WORK IN PROGRESS**" section following ioBroker release script standard
- Always reference related issues in commits and PR descriptions (e.g., "solves #xx" or "fixes #xx")

### Mandatory README Updates for PRs
For **every PR or new feature**, always add a user-friendly entry to README.md:

- Add entries under `## **WORK IN PROGRESS**` section before committing
- Use format: `* (author) **TYPE**: Description of user-visible change`
- Types: **NEW** (features), **FIXED** (bugs), **ENHANCED** (improvements), **TESTING** (test additions), **CI/CD** (automation)
- Focus on user impact, not technical implementation details
- Example: `* (DutchmanNL) **FIXED**: Adapter now properly validates login credentials instead of always showing "credentials missing"`

### Documentation Workflow Standards
- **Mandatory README updates**: Establish requirement to update README.md for every PR/feature
- **Standardized documentation**: Create consistent format and categories for changelog entries
- **Enhanced development workflow**: Integrate documentation requirements into standard development process

### Changelog Management with AlCalzone Release-Script
Follow the [AlCalzone release-script](https://github.com/AlCalzone/release-script) standard for changelog management:

#### Format Requirements
- Always use `## **WORK IN PROGRESS**` as the placeholder for new changes
- Add all PR/commit changes under this section until ready for release
- Never modify version numbers manually - only when merging to main branch
- Maintain this format in README.md or CHANGELOG.md:

```markdown
# Changelog

<!--
  Placeholder for the next version (at the beginning of the line):
  ## **WORK IN PROGRESS**
-->

## **WORK IN PROGRESS**

-   Did some changes
-   Did some more changes

## v0.1.0 (2023-01-01)
Initial release
```

#### Workflow Process
- **During Development**: All changes go under `## **WORK IN PROGRESS**`
- **For Every PR**: Add user-facing changes to the WORK IN PROGRESS section
- **Before Merge**: Version number and date are only added when merging to main
- **Release Process**: The release-script automatically converts the placeholder to the actual version

#### Change Entry Format
Use this consistent format for changelog entries:
- `- (author) **TYPE**: User-friendly description of the change`
- Types: **NEW** (features), **FIXED** (bugs), **ENHANCED** (improvements)
- Focus on user impact, not technical implementation details
- Reference related issues: "fixes #XX" or "solves #XX"

#### Example Entry
```markdown
## **WORK IN PROGRESS**

- (DutchmanNL) **FIXED**: Adapter now properly validates login credentials instead of always showing "credentials missing" (fixes #25)
- (DutchmanNL) **NEW**: Added support for device discovery to simplify initial setup
```

## Dependency Updates

### Package Management
- Always use `npm` for dependency management in ioBroker adapters
- When working on new features in a repository with an existing package-lock.json file, use `npm ci` to install dependencies. Use `npm install` only when adding or updating dependencies.
- Keep dependencies minimal and focused
- Only update dependencies to latest stable versions when necessary or in separate Pull Requests. Avoid updating dependencies when adding features that don't require these updates.
- When you modify `package.json`:
  1. Run `npm install` to update and sync `package-lock.json`.
  2. If `package-lock.json` was updated, commit both `package.json` and `package-lock.json`.

### Dependency Best Practices
- Prefer built-in Node.js modules when possible
- Use `@iobroker/adapter-core` for adapter base functionality
- Avoid deprecated packages
- Document any specific version requirements

## Error Handling and Resilience

### General Error Patterns
- Always catch and log errors appropriately
- Use adapter log levels (error, warn, info, debug)
- Provide meaningful, user-friendly error messages that help users understand what went wrong
- Handle network failures gracefully
- Implement retry mechanisms where appropriate
- Always clean up timers, intervals, and other resources in the `unload()` method

### Example Error Handling:
```javascript
try {
  await this.connectToDevice();
} catch (error) {
  this.log.error(`Failed to connect to device: ${error.message}`);
  this.setState('info.connection', false, true);
  // Implement retry logic if needed
}
```

### Timer and Resource Cleanup:
```javascript
// In your adapter class
private connectionTimer?: NodeJS.Timeout;

async onReady() {
  this.connectionTimer = setInterval(() => {
    this.checkConnection();
  }, 30000);
}

onUnload(callback) {
  try {
    // Clean up timers and intervals
    if (this.connectionTimer) {
      clearInterval(this.connectionTimer);
      this.connectionTimer = undefined;
    }
    // Close connections, clean up resources
    callback();
  } catch (e) {
    callback();
  }
}
```

### Serial Communication Error Patterns for Omnicomm LLS
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

### Resource Management for Serial Connections
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

## Logging Best Practices

### Omnicomm LLS Specific Logging
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

## Configuration Management

### JSON Config for Serial Parameters
Create proper JSON config structure for serial communication settings:

```javascript
// In jsonConfig.json
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

## JSON-Config Admin Instructions

### Configuration Schema
When creating admin configuration interfaces:

- Use JSON-Config format for modern ioBroker admin interfaces
- Provide clear labels and help text for all configuration options
- Include input validation and error messages
- Group related settings logically
- Example structure:
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

### Admin Interface Guidelines
- Use consistent naming conventions
- Provide sensible default values
- Include validation for required fields
- Add tooltips for complex configuration options
- Ensure translations are available for all supported languages (minimum English and German)
- Write end-user friendly labels and descriptions, avoiding technical jargon where possible

## Best Practices for Dependencies

### HTTP Client Libraries
- **Preferred:** Use native `fetch` API (Node.js 20+ required for adapters; built-in since Node.js 18)
- **Avoid:** `axios` unless specific features are required (reduces bundle size)

### Example with fetch:
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

### Other Dependency Recommendations
- **Logging:** Use adapter built-in logging (`this.log.*`)
- **Scheduling:** Use adapter built-in timers and intervals
- **File operations:** Use Node.js `fs/promises` for async file operations
- **Configuration:** Use adapter config system rather than external config libraries

## Code Style and Standards

- Follow JavaScript/TypeScript best practices
- Use async/await for asynchronous operations
- Implement proper resource cleanup in `unload()` method
- Use semantic versioning for adapter releases
- Include proper JSDoc comments for public methods
- Handle Buffer operations safely for binary serial data
- Implement proper error boundaries for hardware communication

## CI/CD and Testing Integration

### GitHub Actions for Hardware Testing
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

### CI/CD Best Practices for Serial Hardware
- Run hardware simulation tests separately from main test suite
- Use mock serial data for reproducible testing
- Don't make hardware tests required for deployment
- Provide clear failure messages for serial communication issues
- Use appropriate timeouts for serial operations (30+ seconds)
- Test with various sensor configurations and error scenarios

### Package.json Script Integration
Add dedicated script for hardware simulation testing:
```json
{
  "scripts": {
    "test:integration-hardware": "mocha test/integration-hardware --exit --timeout 30000"
  }
}
```

### Practical Example: Complete Hardware Simulation Testing Implementation

#### test/integration-hardware.js
```javascript
const path = require("path");
const { tests } = require("@iobroker/testing");

// Mock Omnicomm sensor responses
const MOCK_SENSOR_RESPONSES = {
    fuelLevel: Buffer.from([0x01, 0x03, 0x02, 0x13, 0x88, 0xB4, 0x32]), // 5000 mm fuel level
    temperature: Buffer.from([0x01, 0x03, 0x02, 0x00, 0x96, 0xB8, 0x7C]), // 150 = 15.0°C
    error: Buffer.from([0x01, 0x83, 0x02, 0xC0, 0xF1]) // Error response
};

// Run integration tests with hardware simulation
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
                        // Test mode flag for mock responses
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

## Development Patterns for Omnicomm LLS

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

This comprehensive guide should help GitHub Copilot provide optimal suggestions for developing and maintaining the ioBroker omnicomm-lls adapter, with special attention to serial communication patterns, hardware simulation testing, and fuel sensor-specific requirements.
