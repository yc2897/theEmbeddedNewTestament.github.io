# 🎯 IoT & Wireless Interview Preparation

## 🚀 **Quick Navigation**
- [Common Questions](#common-questions)
- [Problem-Solving Examples](#problem-solving-examples)
- [Practice Problems](#practice-problems)
- [Resources](#resources)

## 📚 **Quick Reference: Key Concepts**
- **Wireless Protocols**: Bluetooth, WiFi, Zigbee, LoRa, cellular
- **IoT Communication**: MQTT, CoAP, HTTP, WebSocket
- **Power Management**: Low-power modes, duty cycling, energy harvesting
- **Security**: Authentication, encryption, secure boot, OTA updates
- **Network Topologies**: Star, mesh, tree, hybrid networks

## 🎯 **Common Interview Questions**

### **Question 1: Design a low-power IoT sensor node with wireless communication**

**Why this matters**: IoT devices often operate on battery power and require careful power management and wireless communication design.

**Problem**: Design an IoT sensor node that can operate for 1 year on a single battery charge.

**Requirements**:
- Temperature and humidity sensing
- Wireless data transmission
- 1-year battery life
- Remote configuration capability
- Secure communication

**Solution Design**:
```c
#include <stdint.h>
#include <stdbool.h>
#include <string.h>
#include "nrf52_hal.h"
#include "ble_stack.h"
#include "sensor_driver.h"
#include "power_manager.h"

// Sensor node configuration
typedef struct {
    uint16_t device_id;
    uint32_t sampling_interval;    // in seconds
    uint32_t transmission_interval; // in seconds
    int16_t temp_offset;           // temperature calibration
    uint16_t humidity_offset;      // humidity calibration
    uint8_t encryption_key[32];    // AES encryption key
    bool low_power_mode;
} node_config_t;

// Sensor data structure
typedef struct {
    uint32_t timestamp;
    int16_t temperature;           // 0.1°C resolution
    uint16_t humidity;             // 0.1% resolution
    uint16_t battery_voltage;      // mV
    uint8_t signal_strength;       // RSSI
    uint16_t sequence_number;
} sensor_data_t;

// Power management states
typedef enum {
    POWER_STATE_ACTIVE,        // Full power, sensors active
    POWER_STATE_SENSING,       // Sensors on, radio off
    POWER_STATE_TRANSMITTING,  // Radio on, sensors off
    POWER_STATE_SLEEP,         // Deep sleep, wake on timer
    POWER_STATE_HIBERNATE      // Ultra-low power, wake on interrupt
} power_state_t;

// Power manager
typedef struct {
    power_state_t current_state;
    uint32_t sleep_duration;
    uint32_t last_wake_time;
    uint16_t battery_voltage;
    uint8_t battery_level;
    bool low_battery;
} power_manager_t;

static power_manager_t power_mgr;
static node_config_t node_config;
static sensor_data_t sensor_data;

// Initialize power management
void power_manager_init(void) {
    power_mgr.current_state = POWER_STATE_ACTIVE;
    power_mgr.sleep_duration = 0;
    power_mgr.last_wake_time = 0;
    
    // Configure low-power modes
    nrf_power_mode_set(NRF_POWER_MODE_LOWPWR);
    
    // Enable DC/DC converter for better efficiency
    nrf_power_dcdcen_set(true);
    
    // Configure low-frequency clock source
    nrf_clock_lf_src_set(NRF_CLOCK_LF_SRC_XTAL);
    
    // Set up power failure detection
    nrf_power_pofcon_set(true, 2700); // 2.7V threshold
}

// Enter low-power mode
void enter_low_power_mode(power_state_t state) {
    switch (state) {
        case POWER_STATE_SLEEP:
            // Light sleep: CPU stopped, peripherals active
            __WFI(); // Wait for interrupt
            break;
            
        case POWER_STATE_HIBERNATE:
            // Deep sleep: Most peripherals stopped
            // Configure wake-up sources
            nrf_gpio_cfg_sense_set(WAKE_PIN, NRF_GPIO_PIN_SENSE_LOW);
            nrf_gpio_cfg_input(WAKE_PIN, NRF_GPIO_PIN_PULLUP);
            
            // Enter system OFF mode
            nrf_power_system_off();
            break;
            
        default:
            break;
    }
    
    power_mgr.current_state = state;
}

// Sensor reading with power optimization
bool read_sensors_power_optimized(void) {
    // Wake up sensors
    sensor_wake_up();
    
    // Read temperature
    if (!sensor_read_temperature(&sensor_data.temperature)) {
        return false;
    }
    
    // Apply calibration
    sensor_data.temperature += node_config.temp_offset;
    
    // Read humidity
    if (!sensor_read_humidity(&sensor_data.humidity)) {
        return false;
    }
    
    // Apply calibration
    sensor_data.humidity += node_config.humidity_offset;
    
    // Read battery voltage
    sensor_data.battery_voltage = read_battery_voltage();
    power_mgr.battery_voltage = sensor_data.battery_voltage;
    
    // Check battery level
    if (sensor_data.battery_voltage < 3000) { // 3.0V
        power_mgr.low_battery = true;
        // Reduce transmission frequency
        node_config.transmission_interval *= 2;
    }
    
    // Put sensors back to sleep
    sensor_sleep();
    
    sensor_data.timestamp = get_system_time();
    sensor_data.sequence_number++;
    
    return true;
}

// BLE advertisement for data transmission
void start_ble_advertisement(void) {
    // Configure advertisement data
    ble_gap_adv_data_t adv_data = {0};
    
    // Include sensor data in advertisement
    uint8_t adv_payload[31];
    uint8_t payload_len = 0;
    
    // Device ID
    adv_payload[payload_len++] = 0x16; // 16-bit UUID
    adv_payload[payload_len++] = 0x12; // Company ID
    adv_payload[payload_len++] = 0x34;
    adv_payload[payload_len++] = (node_config.device_id >> 8) & 0xFF;
    adv_payload[payload_len++] = node_config.device_id & 0xFF;
    
    // Temperature (2 bytes)
    adv_payload[payload_len++] = (sensor_data.temperature >> 8) & 0xFF;
    adv_payload[payload_len++] = sensor_data.temperature & 0xFF;
    
    // Humidity (2 bytes)
    adv_payload[payload_len++] = (sensor_data.humidity >> 8) & 0xFF;
    adv_payload[payload_len++] = sensor_data.humidity & 0xFF;
    
    // Battery voltage (2 bytes)
    adv_payload[payload_len++] = (sensor_data.battery_voltage >> 8) & 0xFF;
    adv_payload[payload_len++] = sensor_data.battery_voltage & 0xFF;
    
    // Sequence number (2 bytes)
    adv_payload[payload_len++] = (sensor_data.sequence_number >> 8) & 0xFF;
    adv_payload[payload_len++] = sensor_data.sequence_number & 0xFF;
    
    // CRC for data integrity
    uint16_t crc = calculate_crc16(adv_payload, payload_len);
    adv_payload[payload_len++] = (crc >> 8) & 0xFF;
    adv_payload[payload_len++] = crc & 0xFF;
    
    adv_data.adv_data.p_data = adv_payload;
    adv_data.adv_data.len = payload_len;
    
    // Start advertisement
    ble_gap_adv_start(BLE_GAP_ADV_TYPE_ADV_IND, BLE_GAP_ADV_FAST_INTERVAL1_MIN,
                      BLE_GAP_ADV_FAST_INTERVAL1_MAX, &adv_data, NULL);
}

// BLE connection handling
void on_ble_connected(ble_gap_evt_connected_t *p_evt) {
    // Stop advertisement
    ble_gap_adv_stop();
    
    // Enable notifications for configuration updates
    enable_configuration_service();
    
    // Send current sensor data
    send_sensor_data_notification();
}

// BLE disconnection handling
void on_ble_disconnected(ble_gap_evt_disconnected_t *p_evt) {
    // Resume advertisement
    start_ble_advertisement();
}

// Configuration service for remote updates
void handle_configuration_write(uint8_t *data, uint16_t length) {
    if (length < sizeof(node_config_t)) {
        return; // Invalid data length
    }
    
    // Verify data integrity
    uint16_t received_crc = *(uint16_t*)(data + length - 2);
    uint16_t calculated_crc = calculate_crc16(data, length - 2);
    
    if (received_crc != calculated_crc) {
        return; // CRC mismatch
    }
    
    // Update configuration
    memcpy(&node_config, data, sizeof(node_config_t));
    
    // Save to flash memory
    flash_write_config(&node_config);
    
    // Apply new settings
    if (node_config.low_power_mode) {
        // Increase sleep duration
        power_mgr.sleep_duration = 300; // 5 minutes
    } else {
        power_mgr.sleep_duration = 60;  // 1 minute
    }
}

// Main application loop
void main_loop(void) {
    static uint32_t last_sensor_read = 0;
    static uint32_t last_transmission = 0;
    uint32_t current_time = get_system_time();
    
    // Check if it's time to read sensors
    if (current_time - last_sensor_read >= node_config.sampling_interval) {
        if (read_sensors_power_optimized()) {
            last_sensor_read = current_time;
        }
    }
    
    // Check if it's time to transmit data
    if (current_time - last_transmission >= node_config.transmission_interval) {
        start_ble_advertisement();
        last_transmission = current_time;
        
        // Enter transmission mode
        power_mgr.current_state = POWER_STATE_TRANSMITTING;
    }
    
    // Check if we should go to sleep
    if (power_mgr.current_state == POWER_STATE_SLEEP ||
        (current_time - last_transmission > 30)) { // 30s timeout
        
        // Calculate sleep duration
        uint32_t next_event = MIN(
            last_sensor_read + node_config.sampling_interval,
            last_transmission + node_config.transmission_interval
        );
        
        uint32_t sleep_time = next_event - current_time;
        
        if (sleep_time > 0) {
            // Configure sleep timer
            nrf_timer_cc_set(NRF_TIMER0, NRF_TIMER_CC_CHANNEL0, sleep_time * 32768);
            nrf_timer_event_enable(NRF_TIMER0, NRF_TIMER_EVENT_COMPARE0);
            
            // Enter sleep mode
            enter_low_power_mode(POWER_STATE_SLEEP);
        }
    }
}

// Power consumption optimization
void optimize_power_consumption(void) {
    // Disable unused peripherals
    nrf_uart_disable(NRF_UART0);
    nrf_spi_disable(NRF_SPI0);
    
    // Reduce clock frequency when possible
    nrf_clock_hf_src_set(NRF_CLOCK_HF_SRC_XTAL);
    
    // Use internal RC oscillator for low-power operations
    nrf_clock_lf_src_set(NRF_CLOCK_LF_SRC_RC);
    
    // Configure GPIO for low power
    for (int i = 0; i < 32; i++) {
        nrf_gpio_cfg_default(i);
    }
    
    // Enable only necessary GPIO pins
    nrf_gpio_cfg_output(LED_PIN);
    nrf_gpio_cfg_input(SENSOR_POWER_PIN);
}

// Battery monitoring
void monitor_battery(void) {
    uint16_t voltage = read_battery_voltage();
    
    if (voltage < 2800) { // 2.8V - critical
        // Enter ultra-low power mode
        power_mgr.current_state = POWER_STATE_HIBERNATE;
        
        // Disable all non-essential functions
        disable_wireless_communication();
        
        // Only wake up for critical sensor readings
        node_config.sampling_interval = 3600; // 1 hour
        
    } else if (voltage < 3000) { // 3.0V - low
        power_mgr.low_battery = true;
        
        // Reduce transmission frequency
        node_config.transmission_interval *= 4;
        
        // Disable LED indicators
        disable_led_indicators();
        
    } else {
        power_mgr.low_battery = false;
        
        // Restore normal operation
        node_config.transmission_interval = 300; // 5 minutes
        enable_led_indicators();
    }
}
```

**IoT Node Features**:
- **Power Management**: Multiple sleep modes and power optimization
- **Wireless Communication**: BLE advertisement for data transmission
- **Remote Configuration**: BLE service for configuration updates
- **Battery Monitoring**: Adaptive power management based on battery level

**Follow-up Questions**:
- How would you handle network congestion?
- What if you need to support multiple wireless protocols?

### **Question 2: Implement a secure MQTT client for industrial IoT**

**Problem**: Create a secure MQTT client that can handle industrial IoT requirements.

**Requirements**:
- TLS/SSL encryption
- Certificate-based authentication
- QoS levels and message persistence
- Automatic reconnection
- Message queuing

**Solution Design**:
```c
#include <stdint.h>
#include <stdbool.h>
#include <string.h>
#include <stdlib.h>
#include "mqtt_client.h"
#include "tls_client.h"
#include "certificate_manager.h"
#include "message_queue.h"

// MQTT client configuration
typedef struct {
    char broker_host[128];
    uint16_t broker_port;
    char client_id[64];
    char username[64];
    char password[128];
    char ca_cert_path[256];
    char client_cert_path[256];
    char client_key_path[256];
    uint8_t keepalive_interval;
    bool clean_session;
    uint8_t max_qos;
    uint16_t max_inflight;
} mqtt_config_t;

// MQTT message structure
typedef struct {
    char topic[256];
    uint8_t *payload;
    uint16_t payload_len;
    uint8_t qos;
    bool retain;
    uint16_t message_id;
    uint32_t timestamp;
} mqtt_message_t;

// MQTT client state
typedef struct {
    mqtt_config_t config;
    tls_client_t *tls_client;
    bool connected;
    uint16_t message_id_counter;
    message_queue_t *outgoing_queue;
    message_queue_t *incoming_queue;
    uint32_t last_ping;
    uint32_t reconnect_attempts;
    uint32_t last_reconnect;
} mqtt_client_t;

// Initialize MQTT client
mqtt_client_t* mqtt_client_init(const mqtt_config_t *config) {
    mqtt_client_t *client = malloc(sizeof(mqtt_client_t));
    if (!client) return NULL;
    
    // Copy configuration
    memcpy(&client->config, config, sizeof(mqtt_config_t));
    
    // Initialize TLS client
    tls_config_t tls_config = {
        .ca_cert_path = config->ca_cert_path,
        .client_cert_path = config->client_cert_path,
        .client_key_path = config->client_key_path,
        .verify_peer = true,
        .verify_hostname = true
    };
    
    client->tls_client = tls_client_init(&tls_config);
    if (!client->tls_client) {
        free(client);
        return NULL;
    }
    
    // Initialize message queues
    client->outgoing_queue = message_queue_create(100);
    client->incoming_queue = message_queue_create(100);
    
    if (!client->outgoing_queue || !client->incoming_queue) {
        tls_client_cleanup(client->tls_client);
        free(client);
        return NULL;
    }
    
    // Initialize client state
    client->connected = false;
    client->message_id_counter = 1;
    client->last_ping = 0;
    client->reconnect_attempts = 0;
    client->last_reconnect = 0;
    
    return client;
}

// Connect to MQTT broker
bool mqtt_connect(mqtt_client_t *client) {
    if (!client || !client->tls_client) return false;
    
    // Establish TLS connection
    if (!tls_client_connect(client->tls_client, 
                           client->config.broker_host, 
                           client->config.broker_port)) {
        return false;
    }
    
    // Send CONNECT packet
    mqtt_connect_packet_t connect_pkt = {
        .client_id = client->config.client_id,
        .username = client->config.username,
        .password = client->config.password,
        .keepalive = client->config.keepalive_interval,
        .clean_session = client->config.clean_session,
        .will_topic = NULL,
        .will_message = NULL,
        .will_qos = 0,
        .will_retain = false
    };
    
    if (!send_mqtt_connect(client->tls_client, &connect_pkt)) {
        tls_client_disconnect(client->tls_client);
        return false;
    }
    
    // Wait for CONNACK
    mqtt_connack_packet_t connack;
    if (!receive_mqtt_connack(client->tls_client, &connack)) {
        tls_client_disconnect(client->tls_client);
        return false;
    }
    
    if (connack.return_code != MQTT_CONNACK_ACCEPTED) {
        tls_client_disconnect(client->tls_client);
        return false;
    }
    
    client->connected = true;
    client->reconnect_attempts = 0;
    
    return true;
}

// Publish message with QoS handling
bool mqtt_publish(mqtt_client_t *client, const char *topic, 
                  const uint8_t *payload, uint16_t payload_len,
                  uint8_t qos, bool retain) {
    if (!client || !client->connected) return false;
    
    // Create message
    mqtt_message_t *message = malloc(sizeof(mqtt_message_t));
    if (!message) return false;
    
    strncpy(message->topic, topic, sizeof(message->topic) - 1);
    message->payload = malloc(payload_len);
    if (!message->payload) {
        free(message);
        return false;
    }
    
    memcpy(message->payload, payload, payload_len);
    message->payload_len = payload_len;
    message->qos = qos;
    message->retain = retain;
    message->timestamp = get_system_time();
    
    if (qos > 0) {
        message->message_id = client->message_id_counter++;
    } else {
        message->message_id = 0;
    }
    
    // Send PUBLISH packet
    mqtt_publish_packet_t publish_pkt = {
        .topic = message->topic,
        .payload = message->payload,
        .payload_len = message->payload_len,
        .qos = message->qos,
        .retain = message->retain,
        .message_id = message->message_id
    };
    
    if (!send_mqtt_publish(client->tls_client, &publish_pkt)) {
        free(message->payload);
        free(message);
        return false;
    }
    
    // Handle QoS > 0
    if (qos == 1) {
        // Wait for PUBACK
        if (!wait_for_puback(client->tls_client, message->message_id)) {
            // Add to outgoing queue for retry
            message_queue_enqueue(client->outgoing_queue, message);
        } else {
            free(message->payload);
            free(message);
        }
        
    } else if (qos == 2) {
        // Wait for PUBREC
        if (!wait_for_pubrec(client->tls_client, message->message_id)) {
            // Add to outgoing queue for retry
            message_queue_enqueue(client->outgoing_queue, message);
        } else {
            // Send PUBREL
            if (!send_mqtt_pubrel(client->tls_client, message->message_id)) {
                message_queue_enqueue(client->outgoing_queue, message);
            } else {
                // Wait for PUBCOMP
                if (!wait_for_pubcomp(client->tls_client, message->message_id)) {
                    message_queue_enqueue(client->outgoing_queue, message);
                } else {
                    free(message->payload);
                    free(message);
                }
            }
        }
    } else {
        // QoS 0 - no acknowledgment needed
        free(message->payload);
        free(message);
    }
    
    return true;
}

// Subscribe to topic
bool mqtt_subscribe(mqtt_client_t *client, const char *topic, uint8_t qos) {
    if (!client || !client->connected) return false;
    
    // Send SUBSCRIBE packet
    mqtt_subscribe_packet_t subscribe_pkt = {
        .message_id = client->message_id_counter++,
        .topic = topic,
        .qos = qos
    };
    
    if (!send_mqtt_subscribe(client->tls_client, &subscribe_pkt)) {
        return false;
    }
    
    // Wait for SUBACK
    mqtt_suback_packet_t suback;
    if (!receive_mqtt_suback(client->tls_client, &suback)) {
        return false;
    }
    
    if (suback.return_code == 0x80) { // Failure
        return false;
    }
    
    return true;
}

// Handle incoming messages
void mqtt_handle_incoming(mqtt_client_t *client) {
    if (!client || !client->connected) return;
    
    // Check for incoming data
    uint8_t packet_type = peek_mqtt_packet_type(client->tls_client);
    
    switch (packet_type) {
        case MQTT_PACKET_TYPE_PUBLISH:
            handle_incoming_publish(client);
            break;
            
        case MQTT_PACKET_TYPE_PUBACK:
            handle_puback(client);
            break;
            
        case MQTT_PACKET_TYPE_PUBREC:
            handle_pubrec(client);
            break;
            
        case MQTT_PACKET_TYPE_PUBCOMP:
            handle_pubcomp(client);
            break;
            
        case MQTT_PACKET_TYPE_PINGRESP:
            handle_pingresp(client);
            break;
            
        default:
            break;
    }
}

// Handle incoming PUBLISH
void handle_incoming_publish(mqtt_client_t *client) {
    mqtt_publish_packet_t publish;
    if (!receive_mqtt_publish(client->tls_client, &publish)) {
        return;
    }
    
    // Create message for application
    mqtt_message_t *message = malloc(sizeof(mqtt_message_t));
    if (!message) return;
    
    strncpy(message->topic, publish.topic, sizeof(message->topic) - 1);
    message->payload = malloc(publish.payload_len);
    if (!message->payload) {
        free(message);
        return;
    }
    
    memcpy(message->payload, publish.payload, publish.payload_len);
    message->payload_len = publish.payload_len;
    message->qos = publish.qos;
    message->retain = publish.retain;
    message->message_id = publish.message_id;
    message->timestamp = get_system_time();
    
    // Add to incoming queue
    message_queue_enqueue(client->incoming_queue, message);
    
    // Send acknowledgment if QoS > 0
    if (publish.qos == 1) {
        send_mqtt_puback(client->tls_client, publish.message_id);
    } else if (publish.qos == 2) {
        send_mqtt_pubrec(client->tls_client, publish.message_id);
    }
}

// Keep-alive and connection monitoring
void mqtt_keepalive(mqtt_client_t *client) {
    if (!client || !client->connected) return;
    
    uint32_t current_time = get_system_time();
    
    // Send PINGREQ if keepalive interval has passed
    if (current_time - client->last_ping >= client->config.keepalive_interval * 1000) {
        if (!send_mqtt_pingreq(client->tls_client)) {
            // Connection lost, attempt reconnection
            handle_connection_loss(client);
            return;
        }
        
        client->last_ping = current_time;
    }
    
    // Check for PINGRESP timeout
    if (current_time - client->last_ping > 5000) { // 5 second timeout
        handle_connection_loss(client);
    }
}

// Handle connection loss
void handle_connection_loss(mqtt_client_t *client) {
    client->connected = false;
    
    // Clean up TLS connection
    tls_client_disconnect(client->tls_client);
    
    // Attempt reconnection
    if (client->reconnect_attempts < 5) {
        uint32_t current_time = get_system_time();
        
        // Exponential backoff
        uint32_t backoff_time = (1 << client->reconnect_attempts) * 1000;
        
        if (current_time - client->last_reconnect >= backoff_time) {
            if (mqtt_connect(client)) {
                // Reconnection successful
                client->reconnect_attempts = 0;
                
                // Resubscribe to topics
                resubscribe_topics(client);
                
                // Resend queued messages
                resend_queued_messages(client);
            } else {
                client->reconnect_attempts++;
                client->last_reconnect = current_time;
            }
        }
    }
}

// Resend queued messages after reconnection
void resend_queued_messages(mqtt_client_t *client) {
    if (!client || !client->outgoing_queue) return;
    
    mqtt_message_t *message;
    
    while ((message = message_queue_dequeue(client->outgoing_queue)) != NULL) {
        // Check if message is too old (older than 1 hour)
        if (get_system_time() - message->timestamp > 3600000) {
            // Discard old message
            free(message->payload);
            free(message);
            continue;
        }
        
        // Resend message
        if (!mqtt_publish(client, message->topic, message->payload, 
                         message->payload_len, message->qos, message->retain)) {
            // Put back in queue for next retry
            message_queue_enqueue(client->outgoing_queue, message);
            break;
        }
        
        // Message sent successfully
        free(message->payload);
        free(message);
    }
}

// Clean up MQTT client
void mqtt_client_cleanup(mqtt_client_t *client) {
    if (!client) return;
    
    // Disconnect from broker
    if (client->connected) {
        send_mqtt_disconnect(client->tls_client);
        tls_client_disconnect(client->tls_client);
    }
    
    // Clean up TLS client
    tls_client_cleanup(client->tls_client);
    
    // Clean up message queues
    if (client->outgoing_queue) {
        message_queue_cleanup(client->outgoing_queue);
    }
    
    if (client->incoming_queue) {
        message_queue_cleanup(client->incoming_queue);
    }
    
    // Free client
    free(client);
}
```

**MQTT Client Features**:
- **TLS Security**: Certificate-based authentication and encryption
- **QoS Handling**: Support for all QoS levels with proper acknowledgment
- **Message Queuing**: Persistent message storage for reliable delivery
- **Automatic Reconnection**: Exponential backoff with message replay

### **Question 3: Design a mesh network for industrial IoT**

**Problem**: Create a mesh network protocol for industrial IoT applications.

**Requirements**:
- Self-healing network topology
- Multi-hop routing
- Load balancing
- Network diagnostics
- Security features

**Solution Design**:
```c
#include <stdint.h>
#include <stdbool.h>
#include <string.h>
#include <stdlib.h>
#include "mesh_protocol.h"
#include "routing_table.h"
#include "network_topology.h"
#include "security_manager.h"

// Mesh node types
typedef enum {
    NODE_TYPE_END_DEVICE,    // Leaf node, no routing
    NODE_TYPE_ROUTER,        // Can route messages
    NODE_TYPE_COORDINATOR    // Network coordinator
} node_type_t;

// Mesh node information
typedef struct {
    uint16_t node_id;
    node_type_t type;
    uint8_t capabilities;
    uint8_t link_quality;
    uint16_t parent_id;
    uint8_t hop_count;
    uint32_t last_seen;
    bool active;
} mesh_node_t;

// Mesh message structure
typedef struct {
    uint16_t source_id;
    uint16_t destination_id;
    uint16_t message_id;
    uint8_t message_type;
    uint8_t ttl;             // Time to live
    uint8_t priority;
    uint16_t payload_len;
    uint8_t *payload;
    uint8_t signature[32];   // Message signature
} mesh_message_t;

// Mesh network context
typedef struct {
    uint16_t node_id;
    node_type_t node_type;
    routing_table_t *routing_table;
    network_topology_t *topology;
    security_manager_t *security;
    uint16_t message_counter;
    uint32_t network_time;
    bool network_joined;
} mesh_network_t;

// Initialize mesh network
bool mesh_network_init(mesh_network_t *network, uint16_t node_id, node_type_t type) {
    if (!network) return false;
    
    network->node_id = node_id;
    network->node_type = type;
    network->message_counter = 1;
    network->network_time = 0;
    network->network_joined = false;
    
    // Initialize routing table
    network->routing_table = routing_table_create(50);
    if (!network->routing_table) return false;
    
    // Initialize network topology
    network->topology = network_topology_create();
    if (!network->topology) return false;
    
    // Initialize security manager
    network->security = security_manager_create();
    if (!network->security) return false;
    
    return true;
}

// Join mesh network
bool mesh_join_network(mesh_network_t *network, uint16_t coordinator_id) {
    if (!network || network->network_joined) return false;
    
    // Send join request
    mesh_message_t join_request = {
        .source_id = network->node_id,
        .destination_id = coordinator_id,
        .message_id = network->message_counter++,
        .message_type = MESH_MSG_TYPE_JOIN_REQUEST,
        .ttl = 5,
        .priority = 0,
        .payload_len = sizeof(uint16_t) + sizeof(node_type_t),
        .payload = (uint8_t*)&network->node_id
    };
    
    // Sign message
    if (!security_sign_message(network->security, &join_request)) {
        return false;
    }
    
    // Send join request
    if (!mesh_send_message(network, &join_request)) {
        return false;
    }
    
    // Wait for join response
    mesh_message_t *response = wait_for_message(network, MESH_MSG_TYPE_JOIN_RESPONSE, 10000);
    if (!response) {
        return false;
    }
    
    // Process join response
    if (response->payload_len >= sizeof(uint16_t)) {
        uint16_t assigned_id = *(uint16_t*)response->payload;
        
        if (assigned_id == network->node_id) {
            network->network_joined = true;
            
            // Add coordinator to routing table
            routing_table_add_entry(network->routing_table, coordinator_id, coordinator_id, 1, 100);
            
            // Start network discovery
            start_network_discovery(network);
            
            free(response->payload);
            free(response);
            return true;
        }
    }
    
    free(response->payload);
    free(response);
    return false;
}

// Send message through mesh network
bool mesh_send_message(mesh_network_t *network, const mesh_message_t *message) {
    if (!network || !message || !network->network_joined) return false;
    
    // Check if destination is direct neighbor
    if (is_direct_neighbor(network, message->destination_id)) {
        return send_direct_message(network, message);
    }
    
    // Find route to destination
    routing_entry_t *route = routing_table_find_route(network->routing_table, message->destination_id);
    if (!route) {
        // No route found, initiate route discovery
        return initiate_route_discovery(network, message->destination_id);
    }
    
    // Forward message through route
    return forward_message_through_route(network, message, route);
}

// Route discovery
bool initiate_route_discovery(mesh_network_t *network, uint16_t destination_id) {
    if (!network) return false;
    
    // Create route request message
    mesh_message_t route_request = {
        .source_id = network->node_id,
        .destination_id = destination_id,
        .message_id = network->message_counter++,
        .message_type = MESH_MSG_TYPE_ROUTE_REQUEST,
        .ttl = 10,
        .priority = 1,
        .payload_len = sizeof(uint16_t) * 2,
        .payload = malloc(sizeof(uint16_t) * 2)
    };
    
    if (!route_request.payload) return false;
    
    // Add source and destination IDs
    *(uint16_t*)route_request.payload = network->node_id;
    *(uint16_t*)(route_request.payload + 2) = destination_id;
    
    // Sign message
    if (!security_sign_message(network->security, &route_request)) {
        free(route_request.payload);
        return false;
    }
    
    // Broadcast route request
    bool success = broadcast_message(network, &route_request);
    
    free(route_request.payload);
    return success;
}

// Handle incoming messages
void mesh_handle_incoming(mesh_network_t *network, const mesh_message_t *message) {
    if (!network || !message) return;
    
    // Verify message signature
    if (!security_verify_message(network->security, message)) {
        return; // Invalid signature
    }
    
    // Check if message is for this node
    if (message->destination_id == network->node_id) {
        handle_local_message(network, message);
        return;
    }
    
    // Check TTL
    if (message->ttl == 0) {
        return; // Message expired
    }
    
    // Forward message if this is a router
    if (network->node_type == NODE_TYPE_ROUTER || 
        network->node_type == NODE_TYPE_COORDINATOR) {
        forward_message(network, message);
    }
}

// Handle local message
void handle_local_message(mesh_network_t *network, const mesh_message_t *message) {
    switch (message->message_type) {
        case MESH_MSG_TYPE_ROUTE_REQUEST:
            handle_route_request(network, message);
            break;
            
        case MESH_MSG_TYPE_ROUTE_REPLY:
            handle_route_reply(network, message);
            break;
            
        case MESH_MSG_TYPE_DATA:
            handle_data_message(network, message);
            break;
            
        case MESH_MSG_TYPE_TOPOLOGY_UPDATE:
            handle_topology_update(network, message);
            break;
            
        case MESH_MSG_TYPE_HEARTBEAT:
            handle_heartbeat(network, message);
            break;
            
        default:
            break;
    }
}

// Handle route request
void handle_route_request(mesh_network_t *network, const mesh_message_t *message) {
    if (!network || !message) return;
    
    uint16_t source_id = *(uint16_t*)message->payload;
    uint16_t destination_id = *(uint16_t*)(message->payload + 2);
    
    // Check if we have a route to destination
    routing_entry_t *route = routing_table_find_route(network->routing_table, destination_id);
    
    if (route) {
        // Send route reply
        mesh_message_t route_reply = {
            .source_id = network->node_id,
            .destination_id = source_id,
            .message_id = network->message_counter++,
            .message_type = MESH_MSG_TYPE_ROUTE_REPLY,
            .ttl = 10,
            .priority = 1,
            .payload_len = sizeof(uint16_t) * 3,
            .payload = malloc(sizeof(uint16_t) * 3)
        };
        
        if (!route_reply.payload) return;
        
        // Add route information
        *(uint16_t*)route_reply.payload = source_id;
        *(uint16_t*)(route_reply.payload + 2) = destination_id;
        *(uint16_t*)(route_reply.payload + 4) = route->next_hop;
        
        // Sign and send
        if (security_sign_message(network->security, &route_reply)) {
            mesh_send_message(network, &route_reply);
        }
        
        free(route_reply.payload);
    }
    
    // Re-broadcast if TTL > 1
    if (message->ttl > 1) {
        mesh_message_t *rebroadcast = copy_message(message);
        if (rebroadcast) {
            rebroadcast->ttl--;
            rebroadcast->source_id = network->node_id;
            rebroadcast->message_id = network->message_counter++;
            
            if (security_sign_message(network->security, rebroadcast)) {
                broadcast_message(network, rebroadcast);
            }
            
            free_message(rebroadcast);
        }
    }
}

// Network topology management
void update_network_topology(mesh_network_t *network, uint16_t neighbor_id, 
                           uint8_t link_quality, bool active) {
    if (!network || !network->topology) return;
    
    // Update neighbor information
    network_topology_update_neighbor(network->topology, neighbor_id, link_quality, active);
    
    // Update routing table
    routing_table_update_link_quality(network->routing_table, neighbor_id, link_quality);
    
    // Trigger topology optimization if needed
    if (should_optimize_topology(network)) {
        optimize_network_topology(network);
    }
}

// Network diagnostics
void mesh_network_diagnostics(mesh_network_t *network) {
    if (!network) return;
    
    // Print routing table
    printf("Routing Table:\n");
    routing_table_print(network->routing_table);
    
    // Print network topology
    printf("Network Topology:\n");
    network_topology_print(network->topology);
    
    // Print network statistics
    printf("Network Statistics:\n");
    printf("  Total nodes: %d\n", network_topology_get_node_count(network->topology));
    printf("  Active links: %d\n", network_topology_get_active_link_count(network->topology));
    printf("  Average link quality: %d\n", network_topology_get_average_link_quality(network->topology));
    
    // Check for network issues
    if (network_topology_get_average_link_quality(network->topology) < 50) {
        printf("Warning: Low link quality detected\n");
    }
    
    if (network_topology_get_disconnected_nodes(network->topology) > 0) {
        printf("Warning: Disconnected nodes detected\n");
    }
}
```

**Mesh Network Features**:
- **Self-Healing**: Automatic route discovery and recovery
- **Multi-Hop Routing**: Efficient routing through intermediate nodes
- **Load Balancing**: Dynamic route selection based on link quality
- **Network Diagnostics**: Comprehensive network monitoring and troubleshooting

## 🧪 **Practice Problems**

### **Problem 1: Power Consumption Analysis**

**Scenario**: Analyze power consumption of an IoT device.

**Question**: How would you optimize power consumption for a battery-powered sensor node?

**Expected Solution**:
```
1. Power Analysis:
   - Measure current consumption in different states
   - Identify power-hungry components
   - Analyze duty cycle requirements

2. Optimization Strategies:
   - Use low-power sleep modes
   - Implement duty cycling
   - Optimize wireless transmission
   - Use energy-efficient sensors

3. Power Management:
   - Dynamic frequency scaling
   - Adaptive sampling rates
   - Smart wake-up strategies
   - Power-aware routing
```

### **Problem 2: Wireless Protocol Selection**

**Scenario**: Choose appropriate wireless protocol for different IoT applications.

**Question**: Which protocol would you choose for each application and why?

**Expected Analysis**:
```
1. Smart Home Sensors:
   - Protocol: Zigbee or Bluetooth Low Energy
   - Reason: Low power, mesh networking, local control

2. Industrial Monitoring:
   - Protocol: LoRaWAN or cellular
   - Reason: Long range, high reliability, industrial standards

3. Asset Tracking:
   - Protocol: NB-IoT or LoRaWAN
   - Reason: Low bandwidth, long battery life, wide coverage

4. Wearable Devices:
   - Protocol: Bluetooth Low Energy
   - Reason: Low power, high data rate, smartphone compatibility
```

### **Problem 3: IoT Security Implementation**

**Scenario**: Design security for an IoT network.

**Question**: How would you implement security for an industrial IoT deployment?

**Expected Solution**:
```
1. Device Security:
   - Secure boot with TPM
   - Certificate-based authentication
   - Encrypted storage
   - Secure firmware updates

2. Communication Security:
   - TLS 1.3 for all connections
   - Certificate pinning
   - Message authentication
   - End-to-end encryption

3. Network Security:
   - Network segmentation
   - Intrusion detection
   - Traffic monitoring
   - Access control

4. Data Security:
   - Data encryption at rest
   - Secure key management
   - Audit logging
   - Privacy protection
```

## ✅ **Self-Assessment Checklist**

### **Wireless Protocols** ✅
- [ ] Can work with Bluetooth, WiFi, Zigbee
- [ ] Understand LoRa and cellular protocols
- [ ] Can select appropriate protocols
- [ ] Know protocol limitations and trade-offs

### **IoT Communication** ✅
- [ ] Can implement MQTT, CoAP clients
- [ ] Understand IoT communication patterns
- [ ] Can handle QoS and reliability
- [ ] Know IoT security requirements

### **Power Management** ✅
- [ ] Can design low-power systems
- [ ] Understand power optimization techniques
- [ ] Can implement duty cycling
- [ ] Know energy harvesting options

### **Network Design** ✅
- [ ] Can design mesh networks
- [ ] Understand routing protocols
- [ ] Can implement network diagnostics
- [ ] Know network optimization techniques

## 🔗 **Related Topics**
- [Communication Protocols](../Communication_Protocols/README.md)
- [Power Management](../Hardware_Fundamentals/Power_Management.md)
- [Security Fundamentals](../Embedded_Security/Security_Fundamentals.md)
- [Network Protocols](../Communication_Protocols/Network_Protocols.md)
- [Low-Power Design](../Advanced_Hardware/Low_Power_Design.md)

## 📚 **Additional Resources**
- **Books**: "Building Wireless Sensor Networks" by Robert Faludi
- **Online**: [Bluetooth SIG](https://www.bluetooth.com/)
- **Practice**: [MQTT Test Broker](https://test.mosquitto.org/)
- **Standards**: [IEEE 802.15.4](https://standards.ieee.org/standard/802_15_4-2015.html)
