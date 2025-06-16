# EthernetICMP Library - Arduino Network Diagnostics

## Overview
EthernetICMP provides ICMP ping functionality for Arduino Ethernet applications. This library enables network connectivity testing, latency measurement, and network diagnostics for embedded systems using Ethernet connectivity.

## Features
- **Synchronous Ping**: Blocking ping operations with immediate results
- **Asynchronous Ping**: Non-blocking ping operations for multitasking applications
- **DHCP Support**: Works with both static IP and DHCP configurations
- **Comprehensive Results**: Returns detailed ping statistics including TTL, response time, and status
- **Multiple Target Support**: Ping multiple hosts sequentially or in parallel
- **Error Handling**: Robust error detection and reporting

## Installation

### Arduino IDE
1. Download the library
2. Extract to your Arduino libraries folder
3. Restart Arduino IDE

```cpp
#include <SPI.h>
#include <Ethernet.h>
#include <EthernetICMP.h>
```

### PlatformIO
```ini
lib_deps = 
    EthernetICMP
    Ethernet
```

## Basic Usage

### Simple Ping Example
```cpp
#include <SPI.h>
#include <Ethernet.h>
#include <EthernetICMP.h>

byte mac[] = {0xDE, 0xAD, 0xBE, 0xEF, 0xFE, 0xED};
byte ip[] = {192, 168, 1, 177};
IPAddress pingAddr(8, 8, 8, 8); // Google DNS

SOCKET pingSocket = 0;
EthernetICMPPing ping(pingSocket, (uint16_t)random(0, 255));

void setup() {
    Serial.begin(9600);
    Ethernet.begin(mac, ip);
    delay(1000); // Allow Ethernet to initialize
}

void loop() {
    EthernetICMPEchoReply echoReply = ping(pingAddr, 4);
    
    if (echoReply.status == SUCCESS) {
        Serial.print("Ping successful: ");
        Serial.print(millis() - echoReply.data.time);
        Serial.println(" ms");
    } else {
        Serial.print("Ping failed with status: ");
        Serial.println(echoReply.status);
    }
    
    delay(1000);
}
```

### DHCP Configuration
```cpp
#include <SPI.h>
#include <Ethernet.h>
#include <EthernetICMP.h>

byte mac[] = {0xDE, 0xAD, 0xBE, 0xEF, 0xFE, 0xED};
IPAddress pingAddr(1, 1, 1, 1); // Cloudflare DNS

SOCKET pingSocket = 0;
EthernetICMPPing ping(pingSocket, (uint16_t)random(0, 255));

void setup() {
    Serial.begin(9600);
    
    // Start Ethernet with DHCP
    if (Ethernet.begin(mac) == 0) {
        Serial.println("Failed to configure Ethernet using DHCP");
        while (true) delay(1000);
    }
    
    Serial.print("IP address: ");
    Serial.println(Ethernet.localIP());
    
    delay(1000);
}

void loop() {
    EthernetICMPEchoReply echoReply = ping(pingAddr, 4);
    
    if (echoReply.status == SUCCESS) {
        Serial.print("Reply from ");
        Serial.print(echoReply.addr[0]); Serial.print(".");
        Serial.print(echoReply.addr[1]); Serial.print(".");
        Serial.print(echoReply.addr[2]); Serial.print(".");
        Serial.print(echoReply.addr[3]);
        Serial.print(": time=");
        Serial.print(millis() - echoReply.data.time);
        Serial.print("ms TTL=");
        Serial.println(echoReply.ttl);
    } else {
        Serial.println("Request timed out");
    }
    
    delay(2000);
}
```

## Advanced Usage

### Asynchronous Ping Operations
```cpp
#include <SPI.h>
#include <Ethernet.h>
#include <EthernetICMP.h>

byte mac[] = {0xDE, 0xAD, 0xBE, 0xEF, 0xFE, 0xED};
IPAddress pingAddr(8, 8, 8, 8);

SOCKET pingSocket = 0;
EthernetICMPPing ping(pingSocket, (uint16_t)random(0, 255));

bool pingInProgress = false;
unsigned long pingStartTime = 0;
const unsigned long PING_TIMEOUT = 5000; // 5 second timeout

void setup() {
    Serial.begin(9600);
    Ethernet.begin(mac);
    delay(1000);
}

void loop() {
    if (!pingInProgress) {
        // Start a new ping
        EthernetICMPEchoReply echoReply = ping.asyncStart(pingAddr, 4);
        if (echoReply.status == SUCCESS) {
            pingInProgress = true;
            pingStartTime = millis();
            Serial.println("Ping started...");
        } else {
            Serial.println("Failed to start ping");
        }
    } else {
        // Check for ping completion
        EthernetICMPEchoReply echoReply = ping.asyncComplete();
        
        if (echoReply.status == SUCCESS) {
            Serial.print("Async ping completed: ");
            Serial.print(millis() - echoReply.data.time);
            Serial.println(" ms");
            pingInProgress = false;
        } else if (millis() - pingStartTime > PING_TIMEOUT) {
            Serial.println("Ping timeout");
            pingInProgress = false;
        }
        
        // Do other work while waiting for ping
        performOtherTasks();
    }
    
    delay(100);
}

void performOtherTasks() {
    // Your other application logic here
    // This runs while waiting for ping responses
}
```

### Multiple Host Monitoring
```cpp
#include <SPI.h>
#include <Ethernet.h>
#include <EthernetICMP.h>

byte mac[] = {0xDE, 0xAD, 0xBE, 0xEF, 0xFE, 0xED};

// Multiple ping targets
IPAddress targets[] = {
    IPAddress(8, 8, 8, 8),      // Google DNS
    IPAddress(1, 1, 1, 1),      // Cloudflare DNS
    IPAddress(192, 168, 1, 1),  // Local gateway
    IPAddress(208, 67, 222, 222) // OpenDNS
};

String targetNames[] = {
    "Google DNS",
    "Cloudflare DNS", 
    "Local Gateway",
    "OpenDNS"
};

const int numTargets = sizeof(targets) / sizeof(targets[0]);

SOCKET pingSocket = 0;
EthernetICMPPing ping(pingSocket, (uint16_t)random(0, 255));

void setup() {
    Serial.begin(9600);
    Ethernet.begin(mac);
    delay(1000);
    
    Serial.println("Network Connectivity Monitor");
    Serial.println("============================");
}

void loop() {
    for (int i = 0; i < numTargets; i++) {
        Serial.print("Pinging ");
        Serial.print(targetNames[i]);
        Serial.print(" (");
        Serial.print(targets[i]);
        Serial.print(")... ");
        
        EthernetICMPEchoReply echoReply = ping(targets[i], 4);
        
        if (echoReply.status == SUCCESS) {
            Serial.print("OK (");
            Serial.print(millis() - echoReply.data.time);
            Serial.println(" ms)");
        } else {
            Serial.println("FAILED");
        }
        
        delay(500); // Brief delay between pings
    }
    
    Serial.println("---");
    delay(5000); // Wait 5 seconds before next round
}
```

### Network Quality Assessment
```cpp
#include <SPI.h>
#include <Ethernet.h>
#include <EthernetICMP.h>

byte mac[] = {0xDE, 0xAD, 0xBE, 0xEF, 0xFE, 0xED};
IPAddress testHost(8, 8, 8, 8);

SOCKET pingSocket = 0;
EthernetICMPPing ping(pingSocket, (uint16_t)random(0, 255));

struct NetworkStats {
    int totalPings;
    int successfulPings;
    long totalResponseTime;
    long minResponseTime;
    long maxResponseTime;
    float packetLoss;
    float avgResponseTime;
};

NetworkStats stats = {0, 0, 0, 999999, 0, 0.0, 0.0};

void setup() {
    Serial.begin(9600);
    Ethernet.begin(mac);
    delay(1000);
    
    Serial.println("Network Quality Assessment");
    Serial.println("=========================");
}

void loop() {
    // Perform 10 pings for assessment
    for (int i = 0; i < 10; i++) {
        stats.totalPings++;
        
        EthernetICMPEchoReply echoReply = ping(testHost, 4);
        
        if (echoReply.status == SUCCESS) {
            stats.successfulPings++;
            long responseTime = millis() - echoReply.data.time;
            stats.totalResponseTime += responseTime;
            
            if (responseTime < stats.minResponseTime) {
                stats.minResponseTime = responseTime;
            }
            if (responseTime > stats.maxResponseTime) {
                stats.maxResponseTime = responseTime;
            }
            
            Serial.print("Ping ");
            Serial.print(i + 1);
            Serial.print(": ");
            Serial.print(responseTime);
            Serial.println(" ms");
        } else {
            Serial.print("Ping ");
            Serial.print(i + 1);
            Serial.println(": FAILED");
        }
        
        delay(1000);
    }
    
    // Calculate statistics
    stats.packetLoss = ((float)(stats.totalPings - stats.successfulPings) / stats.totalPings) * 100.0;
    if (stats.successfulPings > 0) {
        stats.avgResponseTime = (float)stats.totalResponseTime / stats.successfulPings;
    }
    
    // Display results
    Serial.println("\n=== Network Quality Report ===");
    Serial.print("Packets sent: "); Serial.println(stats.totalPings);
    Serial.print("Packets received: "); Serial.println(stats.successfulPings);
    Serial.print("Packet loss: "); Serial.print(stats.packetLoss); Serial.println("%");
    
    if (stats.successfulPings > 0) {
        Serial.print("Min response time: "); Serial.print(stats.minResponseTime); Serial.println(" ms");
        Serial.print("Max response time: "); Serial.print(stats.maxResponseTime); Serial.println(" ms");
        Serial.print("Avg response time: "); Serial.print(stats.avgResponseTime); Serial.println(" ms");
        
        // Quality assessment
        if (stats.packetLoss == 0 && stats.avgResponseTime < 50) {
            Serial.println("Network Quality: EXCELLENT");
        } else if (stats.packetLoss < 5 && stats.avgResponseTime < 100) {
            Serial.println("Network Quality: GOOD");
        } else if (stats.packetLoss < 15 && stats.avgResponseTime < 300) {
            Serial.println("Network Quality: FAIR");
        } else {
            Serial.println("Network Quality: POOR");
        }
    }
    
    Serial.println("===============================\n");
    
    // Reset stats for next assessment
    stats = {0, 0, 0, 999999, 0, 0.0, 0.0};
    
    delay(10000); // Wait 10 seconds before next assessment
}

## API Reference

### EthernetICMPPing Class

#### Constructor
```cpp
EthernetICMPPing(SOCKET socket, uint16_t id);
```
- **socket**: Socket number to use for ICMP operations (0-3)
- **id**: Unique identifier for ping packets

#### Synchronous Methods

##### ping()
```cpp
EthernetICMPEchoReply ping(IPAddress addr, int nRetries);
```
Sends a ping packet and waits for response.
- **addr**: Target IP address
- **nRetries**: Number of retry attempts
- **Returns**: EthernetICMPEchoReply structure with results

#### Asynchronous Methods

##### asyncStart()
```cpp
EthernetICMPEchoReply asyncStart(IPAddress addr, int nRetries);
```
Initiates a ping without blocking.
- **addr**: Target IP address
- **nRetries**: Number of retry attempts
- **Returns**: Initial status (SUCCESS if started successfully)

##### asyncComplete()
```cpp
EthernetICMPEchoReply asyncComplete();
```
Checks for completion of asynchronous ping.
- **Returns**: EthernetICMPEchoReply with results if complete

### EthernetICMPEchoReply Structure

```cpp
struct EthernetICMPEchoReply {
    Status status;           // SUCCESS, SEND_TIMEOUT, NO_RESPONSE, BAD_RESPONSE
    byte addr[4];           // IP address that responded
    char ttl;               // Time To Live value
    ICMPEchoPacket data;    // Echo packet data
};
```

#### Status Values
- **SUCCESS**: Ping completed successfully
- **SEND_TIMEOUT**: Failed to send ping packet
- **NO_RESPONSE**: No response received within timeout
- **BAD_RESPONSE**: Received malformed response

### ICMPEchoPacket Structure

```cpp
struct ICMPEchoPacket {
    uint16_t seq;           // Sequence number
    unsigned long time;     // Timestamp when packet was sent
};
```

## Error Handling

### Connection Validation
```cpp
bool validateEthernetConnection() {
    // Check if Ethernet cable is connected
    if (Ethernet.linkStatus() == LinkOFF) {
        Serial.println("Ethernet cable is disconnected");
        return false;
    }

    // Check if IP address is assigned
    if (Ethernet.localIP() == IPAddress(0, 0, 0, 0)) {
        Serial.println("Failed to obtain IP address");
        return false;
    }

    return true;
}

void setup() {
    Serial.begin(9600);

    // Initialize Ethernet
    if (Ethernet.begin(mac) == 0) {
        Serial.println("Failed to configure Ethernet using DHCP");
        // Try with static IP as fallback
        Ethernet.begin(mac, IPAddress(192, 168, 1, 177));
    }

    if (!validateEthernetConnection()) {
        Serial.println("Ethernet connection validation failed");
        while (true) delay(1000);
    }

    Serial.println("Ethernet connection established");
}
```

### Comprehensive Error Checking
```cpp
void performRobustPing(IPAddress target, String targetName) {
    if (!validateEthernetConnection()) {
        Serial.println("Network connection lost");
        return;
    }

    Serial.print("Pinging ");
    Serial.print(targetName);
    Serial.print("... ");

    EthernetICMPEchoReply echoReply = ping(target, 3);

    switch (echoReply.status) {
        case SUCCESS:
            Serial.print("SUCCESS: ");
            Serial.print(millis() - echoReply.data.time);
            Serial.print(" ms, TTL=");
            Serial.println(echoReply.ttl);
            break;

        case SEND_TIMEOUT:
            Serial.println("FAILED: Unable to send ping packet");
            break;

        case NO_RESPONSE:
            Serial.println("FAILED: No response received (timeout)");
            break;

        case BAD_RESPONSE:
            Serial.println("FAILED: Received malformed response");
            break;

        default:
            Serial.print("FAILED: Unknown error (");
            Serial.print(echoReply.status);
            Serial.println(")");
            break;
    }
}
```

## Integration Examples

### With WiFi Fallback
```cpp
#include <SPI.h>
#include <Ethernet.h>
#include <WiFi.h>
#include <EthernetICMP.h>

byte mac[] = {0xDE, 0xAD, 0xBE, 0xEF, 0xFE, 0xED};
const char* ssid = "YourWiFiNetwork";
const char* password = "YourPassword";

bool useEthernet = true;
SOCKET pingSocket = 0;
EthernetICMPPing ping(pingSocket, (uint16_t)random(0, 255));

void setup() {
    Serial.begin(9600);

    // Try Ethernet first
    if (Ethernet.begin(mac) == 0) {
        Serial.println("Ethernet failed, switching to WiFi");
        useEthernet = false;

        WiFi.begin(ssid, password);
        while (WiFi.status() != WL_CONNECTED) {
            delay(1000);
            Serial.print(".");
        }
        Serial.println("WiFi connected");
    } else {
        Serial.println("Ethernet connected");
    }
}

void loop() {
    IPAddress target(8, 8, 8, 8);

    if (useEthernet) {
        // Use EthernetICMP for Ethernet connection
        EthernetICMPEchoReply reply = ping(target, 4);
        if (reply.status == SUCCESS) {
            Serial.print("Ethernet ping: ");
            Serial.print(millis() - reply.data.time);
            Serial.println(" ms");
        }
    } else {
        // Use WiFi ping alternative (implementation depends on WiFi library)
        Serial.println("Using WiFi connection for connectivity test");
    }

    delay(2000);
}
```

### With Data Logging
```cpp
#include <SPI.h>
#include <Ethernet.h>
#include <EthernetICMP.h>
#include <SD.h>

byte mac[] = {0xDE, 0xAD, 0xBE, 0xEF, 0xFE, 0xED};
IPAddress logTarget(8, 8, 8, 8);

SOCKET pingSocket = 0;
EthernetICMPPing ping(pingSocket, (uint16_t)random(0, 255));

void setup() {
    Serial.begin(9600);
    Ethernet.begin(mac);

    // Initialize SD card
    if (!SD.begin(4)) {
        Serial.println("SD card initialization failed");
        return;
    }

    // Create log file header
    File logFile = SD.open("ping_log.csv", FILE_WRITE);
    if (logFile) {
        logFile.println("Timestamp,Target,Status,ResponseTime,TTL");
        logFile.close();
    }
}

void loop() {
    EthernetICMPEchoReply reply = ping(logTarget, 4);

    // Log to SD card
    File logFile = SD.open("ping_log.csv", FILE_WRITE);
    if (logFile) {
        logFile.print(millis());
        logFile.print(",");
        logFile.print(logTarget);
        logFile.print(",");

        if (reply.status == SUCCESS) {
            logFile.print("SUCCESS");
            logFile.print(",");
            logFile.print(millis() - reply.data.time);
            logFile.print(",");
            logFile.println(reply.ttl);
        } else {
            logFile.print("FAILED");
            logFile.print(",");
            logFile.print("0");
            logFile.print(",");
            logFile.println("0");
        }

        logFile.close();
    }

    delay(5000);
}
```

## Performance Considerations

### Memory Usage
- Each ping operation uses approximately 64 bytes for packet data
- Asynchronous operations require additional buffer space
- Consider memory constraints when implementing multiple concurrent pings

### Timing Considerations
```cpp
// Avoid rapid successive pings
void rateLimitedPing(IPAddress target) {
    static unsigned long lastPingTime = 0;
    const unsigned long MIN_PING_INTERVAL = 1000; // 1 second minimum

    if (millis() - lastPingTime >= MIN_PING_INTERVAL) {
        EthernetICMPEchoReply reply = ping(target, 4);
        lastPingTime = millis();

        // Process reply...
    }
}
```

### Socket Management
```cpp
// Proper socket cleanup
class PingManager {
private:
    SOCKET pingSocket;
    EthernetICMPPing* pinger;

public:
    PingManager(SOCKET socket) : pingSocket(socket) {
        pinger = new EthernetICMPPing(pingSocket, random(0, 255));
    }

    ~PingManager() {
        delete pinger;
        // Socket cleanup handled by Ethernet library
    }

    EthernetICMPEchoReply ping(IPAddress addr) {
        return pinger->operator()(addr, 4);
    }
};
```

## Troubleshooting

### Common Issues

1. **No Response from Local Network**
   - Check Ethernet cable connection
   - Verify IP configuration matches network
   - Ensure target device responds to ping

2. **Intermittent Failures**
   - Network congestion may cause timeouts
   - Increase retry count for unreliable networks
   - Implement exponential backoff for failed pings

3. **Memory Issues**
   - Reduce concurrent ping operations
   - Use static allocation instead of dynamic
   - Monitor available RAM

### Debug Helpers
```cpp
void printNetworkInfo() {
    Serial.println("=== Network Information ===");
    Serial.print("Local IP: "); Serial.println(Ethernet.localIP());
    Serial.print("Subnet Mask: "); Serial.println(Ethernet.subnetMask());
    Serial.print("Gateway: "); Serial.println(Ethernet.gatewayIP());
    Serial.print("DNS Server: "); Serial.println(Ethernet.dnsServerIP());
    Serial.print("Link Status: ");
    Serial.println(Ethernet.linkStatus() == LinkON ? "Connected" : "Disconnected");
    Serial.println("===========================");
}
```

## License
This library is released under the MIT License. See the original source files for complete license information.

## Credits
- Original ICMP implementation by Bjoern Hartmann
- Enhanced by Blake Foster
- Additional contributions by various community members

## References
- [Arduino Ethernet Library Documentation](https://www.arduino.cc/en/Reference/Ethernet)
- [ICMP Protocol Specification (RFC 792)](https://tools.ietf.org/html/rfc792)
- [Wiznet W5100 Datasheet](http://wizwiki.net/wiki/doku.php?id=products:w5100:start)
```
