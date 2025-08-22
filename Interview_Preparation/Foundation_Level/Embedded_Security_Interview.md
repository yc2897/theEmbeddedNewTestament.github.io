# 🔒 Embedded Security Interview Preparation

## 🚀 **Quick Navigation**
- [Security Fundamentals](#security-fundamentals)
- [Cryptographic Implementation](#cryptographic-implementation)
- [Secure Boot & Chain of Trust](#secure-boot--chain-of-trust)
- [Memory Protection](#memory-protection)
- [Security Testing](#security-testing)

## 📚 **Quick Reference: Key Concepts**
- **Security Principles**: Confidentiality, Integrity, Availability (CIA triad)
- **Cryptography**: Symmetric/asymmetric encryption, hashing, digital signatures
- **Secure Boot**: Verified boot process, chain of trust, TPM integration
- **Memory Protection**: MPU, secure enclaves, buffer overflow prevention
- **Security Testing**: Penetration testing, fuzzing, side-channel analysis

## 🔒 **Security Fundamentals**

### **Security Principles**

**CIA Triad**:
```
1. Confidentiality: Data is accessible only to authorized users
2. Integrity: Data remains unaltered and authentic
3. Availability: System and data are accessible when needed
```

**Threat Model**:
```
1. Attack Vectors
   - Physical access to hardware
   - Network-based attacks
   - Supply chain compromise
   - Side-channel attacks

2. Attack Types
   - Man-in-the-middle
   - Replay attacks
   - Buffer overflows
   - Timing attacks
```

### **Security Architecture**

**Layered Security**:
```
┌─────────────────────────────────────┐
│           Application Layer          │
├─────────────────────────────────────┤
│           Runtime Security          │
├─────────────────────────────────────┤
│           OS Security               │
├─────────────────────────────────────┤
│           Hardware Security         │
├─────────────────────────────────────┤
│           Physical Security         │
└─────────────────────────────────────┘
```

## 🔒 **Cryptographic Implementation**

### **Hash Functions**

**SHA-256 Implementation**:
```c
#include <stdint.h>
#include <string.h>

// SHA-256 constants
static const uint32_t K[64] = {
    0x428a2f98, 0x71374491, 0xb5c0fbcf, 0xe9b5dba5,
    0x3956c25b, 0x59f111f1, 0x923f82a4, 0xab1c5ed5,
    // ... more constants
};

// SHA-256 context
typedef struct {
    uint32_t state[8];
    uint32_t count[2];
    uint8_t buffer[64];
} sha256_context_t;

// Initialize SHA-256
void sha256_init(sha256_context_t *ctx) {
    ctx->state[0] = 0x6a09e667;
    ctx->state[1] = 0xbb67ae85;
    ctx->state[2] = 0x3c6ef372;
    ctx->state[3] = 0xa54ff53a;
    ctx->state[4] = 0x510e527f;
    ctx->state[5] = 0x9b05688c;
    ctx->state[6] = 0x1f83d9ab;
    ctx->state[7] = 0x5be0cd19;
    ctx->count[0] = ctx->count[1] = 0;
}

// SHA-256 transform
void sha256_transform(sha256_context_t *ctx, const uint8_t *data) {
    uint32_t a, b, c, d, e, f, g, h;
    uint32_t W[64];
    uint32_t temp1, temp2;
    
    // Prepare message schedule
    for (int i = 0; i < 16; i++) {
        W[i] = (data[i*4] << 24) | (data[i*4+1] << 16) | 
                (data[i*4+2] << 8) | data[i*4+3];
    }
    
    for (int i = 16; i < 64; i++) {
        W[i] = W[i-16] + W[i-7] + 
                ((W[i-15] >> 7) | (W[i-15] << 25)) +
                ((W[i-2] >> 17) | (W[i-2] << 15));
    }
    
    // Initialize working variables
    a = ctx->state[0]; b = ctx->state[1]; c = ctx->state[2]; d = ctx->state[3];
    e = ctx->state[4]; f = ctx->state[5]; g = ctx->state[6]; h = ctx->state[7];
    
    // Main loop
    for (int i = 0; i < 64; i++) {
        temp1 = h + ((e >> 6) | (e << 26)) + ((e >> 11) | (e << 21)) + 
                (e ^ f ^ g) + K[i] + W[i];
        temp2 = ((a >> 2) | (a << 30)) + ((a >> 13) | (a << 19)) + 
                (a ^ b ^ c) + ((b & c) ^ (a & (b ^ c)));
        
        h = g; g = f; f = e; e = d + temp1;
        d = c; c = b; b = a; a = temp1 + temp2;
    }
    
    // Update state
    ctx->state[0] += a; ctx->state[1] += b; ctx->state[2] += c; ctx->state[3] += d;
    ctx->state[4] += e; ctx->state[5] += f; ctx->state[6] += g; ctx->state[7] += h;
}

// Calculate SHA-256 hash
void sha256_calculate(const uint8_t *data, size_t length, uint8_t *hash) {
    sha256_context_t ctx;
    sha256_init(&ctx);
    
    // Process data in blocks
    size_t remaining = length;
    while (remaining >= 64) {
        sha256_transform(&ctx, data);
        data += 64;
        remaining -= 64;
        ctx.count[0] += 64 * 8;
    }
    
    // Finalize
    // ... padding and final transform
}
```

### **AES Encryption**

**AES-128 Implementation**:
```c
#include <stdint.h>

// AES S-box
static const uint8_t SBOX[256] = {
    0x63, 0x7c, 0x77, 0x7b, 0xf2, 0x6b, 0x6f, 0xc5,
    // ... full S-box
};

// AES round constants
static const uint8_t RCON[10] = {
    0x01, 0x02, 0x04, 0x08, 0x10, 0x20, 0x40, 0x80, 0x1b, 0x36
};

// AES key expansion
void aes_key_expansion(const uint8_t *key, uint8_t *round_keys) {
    uint8_t temp[4];
    
    // Copy initial key
    memcpy(round_keys, key, 16);
    
    for (int i = 4; i < 44; i++) {
        memcpy(temp, round_keys + (i-1) * 4, 4);
        
        if (i % 4 == 0) {
            // Rotate and substitute
            uint8_t t = temp[0];
            temp[0] = temp[1]; temp[1] = temp[2]; 
            temp[2] = temp[3]; temp[3] = t;
            
            for (int j = 0; j < 4; j++) {
                temp[j] = SBOX[temp[j]];
            }
            
            temp[0] ^= RCON[i/4 - 1];
        }
        
        // XOR with previous round key
        for (int j = 0; j < 4; j++) {
            round_keys[i * 4 + j] = round_keys[(i-4) * 4 + j] ^ temp[j];
        }
    }
}

// AES encryption
void aes_encrypt(const uint8_t *plaintext, const uint8_t *round_keys, uint8_t *ciphertext) {
    uint8_t state[16];
    memcpy(state, plaintext, 16);
    
    // Initial round key addition
    for (int i = 0; i < 16; i++) {
        state[i] ^= round_keys[i];
    }
    
    // Main rounds
    for (int round = 1; round < 10; round++) {
        // SubBytes, ShiftRows, MixColumns, AddRoundKey
        aes_sub_bytes(state);
        aes_shift_rows(state);
        aes_mix_columns(state);
        aes_add_round_key(state, round_keys + round * 16);
    }
    
    // Final round
    aes_sub_bytes(state);
    aes_shift_rows(state);
    aes_add_round_key(state, round_keys + 160);
    
    memcpy(ciphertext, state, 16);
}
```

## 🔒 **Secure Boot & Chain of Trust**

### **Secure Boot Implementation**

**Boot Verification**:
```c
#include <stdint.h>
#include <stdbool.h>

// Boot image header
typedef struct {
    uint32_t magic;
    uint32_t version;
    uint32_t size;
    uint8_t hash[32];
    uint8_t signature[256];
} secure_boot_header_t;

// Verify boot image
bool verify_boot_image(const secure_boot_header_t *header, const uint8_t *image_data) {
    // Check magic number
    if (header->magic != SECURE_BOOT_MAGIC) {
        return false;
    }
    
    // Calculate hash of image data
    uint8_t calculated_hash[32];
    sha256_calculate(image_data, header->size, calculated_hash);
    
    // Verify hash
    if (memcmp(calculated_hash, header->hash, 32) != 0) {
        return false;
    }
    
    // Verify signature
    return verify_signature(calculated_hash, header->signature, PUBLIC_KEY);
}

// Secure boot sequence
bool secure_boot(void) {
    // Load bootloader from secure storage
    secure_boot_header_t bootloader_header;
    uint8_t *bootloader_data;
    
    if (!load_bootloader(&bootloader_header, &bootloader_data)) {
        return false;
    }
    
    // Verify bootloader
    if (!verify_boot_image(&bootloader_header, bootloader_data)) {
        return false;
    }
    
    // Load and verify application
    secure_boot_header_t app_header;
    uint8_t *app_data;
    
    if (!load_application(&app_header, &app_data)) {
        return false;
    }
    
    if (!verify_boot_image(&app_header, app_data)) {
        return false;
    }
    
    // Jump to verified application
    jump_to_application(app_data);
    return true;
}
```

### **TPM Integration**

**TPM 2.0 Operations**:
```c
#include <stdint.h>
#include <stdbool.h>

// TPM context
typedef struct {
    uint32_t tpm_handle;
    bool initialized;
} tpm_context_t;

// Initialize TPM
bool tpm_init(tpm_context_t *tpm) {
    // Initialize TPM hardware interface
    if (!init_tpm_hardware()) {
        return false;
    }
    
    // Start TPM
    if (!tpm_startup()) {
        return false;
    }
    
    tpm->initialized = true;
    return true;
}

// Extend PCR (Platform Configuration Register)
bool tpm_extend_pcr(tpm_context_t *tpm, uint32_t pcr_index, const uint8_t *data, size_t data_size) {
    uint8_t digest[32];
    
    // Calculate hash
    sha256_calculate(data, data_size, digest);
    
    // TPM2_PCR_Extend command
    uint8_t command[256];
    uint16_t command_size = build_pcr_extend_command(pcr_index, digest, command);
    
    // Send command to TPM
    uint8_t response[256];
    uint16_t response_size;
    
    if (!tpm_send_command(command, command_size, response, &response_size)) {
        return false;
    }
    
    return parse_pcr_extend_response(response, response_size);
}

// Quote operation (attestation)
bool tpm_quote(tpm_context_t *tpm, const uint8_t *nonce, uint8_t *quote, uint8_t *signature) {
    // Build quote command
    uint8_t command[512];
    uint16_t command_size = build_quote_command(nonce, command);
    
    // Send command to TPM
    uint8_t response[512];
    uint16_t response_size;
    
    if (!tpm_send_command(command, command_size, response, &response_size)) {
        return false;
    }
    
    // Parse response
    return parse_quote_response(response, response_size, quote, signature);
}
```

## 🔒 **Memory Protection**

### **MPU Configuration**

**Memory Protection Unit Setup**:
```c
#include <stdint.h>
#include <stdbool.h>

// MPU region configuration
typedef struct {
    uint32_t base_address;
    uint32_t size;
    uint8_t access_permissions;
    uint8_t attributes;
    bool enabled;
} mpu_region_t;

// Configure MPU region
bool configure_mpu_region(uint8_t region_number, const mpu_region_t *config) {
    if (region_number >= MAX_MPU_REGIONS) {
        return false;
    }
    
    // Disable region first
    disable_mpu_region(region_number);
    
    // Configure region base address and size
    set_mpu_region_base(region_number, config->base_address);
    set_mpu_region_size(region_number, config->size);
    
    // Configure access permissions
    set_mpu_region_access(region_number, config->access_permissions);
    
    // Configure memory attributes
    set_mpu_region_attributes(region_number, config->attributes);
    
    // Enable region
    if (config->enabled) {
        enable_mpu_region(region_number);
    }
    
    return true;
}

// Secure memory layout
void configure_secure_memory_layout(void) {
    // Flash memory (executable, read-only)
    mpu_region_t flash_region = {
        .base_address = FLASH_BASE,
        .size = FLASH_SIZE,
        .access_permissions = MPU_READ_ONLY | MPU_EXECUTABLE,
        .attributes = MPU_NORMAL_MEMORY,
        .enabled = true
    };
    configure_mpu_region(0, &flash_region);
    
    // RAM (read-write, non-executable)
    mpu_region_t ram_region = {
        .base_address = RAM_BASE,
        .size = RAM_SIZE,
        .access_permissions = MPU_READ_WRITE,
        .attributes = MPU_NORMAL_MEMORY,
        .enabled = true
    };
    configure_mpu_region(1, &ram_region);
    
    // Peripherals (read-write, non-executable)
    mpu_region_t peripheral_region = {
        .base_address = PERIPHERAL_BASE,
        .size = PERIPHERAL_SIZE,
        .access_permissions = MPU_READ_WRITE,
        .attributes = MPU_DEVICE_MEMORY,
        .enabled = true
    };
    configure_mpu_region(2, &peripheral_region);
}
```

### **Buffer Overflow Protection**

**Stack Canary Implementation**:
```c
#include <stdint.h>
#include <stdbool.h>

// Stack canary
static uint32_t stack_canary = 0xDEADBEEF;

// Initialize stack canary
void init_stack_canary(void) {
    // Generate random canary
    stack_canary = generate_random_uint32();
}

// Check stack canary
bool check_stack_canary(void) {
    // This would be implemented in assembly
    // to check the canary value on the stack
    return true;
}

// Function with stack protection
void secure_function(void) {
    uint32_t local_canary = stack_canary;
    
    // Function body
    // ...
    
    // Check canary before return
    if (local_canary != stack_canary) {
        // Stack corruption detected
        handle_stack_corruption();
    }
}
```

## 🔒 **Security Testing**

### **Penetration Testing**

**Security Test Framework**:
```c
#include <stdint.h>
#include <stdbool.h>

// Security test results
typedef struct {
    bool passed;
    char description[128];
    uint32_t test_duration;
    uint32_t vulnerabilities_found;
} security_test_result_t;

// Buffer overflow test
security_test_result_t test_buffer_overflow(void) {
    security_test_result_t result = {0};
    strcpy(result.description, "Buffer Overflow Test");
    
    uint32_t start_time = get_system_time();
    
    // Test various buffer sizes
    for (int size = 1; size <= 1024; size *= 2) {
        uint8_t *buffer = malloc(size);
        if (!buffer) continue;
        
        // Try to overflow buffer
        uint8_t *overflow_data = malloc(size + 100);
        if (overflow_data) {
            memset(overflow_data, 0xAA, size + 100);
            
            // This should cause overflow
            memcpy(buffer, overflow_data, size + 100);
            
            free(overflow_data);
        }
        
        free(buffer);
    }
    
    result.test_duration = get_system_time() - start_time;
    result.passed = true;  // Would check for actual overflow detection
    
    return result;
}

// Timing attack test
security_test_result_t test_timing_attack(void) {
    security_test_result_t result = {0};
    strcpy(result.description, "Timing Attack Test");
    
    uint32_t start_time = get_system_time();
    
    // Test constant-time comparison
    uint8_t secret[] = "secret123";
    uint8_t guess[] = "secret123";
    
    uint32_t time1 = get_system_time();
    bool match1 = constant_time_compare(secret, guess, strlen(secret));
    uint32_t time2 = get_system_time();
    
    uint32_t time_diff = time2 - time1;
    
    // Check if timing is consistent
    if (time_diff < MAX_TIMING_VARIATION) {
        result.passed = true;
    } else {
        result.vulnerabilities_found++;
    }
    
    result.test_duration = get_system_time() - start_time;
    return result;
}
```

## 🧪 **Common Interview Questions**

### **Question 1: Implement Secure Communication**

**Problem**: Design a secure communication protocol between two embedded devices.

**Solution Approach**:
```
1. Protocol Design:
   - Key exchange using Diffie-Hellman
   - AES encryption for data
   - HMAC for message authentication
   - Nonce for replay protection

2. Implementation:
   - Secure random number generation
   - Constant-time cryptographic operations
   - Secure key storage
   - Error handling
```

**Implementation**:
```c
// Secure communication context
typedef struct {
    uint8_t shared_key[32];
    uint8_t session_nonce[16];
    uint32_t message_counter;
    bool key_established;
} secure_comm_context_t;

// Establish secure connection
bool establish_secure_connection(secure_comm_context_t *ctx) {
    // Generate private key
    uint8_t private_key[32];
    generate_random_bytes(private_key, 32);
    
    // Perform key exchange
    uint8_t public_key[32];
    if (!diffie_hellman_key_exchange(private_key, public_key, ctx->shared_key)) {
        return false;
    }
    
    // Generate session nonce
    generate_random_bytes(ctx->session_nonce, 16);
    ctx->message_counter = 0;
    ctx->key_established = true;
    
    return true;
}

// Send secure message
bool send_secure_message(secure_comm_context_t *ctx, const uint8_t *data, uint16_t length) {
    if (!ctx->key_established) return false;
    
    // Prepare message
    uint8_t message[512];
    uint16_t message_size = 0;
    
    // Add nonce and counter
    memcpy(message, ctx->session_nonce, 16);
    memcpy(message + 16, &ctx->message_counter, 4);
    message_size = 20;
    
    // Encrypt data
    uint8_t encrypted_data[length + 16];  // +16 for padding
    uint16_t encrypted_size;
    
    if (!aes_encrypt_cbc(data, length, ctx->shared_key, ctx->session_nonce, 
                         encrypted_data, &encrypted_size)) {
        return false;
    }
    
    // Add encrypted data
    memcpy(message + message_size, encrypted_data, encrypted_size);
    message_size += encrypted_size;
    
    // Calculate HMAC
    uint8_t hmac[32];
    calculate_hmac(message, message_size, ctx->shared_key, hmac);
    
    // Add HMAC
    memcpy(message + message_size, hmac, 32);
    message_size += 32;
    
    // Send message
    bool success = send_message(message, message_size);
    
    if (success) {
        ctx->message_counter++;
    }
    
    return success;
}
```

### **Question 2: Secure Firmware Update**

**Problem**: Implement a secure firmware update mechanism.

**Solution Approach**:
```
1. Security Requirements:
   - Firmware authenticity verification
   - Integrity checking
   - Rollback protection
   - Secure storage

2. Implementation:
   - Digital signature verification
   - Secure boot integration
   - Version management
   - Error recovery
```

**Implementation**:
```c
// Firmware update context
typedef struct {
    uint32_t current_version;
    uint32_t new_version;
    uint8_t firmware_hash[32];
    uint8_t signature[256];
    bool update_pending;
} firmware_update_context_t;

// Verify firmware update
bool verify_firmware_update(const uint8_t *firmware_data, size_t firmware_size,
                           const firmware_update_context_t *update_info) {
    // Check version
    if (update_info->new_version <= update_info->current_version) {
        return false;
    }
    
    // Verify hash
    uint8_t calculated_hash[32];
    sha256_calculate(firmware_data, firmware_size, calculated_hash);
    
    if (memcmp(calculated_hash, update_info->firmware_hash, 32) != 0) {
        return false;
    }
    
    // Verify signature
    return verify_signature(calculated_hash, update_info->signature, UPDATE_PUBLIC_KEY);
}

// Apply firmware update
bool apply_firmware_update(const uint8_t *firmware_data, size_t firmware_size) {
    // Backup current firmware
    if (!backup_current_firmware()) {
        return false;
    }
    
    // Write new firmware to backup location
    if (!write_firmware_to_backup(firmware_data, firmware_size)) {
        return false;
    }
    
    // Verify backup
    if (!verify_backup_firmware()) {
        restore_original_firmware();
        return false;
    }
    
    // Switch to new firmware
    if (!switch_to_new_firmware()) {
        restore_original_firmware();
        return false;
    }
    
    return true;
}
```

## 🧪 **Practice Problems**

### **Problem 1: Secure Key Storage**

**Scenario**: Design a system to securely store cryptographic keys in an embedded device.

**Question**: Implement secure key storage with hardware protection.

**Expected Analysis**:
```
1. Security Requirements:
   - Key confidentiality
   - Access control
   - Tamper detection
   - Secure deletion

2. Implementation:
   - Hardware security module
   - Key derivation functions
   - Access policies
   - Audit logging
```

### **Problem 2: Side-Channel Attack Prevention**

**Scenario**: Implement cryptographic functions resistant to timing and power analysis attacks.

**Question**: Design constant-time cryptographic operations.

**Expected Analysis**:
```
1. Attack Vectors:
   - Timing analysis
   - Power analysis
   - Cache attacks
   - Electromagnetic analysis

2. Countermeasures:
   - Constant-time algorithms
   - Random delays
   - Power masking
   - Cache isolation
```

## ✅ **Self-Assessment Checklist**

### **Security Fundamentals** ✅
- [ ] Can explain CIA triad and security principles
- [ ] Can identify common attack vectors
- [ ] Can design security architecture
- [ ] Can implement threat modeling

### **Cryptography** ✅
- [ ] Can implement hash functions
- [ ] Can implement encryption algorithms
- [ ] Can handle key management
- [ ] Can prevent side-channel attacks

### **Secure Boot** ✅
- [ ] Can implement secure boot process
- [ ] Can verify firmware integrity
- [ ] Can integrate TPM functionality
- [ ] Can establish chain of trust

### **Memory Protection** ✅
- [ ] Can configure MPU regions
- [ ] Can prevent buffer overflows
- [ ] Can implement secure memory layout
- [ ] Can handle memory access violations

### **Security Testing** ✅
- [ ] Can perform penetration testing
- [ ] Can identify vulnerabilities
- [ ] Can implement security test frameworks
- [ ] Can validate security measures

## 🔗 **Related Topics**
- [C Programming Interview](./C_Programming_Interview.md)
- [Embedded Security Interview](../Advanced_Level/Embedded_Security_Interview.md)
- [System Integration Interview](../Intermediate_Level/System_Integration_Interview.md)
- [Performance Optimization Interview](../Advanced_Level/Performance_Optimization_Interview.md)

## 📚 **Additional Resources**
- **Security Standards**: [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework)
- **Cryptography**: [Cryptographic Standards](https://www.nist.gov/cryptography)
- **Secure Boot**: [UEFI Secure Boot](https://uefi.org/secureboot)
- **Security Testing**: [OWASP Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)
