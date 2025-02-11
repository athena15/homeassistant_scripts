# Shelly Home Assistant Failover Script

A script for Shelly devices that automatically switches between "smart" and "dumb" modes based on Home Assistant connectivity. Perfect for setups where you want smart home functionality when Home Assistant is available, but need traditional switch functionality during outages.

## Primary Use Case

Originally developed for controlling smart bulbs (like Philips Hue) where:
- Shelly device controls power to smart bulbs
- Physical switch input is used as a trigger for Home Assistant automations
- Power to smart bulbs remains always on during normal operation
- Physical switch state changes trigger Home Assistant automations to control bulbs via their native protocol

When Home Assistant becomes unreachable, the script switches the Shelly to act as a traditional switch, allowing direct power control until Home Assistant connectivity is restored.

## Features

- Automatic detection of Home Assistant availability
- Smooth transition between smart and traditional switch modes
- Configurable check intervals and failure thresholds
- Detailed logging for troubleshooting
- Minimal network overhead
- Works with Shelly's stock firmware
- No external dependencies

## Compatible Devices

Tested on:
- Shelly Plus 2PM (firmware 1.4.4)

Should work with:
- Other Shelly Plus series devices
- Shelly Pro series
- Most Gen 3 Shelly devices

## Prerequisites

1. A Shelly device running stock firmware
2. Home Assistant instance accessible via HTTP
3. Long-lived access token from Home Assistant

## Setup Instructions

### 1. Create a Long-Lived Access Token in Home Assistant

1. Log into your Home Assistant instance
2. Click on your user profile (bottom left corner)
3. Scroll to the bottom to "Long-Lived Access Tokens"
4. Click "Create Token"
5. Give it a name (e.g., "Shelly_Failover")
6. Copy the token immediately (you won't be able to see it again)

### 2. Configure Your Shelly Device

1. Access your Shelly's web interface
2. Go to "Settings" → "Scripts"
3. Click "Add Script"
4. Name it (e.g., "HA_Failover")
5. Set Type to "JavaScript"
6. Enable "Auto start"
7. Paste the script code (see below)
8. Configure the variables in the CONFIG section:
   - Set your Home Assistant URL
   - Insert your access token
   - Set the correct relay index
9. Save the script

## Script Code

```javascript
// Shelly Home Assistant Failover Script
// Version 1.2.0

let CONFIG = {
    // How often to check HA connection (in seconds)
    CHECK_INTERVAL: 180,
    
    // How many failed checks before switching modes
    FAILURE_THRESHOLD: 2,
    
    // Your HA URL with protocol (use http:// or https://)
    HA_URL: "http://YOUR_HOME_ASSISTANT_IP:8123/api/",
    
    // Your long-lived access token
    HA_TOKEN: "YOUR_TOKEN_HERE",
    
    // Which relay to control (0 for single-relay devices, 0 or 1 for dual-relay)
    RELAY_INDEX: 0,
    
    // Enable detailed logging
    DEBUG: true
};

let failedChecks = 0;
let currentMode = "detached";  // Start in detached mode

function log(message) {
    if (CONFIG.DEBUG) {
        console.log(message);
    }
}

// Function to update switch configuration only if needed
function updateSwitchConfig(currentConfig) {
    // Only change config if the mode is different
    if (currentConfig.in_mode !== currentMode) {
        log("Changing switch mode to: " + currentMode);
        Shelly.call(
            "Switch.SetConfig",
            {
                id: CONFIG.RELAY_INDEX,
                config: {
                    in_mode: currentMode,
                    initial_state: currentMode === "detached" ? "on" : "match_input"
                }
            },
            function(result, error_code, error_message) {
                if (error_code !== 0) {
                    log("Error changing mode: " + error_message);
                } else {
                    log("Successfully changed to " + currentMode + " mode");
                }
            }
        );
    } else {
        log("Switch already in correct mode: " + currentMode);
    }
}

function checkConnection() {
    log("Checking HA connection...");
    Shelly.call(
        "HTTP.REQUEST",
        { 
            method: "GET",
            url: CONFIG.HA_URL,
            headers: {
                "Authorization": "Bearer " + CONFIG.HA_TOKEN,
                "Content-Type": "application/json"
            }
        },
        function(response, error_code, error_message) {
            if (error_code !== 0) {
                failedChecks++;
                log("Failed check #" + failedChecks + ". Error: " + error_message);
                
                if (failedChecks >= CONFIG.FAILURE_THRESHOLD) {
                    // Switch to follow mode
                    currentMode = "follow";
                    // Get current config before making changes
                    Shelly.call(
                        "Switch.GetConfig",
                        { id: CONFIG.RELAY_INDEX },
                        updateSwitchConfig
                    );
                }
            } else {
                log("Connection check successful");
                failedChecks = 0;
                // Switch to detached mode
                currentMode = "detached";
                // Get current config before making changes
                Shelly.call(
                    "Switch.GetConfig",
                    { id: CONFIG.RELAY_INDEX },
                    updateSwitchConfig
                );
            }
            log("Current mode: " + currentMode);
        }
    );
}

// Initialize the timer to check connection periodically
Timer.set(CONFIG.CHECK_INTERVAL * 1000, true, checkConnection);

// Perform initial check and setup
checkConnection();
log("Failover script started");
```

## Configuration Options

- `CHECK_INTERVAL`: How often to check HA connection (in seconds)
- `FAILURE_THRESHOLD`: Number of failed checks before switching modes
- `HA_URL`: Your Home Assistant URL
- `HA_TOKEN`: Your long-lived access token
- `RELAY_INDEX`: Which relay to control (0 or 1)
- `DEBUG`: Enable/disable detailed logging

## Edge Cases and Considerations

1. **Power Outages**
   - Script starts in detached mode
   - If HA isn't available at boot, switches to follow mode after failure threshold
   - Returns to detached mode when HA becomes available

2. **Network Issues**
   - Failure threshold prevents mode switching from temporary network glitches
   - Adjust CHECK_INTERVAL and FAILURE_THRESHOLD based on your network stability

3. **Initial State**
   - In detached mode: relay stays on to maintain power to smart bulbs
   - In follow mode: relay matches physical switch state

4. **Home Assistant Restarts**
   - Script will detect HA unavailability during restart
   - Will temporarily switch to follow mode
   - Automatically returns to detached mode when HA is back

## Troubleshooting

1. **Authentication Issues**
   - Verify token is copied correctly
   - Check HA URL is accessible
   - Ensure protocol (http/https) is correct

2. **Mode Switching Issues**
   - Check relay index is correct
   - Verify firmware version
   - Review console logs

3. **Frequent Mode Switches**
   - Increase FAILURE_THRESHOLD
   - Increase CHECK_INTERVAL
   - Check network stability

## Credits

- Original concept based on forum discussions
- Improved with robust error handling and configuration verification
- Tested and refined with community feedback

## License

MIT License - Feel free to use and modify as needed.
