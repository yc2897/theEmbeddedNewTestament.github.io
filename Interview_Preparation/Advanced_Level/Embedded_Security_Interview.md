# 🎯 Embedded Security Interview Preparation

## 🚀 **Quick Navigation**
- [Common Questions](#common-questions)
- [Problem-Solving Examples](#problem-solving-examples)
- [Practice Problems](#practice-problems)
- [Resources](#resources)

## 📚 **Quick Reference: Key Concepts**
- **Secure Boot**: Chain of trust, signature verification, anti-rollback protection
- **Cryptography**: Symmetric/asymmetric algorithms, key management, secure storage
- **Side-Channel Attacks**: Timing attacks, power analysis, fault injection
- **Platform Security**: ARM TrustZone, secure enclaves, memory protection
- **Security Protocols**: TLS, secure communication, authentication mechanisms

## 🎯 **Common Interview Questions**

### **Question 1: Implement a secure boot system with TPM integration**

**Why this matters**: Secure boot is critical for preventing unauthorized firmware and maintaining system integrity.

**Problem**: Design a secure boot system that validates firmware using TPM 2.0.

**Requirements**:
- Validate bootloader signature
- Measure firmware into TPM PCRs
- Prevent rollback attacks
- Handle update scenarios

**Solution Design**:
```c
typedef struct {
    uint32_t magic_number;
    uint32_t version;
    uint32_t size;
    uint8_t hash[SHA256_DIGEST_SIZE];
    uint8_t signature[RSA_SIGNATURE_SIZE];
    uint32_t flags;
} secure_firmware_header_t;

typedef struct {
    TPM2_HANDLE tpm_handle;
    uint8_t pcr_values[24][SHA256_DIGEST_SIZE];
    bool pcr_extended;
} tpm_context_t;

// TPM 2.0 PCR extension
bool extend_pcr(tpm_context_t* tpm, uint32_t pcr_index, const uint8_t* data, size_t data_size) {
    uint8_t digest[SHA256_DIGEST_SIZE];
    
    // Calculate SHA256 hash
    if (!sha256_calculate(data, data_size, digest)) {
        return false;
    }
    
    // TPM2_PCR_Extend command
    TPM2_PCR_EXTEND_CMD cmd = {
        .pcrHandle = pcr_index,
        .digests.count = 1,
        .digests.digests[0].hashAlg = TPM2_ALG_SHA256,
        .digests.digests[0].digest = {0}
    };
    memcpy(cmd.digests.digests[0].digest, digest, SHA256_DIGEST_SIZE);
    
    // Send command to TPM
    TPM2_PCR_EXTEND_RSP rsp;
    if (!tpm2_send_command(TPM2_CC_PCR_Extend, &cmd, sizeof(cmd), &rsp, sizeof(rsp))) {
        return false;
    }
    
    // Update local PCR values
    memcpy(tpm->pcr_values[pcr_index], digest, SHA256_DIGEST_SIZE);
    tpm->pcr_extended = true;
    
    return true;
}

// Secure boot sequence
boot_result_t secure_boot_with_tpm(void) {
    tpm_context_t tpm = {0};
    
    // Initialize TPM
    if (!tpm2_init(&tpm.tpm_handle)) {
        return BOOT_TPM_INIT_FAILED;
    }
    
    // Measure bootloader into PCR 0
    if (!extend_pcr(&tpm, 0, (uint8_t*)BOOTLOADER_START, BOOTLOADER_SIZE)) {
        return BOOT_TPM_MEASUREMENT_FAILED;
    }
    
    // Validate bootloader signature
    if (!verify_bootloader_signature()) {
        return BOOT_INVALID_SIGNATURE;
    }
    
    // Check for firmware update
    if (check_firmware_update()) {
        if (!secure_firmware_update(&tpm)) {
            return BOOT_UPDATE_FAILED;
        }
    }
    
    // Measure application firmware into PCR 1
    secure_firmware_header_t* header = (secure_firmware_header_t*)APP_START_ADDRESS;
    
    if (!extend_pcr(&tpm, 1, (uint8_t*)header, header->size)) {
        return BOOT_TPM_MEASUREMENT_FAILED;
    }
    
    // Verify firmware signature
    if (!verify_firmware_signature(header)) {
        return BOOT_INVALID_SIGNATURE;
    }
    
    // Verify firmware hash
    uint8_t calculated_hash[SHA256_DIGEST_SIZE];
    if (!sha256_calculate((uint8_t*)header, header->size, calculated_hash)) {
        return BOOT_HASH_CALCULATION_FAILED;
    }
    
    if (memcmp(calculated_hash, header->hash, SHA256_DIGEST_SIZE) != 0) {
        return BOOT_HASH_MISMATCH;
    }
    
    // Check rollback protection
    if (!check_rollback_protection(&tpm, header->version)) {
        return BOOT_VERSION_ROLLBACK;
    }
    
    // Final PCR measurement for boot success
    if (!extend_pcr(&tpm, 2, (uint8_t*)"BOOT_SUCCESS", 12)) {
        return BOOT_TPM_MEASUREMENT_FAILED;
    }
    
    return BOOT_SUCCESS;
}

// Rollback protection using TPM
bool check_rollback_protection(tpm_context_t* tpm, uint32_t firmware_version) {
    // Store version in TPM NV index
    uint32_t stored_version = 0;
    
    if (!tpm2_nv_read(tpm->tpm_handle, VERSION_NV_INDEX, (uint8_t*)&stored_version, sizeof(stored_version))) {
        // First boot, store current version
        return tpm2_nv_write(tpm->tpm_handle, VERSION_NV_INDEX, (uint8_t*)&firmware_version, sizeof(firmware_version));
    }
    
    // Check if new version is higher
    if (firmware_version <= stored_version) {
        return false;  // Rollback detected
    }
    
    // Update stored version
    return tpm2_nv_write(tpm->tpm_handle, VERSION_NV_INDEX, (uint8_t*)&firmware_version, sizeof(firmware_version));
}
```

**Security Features**:
- **TPM Integration**: Hardware-based security measurements
- **PCR Extensions**: Chain of trust verification
- **Rollback Protection**: Version control using TPM NV storage
- **Signature Verification**: Cryptographic validation of all components

**Follow-up Questions**:
- How would you handle TPM failures?
- What if the TPM is compromised?

### **Question 2: Implement side-channel attack countermeasures**

**Problem**: Design a cryptographic implementation that's resistant to timing and power analysis attacks.

**Requirements**:
- Constant-time operations
- Power analysis resistance
- Fault injection protection
- Secure key storage

**Solution Design**:
```c
// Constant-time AES implementation
void aes_encrypt_constant_time(const uint8_t* key, const uint8_t* plaintext, uint8_t* ciphertext) {
    // Use lookup tables with constant memory access patterns
    static const uint8_t sbox[256] = AES_SBOX;
    static const uint8_t inv_sbox[256] = AES_INV_SBOX;
    
    // Key expansion (constant time)
    uint32_t round_keys[44];
    aes_key_expansion(key, round_keys);
    
    // Initial round
    uint8_t state[16];
    memcpy(state, plaintext, 16);
    add_round_key(state, round_keys, 0);
    
    // Main rounds (constant time)
    for (int round = 1; round < 10; round++) {
        // SubBytes - constant time using lookup
        for (int i = 0; i < 16; i++) {
            state[i] = sbox[state[i]];
        }
        
        // ShiftRows - constant time
        shift_rows_constant_time(state);
        
        // MixColumns - constant time
        mix_columns_constant_time(state);
        
        // AddRoundKey
        add_round_key(state, round_keys, round);
    }
    
    // Final round
    for (int i = 0; i < 16; i++) {
        state[i] = sbox[state[i]];
    }
    shift_rows_constant_time(state);
    add_round_key(state, round_keys, 10);
    
    memcpy(ciphertext, state, 16);
}

// Constant-time shift rows
void shift_rows_constant_time(uint8_t* state) {
    // Row 0: no shift
    // Row 1: shift left by 1
    uint8_t temp = state[1];
    state[1] = state[5];
    state[5] = state[9];
    state[9] = state[13];
    state[13] = temp;
    
    // Row 2: shift left by 2
    temp = state[2];
    state[2] = state[10];
    state[10] = temp;
    temp = state[6];
    state[6] = state[14];
    state[14] = temp;
    
    // Row 3: shift left by 3
    temp = state[3];
    state[3] = state[15];
    state[15] = state[11];
    state[11] = state[7];
    state[7] = temp;
}

// Power analysis resistant key comparison
bool secure_key_compare(const uint8_t* key1, const uint8_t* key2, size_t length) {
    uint8_t result = 0;
    
    // Constant-time comparison
    for (size_t i = 0; i < length; i++) {
        result |= key1[i] ^ key2[i];
    }
    
    // Return true if all bytes match (result == 0)
    return (result == 0);
}

// Fault injection protection
bool verify_checksum_with_protection(const uint8_t* data, size_t data_size, uint32_t expected_checksum) {
    // Calculate checksum multiple times
    uint32_t checksum1 = calculate_crc32(data, data_size);
    uint32_t checksum2 = calculate_crc32(data, data_size);
    uint32_t checksum3 = calculate_crc32(data, data_size);
    
    // All three must match
    if (checksum1 != checksum2 || checksum2 != checksum3) {
        return false;  // Fault injection detected
    }
    
    // Compare with expected value
    return (checksum1 == expected_checksum);
}
```

**Side-Channel Countermeasures**:
- **Constant-Time Operations**: Eliminate timing variations
- **Power Analysis Resistance**: Use lookup tables and constant patterns
- **Fault Injection Protection**: Multiple calculations and verification
- **Secure Key Storage**: Hardware security modules when possible

### **Question 3: Design a secure communication protocol for embedded devices**

**Problem**: Create a secure communication protocol that protects against man-in-the-middle attacks and ensures data integrity.

**Requirements**:
- Mutual authentication
- Encrypted communication
- Integrity verification
- Session management

**Solution Design**:
```c
typedef struct {
    uint8_t device_id[32];
    uint8_t public_key[64];
    uint32_t session_id;
    uint32_t sequence_number;
} device_identity_t;

typedef struct {
    uint8_t session_key[32];
    uint8_t iv[16];
    uint32_t session_id;
    uint32_t sequence_number;
    uint64_t timestamp;
} session_context_t;

// Secure handshake protocol
bool establish_secure_session(device_identity_t* local_device, 
                            device_identity_t* remote_device,
                            session_context_t* session) {
    // Generate random challenge
    uint8_t challenge[32];
    if (!generate_random_bytes(challenge, sizeof(challenge))) {
        return false;
    }
    
    // Send challenge to remote device
    secure_message_t challenge_msg = {
        .type = MSG_TYPE_CHALLENGE,
        .data = challenge,
        .data_size = sizeof(challenge)
    };
    
    if (!send_secure_message(&challenge_msg)) {
        return false;
    }
    
    // Receive challenge response
    secure_message_t response_msg;
    if (!receive_secure_message(&response_msg)) {
        return false;
    }
    
    // Verify challenge response
    if (!verify_challenge_response(challenge, &response_msg, remote_device)) {
        return false;
    }
    
    // Generate session key
    uint8_t session_key[32];
    if (!generate_session_key(local_device, remote_device, challenge, session_key)) {
        return false;
    }
    
    // Initialize session context
    session->session_id = generate_random_uint32();
    session->sequence_number = 0;
    memcpy(session->session_key, session_key, sizeof(session_key));
    
    // Generate random IV
    if (!generate_random_bytes(session->iv, sizeof(session->iv))) {
        return false;
    }
    
    return true;
}

// Encrypted message transmission
bool send_encrypted_message(session_context_t* session, 
                           const uint8_t* data, 
                           size_t data_size) {
    // Create message header
    message_header_t header = {
        .session_id = session->session_id,
        .sequence_number = session->sequence_number++,
        .timestamp = get_system_time(),
        .data_size = data_size
    };
    
    // Calculate HMAC for integrity
    uint8_t hmac[32];
    if (!calculate_hmac(session->session_key, sizeof(session->session_key),
                       (uint8_t*)&header, sizeof(header),
                       data, data_size, hmac)) {
        return false;
    }
    
    // Encrypt data
    uint8_t encrypted_data[data_size + 16];  // +16 for padding
    size_t encrypted_size;
    
    if (!aes_encrypt_cbc(session->session_key, session->iv,
                         data, data_size,
                         encrypted_data, &encrypted_size)) {
        return false;
    }
    
    // Create final message
    secure_message_t secure_msg = {
        .type = MSG_TYPE_ENCRYPTED,
        .header = header,
        .hmac = hmac,
        .data = encrypted_data,
        .data_size = encrypted_size
    };
    
    // Send message
    return send_secure_message(&secure_msg);
}

// Message verification and decryption
bool receive_encrypted_message(session_context_t* session,
                              uint8_t* data,
                              size_t* data_size) {
    secure_message_t secure_msg;
    
    if (!receive_secure_message(&secure_msg)) {
        return false;
    }
    
    // Verify session ID
    if (secure_msg.header.session_id != session->session_id) {
        return false;
    }
    
    // Verify sequence number
    if (secure_msg.header.sequence_number <= session->sequence_number) {
        return false;  // Replay attack detected
    }
    
    // Verify HMAC
    uint8_t calculated_hmac[32];
    if (!calculate_hmac(session->session_key, sizeof(session->session_key),
                       (uint8_t*)&secure_msg.header, sizeof(secure_msg.header),
                       secure_msg.data, secure_msg.data_size, calculated_hmac)) {
        return false;
    }
    
    if (memcmp(calculated_hmac, secure_msg.hmac, sizeof(hmac)) != 0) {
        return false;  // Integrity check failed
    }
    
    // Decrypt data
    if (!aes_decrypt_cbc(session->session_key, session->iv,
                         secure_msg.data, secure_msg.data_size,
                         data, data_size)) {
        return false;
    }
    
    // Update session sequence number
    session->sequence_number = secure_msg.header.sequence_number;
    
    return true;
}
```

**Security Protocol Features**:
- **Mutual Authentication**: Challenge-response protocol
- **Session Management**: Unique session keys and IDs
- **Encryption**: AES-CBC with random IVs
- **Integrity**: HMAC verification for all messages
- **Replay Protection**: Sequence number validation

## 🧪 **Practice Problems**

### **Problem 1: TPM PCR Analysis**

**Scenario**: Analyze TPM PCR values after a secure boot sequence.

**PCR Values After Boot**:
- PCR 0: Bootloader hash
- PCR 1: Application firmware hash
- PCR 2: Boot success indicator
- PCR 3-7: Platform configuration

**Question**: How would you verify the system integrity using these PCR values?

**Expected Solution**:
```
1. PCR Validation:
   - Compare PCR 0 with expected bootloader hash
   - Verify PCR 1 matches firmware hash
   - Confirm PCR 2 indicates successful boot

2. Attestation:
   - Sign PCR values with TPM identity key
   - Send signed PCRs to remote verifier
   - Remote verifier compares with known good values

3. Integrity Check:
   - Any change in firmware changes PCR values
   - Compromised system will have different PCRs
   - Remote verification can detect tampering
```

### **Problem 2: Side-Channel Attack Analysis**

**Scenario**: Analyze a vulnerable cryptographic implementation.

**Vulnerable Code**:
```c
bool check_password(const char* input, const char* correct) {
    for (int i = 0; input[i] != '\0' && correct[i] != '\0'; i++) {
        if (input[i] != correct[i]) {
            return false;  // Early exit reveals password length
        }
    }
    return (input[i] == '\0' && correct[i] == '\0');
}
```

**Question**: What side-channel vulnerabilities exist and how would you fix them?

**Expected Analysis**:
```
Vulnerabilities:
1. Timing attack: Early exit reveals password length
2. Power analysis: Different execution paths
3. Cache timing: Memory access patterns

Fixes:
1. Constant-time comparison
2. Always iterate through full password
3. Use secure comparison functions
4. Add random delays (not recommended)
```

### **Problem 3: Secure Communication Design**

**Scenario**: Design security for an IoT device network.

**Requirements**:
- 100 IoT devices
- Central gateway
- Wireless communication
- Secure firmware updates

**Expected Solution**:
```
1. Device Authentication:
   - Unique device certificates
   - Certificate validation chain
   - Device registration protocol

2. Communication Security:
   - TLS 1.3 for all connections
   - Certificate pinning
   - Secure key exchange

3. Firmware Updates:
   - Signed firmware packages
   - Secure boot verification
   - Rollback protection
   - Update authentication

4. Network Security:
   - Device isolation
   - Traffic monitoring
   - Intrusion detection
```

## ✅ **Self-Assessment Checklist**

### **Secure Boot** ✅
- [ ] Can implement chain of trust
- [ ] Understand TPM integration
- [ ] Can prevent rollback attacks
- [ ] Know signature verification

### **Cryptography** ✅
- [ ] Can implement constant-time operations
- [ ] Understand side-channel attacks
- [ ] Can design secure protocols
- [ ] Know key management principles

### **Platform Security** ✅
- [ ] Can use ARM TrustZone
- [ ] Understand secure enclaves
- [ ] Can implement memory protection
- [ ] Know secure communication

### **Security Testing** ✅
- [ ] Can perform security audits
- [ ] Understand attack vectors
- [ ] Can implement countermeasures
- [ ] Know security best practices

## 🔗 **Related Topics**
- [Security Fundamentals](../Embedded_Security/Security_Fundamentals.md)
- [Secure Boot and Chain of Trust](../Embedded_Security/Secure_Boot_Chain_Trust.md)
- [TPM 2.0 Basics](../Embedded_Security/TPM2_Basics.md)
- [Cryptographic Foundations](../Embedded_Security/Cryptographic_Foundations.md)
- [Platform Security](../Embedded_Security/Platform_Security.md)

## 📚 **Additional Resources**
- **Books**: "Applied Cryptography" by Bruce Schneier
- **Online**: [ARM TrustZone Documentation](https://developer.arm.com/ip-products/security-ip/trustzone)
- **Practice**: [TPM 2.0 Tools](https://github.com/tpm2-software/tpm2-tools)
- **Standards**: [TPM 2.0 Specification](https://trustedcomputinggroup.org/resource/tpm-library-specification/)
