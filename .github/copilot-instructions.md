# ioBroker Adapter Development with GitHub Copilot

**Version:** 0.4.0
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

// Use tests.integration() with defineAdditionalTests
tests.integration(path.join(__dirname, '..'), {
    defineAdditionalTests({ suite }) {
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
    }
});
```

#### Additional Testing Patterns for Serial Communication

```javascript
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

### Performance Testing
- Test polling intervals don't overload the serial bus
- Verify memory usage doesn't increase over time with continuous polling
- Test multiple sensor handling on same serial connection
- Monitor adapter performance with different baud rates

## Error Handling and Resilience

### Serial Communication Error Patterns
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