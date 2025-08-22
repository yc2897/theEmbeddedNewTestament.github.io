# 🎯 Problem-Solving Framework

## 🚀 **Quick Navigation**
- [Problem Analysis](#problem-analysis)
- [Solution Strategies](#solution-strategies)
- [Implementation Techniques](#implementation-techniques)
- [Testing and Validation](#testing-and-validation)

## 📚 **Quick Reference: Key Concepts**
- **Problem Decomposition**: Break complex problems into manageable parts
- **Algorithm Selection**: Choose appropriate algorithms for different problem types
- **Optimization Techniques**: Improve efficiency and performance
- **Error Handling**: Anticipate and handle edge cases and failures
- **Validation Methods**: Verify correctness and robustness

## 🎯 **Problem Analysis Framework**

### **Step 1: Understand the Problem**

**Key Questions to Ask**:
```
1. What is the problem asking for?
   - Input: What data/parameters are given?
   - Output: What result is expected?
   - Constraints: What limitations exist?

2. What are the requirements?
   - Functional requirements
   - Performance requirements
   - Resource constraints
   - Quality requirements

3. What assumptions can I make?
   - Data types and ranges
   - System constraints
   - Performance expectations
   - Error handling requirements
```

**Example Problem Analysis**:
```
Problem: "Design a sensor data collection system that can handle 1000 samples per second"

Analysis:
- Input: Sensor data at 1kHz rate
- Output: Collected and processed sensor data
- Constraints: Real-time requirements, memory limitations
- Requirements: Handle 1000 samples/sec, process data, store results
- Assumptions: Single sensor, fixed data format, real-time constraints
```

### **Step 2: Break Down the Problem**

**Problem Decomposition Techniques**:
```
1. Functional Decomposition
   - Data acquisition
   - Data processing
   - Data storage
   - Data transmission

2. Temporal Decomposition
   - Initialization phase
   - Main processing loop
   - Cleanup phase

3. Component Decomposition
   - Hardware interface
   - Software modules
   - Communication protocols
   - Storage systems
```

**Example Decomposition**:
```
Sensor Data Collection System:
├── Data Acquisition
│   ├── Sensor interface
│   ├── Sampling control
│   └── Data validation
├── Data Processing
│   ├── Filtering
│   ├── Calibration
│   └── Aggregation
├── Data Storage
│   ├── Buffer management
│   ├── Memory allocation
│   └── Overflow handling
└── Data Transmission
    ├── Protocol implementation
    ├── Error handling
    └── Retry mechanisms
```

### **Step 3: Identify Constraints and Requirements**

**Constraint Categories**:
```
1. Performance Constraints
   - Timing requirements
   - Throughput needs
   - Latency limits
   - Resource usage

2. Resource Constraints
   - Memory limitations
   - Processing power
   - Power consumption
   - Hardware capabilities

3. Quality Constraints
   - Reliability requirements
   - Accuracy needs
   - Error tolerance
   - Safety requirements
```

**Example Constraint Analysis**:
```
Performance: 1000 samples/sec = 1ms per sample
Memory: Limited to 64KB RAM
Power: Battery operation, low power required
Quality: 99.9% data integrity, <1% error rate
```

## 🎯 **Solution Strategies**

### **Strategy 1: Algorithm Selection**

**Common Algorithm Patterns**:
```
1. Data Structures
   - Arrays: Sequential access, fixed size
   - Linked Lists: Dynamic size, flexible insertion
   - Stacks: LIFO operations, function calls
   - Queues: FIFO operations, data buffering
   - Trees: Hierarchical data, searching
   - Hash Tables: Fast lookup, key-value pairs

2. Search Algorithms
   - Linear Search: O(n), simple, any data
   - Binary Search: O(log n), sorted data
   - Hash Search: O(1), hash table lookup

3. Sort Algorithms
   - Bubble Sort: O(n²), simple, small data
   - Quick Sort: O(n log n), efficient, general purpose
   - Merge Sort: O(n log n), stable, predictable
   - Heap Sort: O(n log n), in-place, heap structure
```

**Algorithm Selection Criteria**:
```
1. Time Complexity
   - O(1): Constant time, ideal
   - O(log n): Logarithmic, very good
   - O(n): Linear, good
   - O(n log n): Linearithmic, acceptable
   - O(n²): Quadratic, avoid for large data
   - O(2ⁿ): Exponential, avoid

2. Space Complexity
   - In-place: Minimal extra memory
   - Linear: O(n) extra memory
   - Quadratic: O(n²) extra memory

3. Stability
   - Stable: Maintains relative order
   - Unstable: May change relative order
```

### **Strategy 2: Design Patterns**

**Common Design Patterns**:
```
1. Creational Patterns
   - Singleton: Single instance
   - Factory: Object creation
   - Builder: Complex object construction

2. Structural Patterns
   - Adapter: Interface compatibility
   - Bridge: Implementation abstraction
   - Composite: Tree structures
   - Decorator: Dynamic behavior addition

3. Behavioral Patterns
   - Observer: Event notification
   - Strategy: Algorithm selection
   - State: Object state management
   - Template: Algorithm skeleton
```

**Pattern Selection Guidelines**:
```
1. Problem Type
   - Object creation: Factory, Builder
   - Interface mismatch: Adapter
   - Event handling: Observer
   - Algorithm variation: Strategy

2. System Constraints
   - Memory: Lightweight patterns
   - Performance: Efficient patterns
   - Maintainability: Clear patterns
   - Flexibility: Extensible patterns
```

### **Strategy 3: Optimization Techniques**

**Performance Optimization**:
```
1. Algorithm Optimization
   - Choose better algorithms
   - Reduce unnecessary operations
   - Use appropriate data structures
   - Implement caching strategies

2. Memory Optimization
   - Minimize allocations
   - Use memory pools
   - Implement garbage collection
   - Optimize data layout

3. Code Optimization
   - Compiler optimizations
   - Loop unrolling
   - Function inlining
   - Branch prediction
```

**Power Optimization**:
```
1. Sleep Mode Management
   - Deep sleep when idle
   - Wake on events
   - Duty cycling
   - Dynamic frequency scaling

2. Hardware Optimization
   - Disable unused peripherals
   - Use low-power modes
   - Optimize clock frequencies
   - Minimize I/O operations
```

## 🎯 **Implementation Techniques**

### **Technique 1: Incremental Development**

**Development Approach**:
```
1. Start Simple
   - Basic functionality first
   - Minimal features
   - Simple algorithms
   - Basic error handling

2. Add Complexity
   - Enhance functionality
   - Improve algorithms
   - Add error handling
   - Optimize performance

3. Refine and Polish
   - Edge case handling
   - Performance tuning
   - Code cleanup
   - Documentation
```

**Example Implementation**:
```c
// Version 1: Basic functionality
int process_sensor_data(int sensor_value) {
    return sensor_value * 2;  // Simple doubling
}

// Version 2: Add validation
int process_sensor_data(int sensor_value) {
    if (sensor_value < 0 || sensor_value > 1000) {
        return -1;  // Error indicator
    }
    return sensor_value * 2;
}

// Version 3: Add calibration
int process_sensor_data(int sensor_value) {
    if (sensor_value < 0 || sensor_value > 1000) {
        return -1;
    }
    
    // Apply calibration
    int calibrated_value = sensor_value + CALIBRATION_OFFSET;
    return calibrated_value * CALIBRATION_SCALE;
}

// Version 4: Add filtering
int process_sensor_data(int sensor_value) {
    if (sensor_value < 0 || sensor_value > 1000) {
        return -1;
    }
    
    // Apply moving average filter
    static int filter_buffer[FILTER_SIZE];
    static int filter_index = 0;
    static int filter_sum = 0;
    
    filter_sum -= filter_buffer[filter_index];
    filter_buffer[filter_index] = sensor_value;
    filter_sum += sensor_value;
    filter_index = (filter_index + 1) % FILTER_SIZE;
    
    int filtered_value = filter_sum / FILTER_SIZE;
    
    // Apply calibration
    int calibrated_value = filtered_value + CALIBRATION_OFFSET;
    return calibrated_value * CALIBRATION_SCALE;
}
```

### **Technique 2: Error Handling**

**Error Handling Strategies**:
```
1. Defensive Programming
   - Input validation
   - Boundary checking
   - Null pointer checks
   - Resource validation

2. Graceful Degradation
   - Fallback mechanisms
   - Default values
   - Reduced functionality
   - User notification

3. Error Recovery
   - Retry mechanisms
   - Reset procedures
   - Alternative paths
   - System restart
```

**Error Handling Implementation**:
```c
typedef enum {
    ERROR_NONE = 0,
    ERROR_INVALID_INPUT,
    ERROR_MEMORY_ALLOCATION,
    ERROR_HARDWARE_FAILURE,
    ERROR_TIMEOUT,
    ERROR_COMMUNICATION
} error_code_t;

typedef struct {
    error_code_t code;
    char message[128];
    uint32_t timestamp;
    uint32_t retry_count;
} error_info_t;

// Error handling function
error_code_t process_sensor_data_robust(int sensor_value, int *result) {
    // Input validation
    if (result == NULL) {
        return ERROR_INVALID_INPUT;
    }
    
    if (sensor_value < 0 || sensor_value > 1000) {
        return ERROR_INVALID_INPUT;
    }
    
    // Memory allocation with error handling
    int *temp_buffer = malloc(1024);
    if (temp_buffer == NULL) {
        return ERROR_MEMORY_ALLOCATION;
    }
    
    // Process data with timeout protection
    uint32_t start_time = get_system_time();
    bool success = false;
    
    while (!success && (get_system_time() - start_time) < TIMEOUT_MS) {
        success = try_process_data(sensor_value, temp_buffer, result);
        if (!success) {
            delay_ms(10);  // Brief delay before retry
        }
    }
    
    free(temp_buffer);
    
    if (!success) {
        return ERROR_TIMEOUT;
    }
    
    return ERROR_NONE;
}

// Error recovery function
bool handle_error(error_code_t error, error_info_t *error_info) {
    switch (error) {
        case ERROR_MEMORY_ALLOCATION:
            // Try to free memory and retry
            cleanup_memory();
            return true;
            
        case ERROR_HARDWARE_FAILURE:
            // Reset hardware and retry
            reset_hardware();
            return true;
            
        case ERROR_TIMEOUT:
            // Increase timeout and retry
            if (error_info->retry_count < MAX_RETRIES) {
                error_info->retry_count++;
                return true;
            }
            break;
            
        default:
            break;
    }
    
    return false;
}
```

### **Technique 3: Testing and Validation**

**Testing Strategies**:
```
1. Unit Testing
   - Individual function testing
   - Input validation testing
   - Edge case testing
   - Error condition testing

2. Integration Testing
   - Module interaction testing
   - Interface testing
   - Data flow testing
   - System behavior testing

3. Performance Testing
   - Timing measurements
   - Memory usage testing
   - Throughput testing
   - Stress testing
```

**Testing Implementation**:
```c
// Test framework
typedef struct {
    char test_name[64];
    bool (*test_function)(void);
    bool passed;
    uint32_t execution_time;
} test_case_t;

// Test case example
bool test_sensor_data_processing(void) {
    // Test normal case
    int result;
    error_code_t error = process_sensor_data_robust(500, &result);
    if (error != ERROR_NONE || result != 1000) {
        return false;
    }
    
    // Test boundary case
    error = process_sensor_data_robust(0, &result);
    if (error != ERROR_NONE || result != 0) {
        return false;
    }
    
    // Test error case
    error = process_sensor_data_robust(-1, &result);
    if (error != ERROR_INVALID_INPUT) {
        return false;
    }
    
    // Test null pointer
    error = process_sensor_data_robust(100, NULL);
    if (error != ERROR_INVALID_INPUT) {
        return false;
    }
    
    return true;
}

// Test runner
void run_tests(test_case_t *tests, int test_count) {
    printf("Running %d tests...\n", test_count);
    
    int passed = 0;
    uint32_t total_time = 0;
    
    for (int i = 0; i < test_count; i++) {
        uint32_t start_time = get_system_time();
        
        tests[i].passed = tests[i].test_function();
        tests[i].execution_time = get_system_time() - start_time;
        
        if (tests[i].passed) {
            passed++;
        }
        
        total_time += tests[i].execution_time;
        
        printf("Test %s: %s (%lu ms)\n", 
               tests[i].test_name,
               tests[i].passed ? "PASS" : "FAIL",
               tests[i].execution_time);
    }
    
    printf("\nTest Results: %d/%d passed, Total time: %lu ms\n", 
           passed, test_count, total_time);
}
```

## 🧪 **Practice Problems and Solutions**

### **Problem 1: Circular Buffer Implementation**

**Problem Statement**: Implement a thread-safe circular buffer for storing sensor data.

**Problem Analysis**:
```
Input: Sensor data values
Output: Circular buffer with push/pop operations
Constraints: Fixed size, thread-safe, real-time performance
Requirements: Handle overflow, underflow, multiple producers/consumers
```

**Solution Design**:
```c
typedef struct {
    int *buffer;
    size_t capacity;
    size_t head;
    size_t tail;
    size_t count;
    pthread_mutex_t mutex;
    pthread_cond_t not_empty;
    pthread_cond_t not_full;
} circular_buffer_t;

// Initialize circular buffer
bool circular_buffer_init(circular_buffer_t *cb, size_t capacity) {
    if (!cb || capacity == 0) return false;
    
    cb->buffer = malloc(capacity * sizeof(int));
    if (!cb->buffer) return false;
    
    cb->capacity = capacity;
    cb->head = 0;
    cb->tail = 0;
    cb->count = 0;
    
    pthread_mutex_init(&cb->mutex, NULL);
    pthread_cond_init(&cb->not_empty, NULL);
    pthread_cond_init(&cb->not_full, NULL);
    
    return true;
}

// Push data to buffer (producer)
bool circular_buffer_push(circular_buffer_t *cb, int value, bool wait) {
    if (!cb) return false;
    
    pthread_mutex_lock(&cb->mutex);
    
    // Wait if buffer is full
    while (cb->count >= cb->capacity && wait) {
        pthread_cond_wait(&cb->not_full, &cb->mutex);
    }
    
    // Check if buffer is full
    if (cb->count >= cb->capacity) {
        pthread_mutex_unlock(&cb->mutex);
        return false;  // Buffer full
    }
    
    // Add data to buffer
    cb->buffer[cb->head] = value;
    cb->head = (cb->head + 1) % cb->capacity;
    cb->count++;
    
    // Signal consumer
    pthread_cond_signal(&cb->not_empty);
    
    pthread_mutex_unlock(&cb->mutex);
    return true;
}

// Pop data from buffer (consumer)
bool circular_buffer_pop(circular_buffer_t *cb, int *value, bool wait) {
    if (!cb || !value) return false;
    
    pthread_mutex_lock(&cb->mutex);
    
    // Wait if buffer is empty
    while (cb->count == 0 && wait) {
        pthread_cond_wait(&cb->not_empty, &cb->mutex);
    }
    
    // Check if buffer is empty
    if (cb->count == 0) {
        pthread_mutex_unlock(&cb->mutex);
        return false;  // Buffer empty
    }
    
    // Remove data from buffer
    *value = cb->buffer[cb->tail];
    cb->tail = (cb->tail + 1) % cb->capacity;
    cb->count--;
    
    // Signal producer
    pthread_cond_signal(&cb->not_full);
    
    pthread_mutex_unlock(&cb->mutex);
    return true;
}
```

### **Problem 2: Real-Time Task Scheduler**

**Problem Statement**: Design a simple real-time task scheduler for embedded systems.

**Problem Analysis**:
```
Input: Task definitions with priorities and periods
Output: Task execution schedule
Constraints: Limited memory, real-time requirements
Requirements: Priority-based scheduling, deadline handling
```

**Solution Design**:
```c
typedef struct {
    uint32_t task_id;
    void (*function)(void*);
    void *data;
    uint32_t priority;
    uint32_t period;
    uint32_t deadline;
    uint32_t last_execution;
    uint32_t next_execution;
    bool active;
} rt_task_t;

typedef struct {
    rt_task_t *tasks;
    size_t task_count;
    size_t max_tasks;
    uint32_t current_time;
} rt_scheduler_t;

// Initialize scheduler
bool rt_scheduler_init(rt_scheduler_t *scheduler, size_t max_tasks) {
    if (!scheduler || max_tasks == 0) return false;
    
    scheduler->tasks = malloc(max_tasks * sizeof(rt_task_t));
    if (!scheduler->tasks) return false;
    
    scheduler->max_tasks = max_tasks;
    scheduler->task_count = 0;
    scheduler->current_time = 0;
    
    return true;
}

// Add task to scheduler
bool rt_scheduler_add_task(rt_scheduler_t *scheduler, rt_task_t *task) {
    if (!scheduler || !task || scheduler->task_count >= scheduler->max_tasks) {
        return false;
    }
    
    // Copy task
    scheduler->tasks[scheduler->task_count] = *task;
    scheduler->tasks[scheduler->task_count].active = true;
    scheduler->tasks[scheduler->task_count].next_execution = scheduler->current_time;
    
    scheduler->task_count++;
    return true;
}

// Find highest priority ready task
rt_task_t* find_highest_priority_task(rt_scheduler_t *scheduler) {
    rt_task_t *highest_priority_task = NULL;
    uint32_t highest_priority = 0;
    
    for (size_t i = 0; i < scheduler->task_count; i++) {
        rt_task_t *task = &scheduler->tasks[i];
        
        if (task->active && 
            scheduler->current_time >= task->next_execution &&
            task->priority > highest_priority) {
            
            highest_priority = task->priority;
            highest_priority_task = task;
        }
    }
    
    return highest_priority_task;
}

// Run scheduler
void rt_scheduler_run(rt_scheduler_t *scheduler) {
    while (1) {
        // Find highest priority ready task
        rt_task_t *task = find_highest_priority_task(scheduler);
        
        if (task) {
            // Execute task
            task->function(task->data);
            
            // Update task timing
            task->last_execution = scheduler->current_time;
            task->next_execution = scheduler->current_time + task->period;
            
            // Check deadline
            if (scheduler->current_time > task->deadline) {
                // Deadline missed - handle appropriately
                handle_deadline_miss(task);
            }
        }
        
        // Update time
        scheduler->current_time++;
        
        // Small delay to prevent busy waiting
        delay_ms(1);
    }
}
```

## ✅ **Self-Assessment Checklist**

### **Problem Analysis** ✅
- [ ] Can break down complex problems
- [ ] Can identify requirements and constraints
- [ ] Can make appropriate assumptions
- [ ] Can validate problem understanding

### **Solution Design** ✅
- [ ] Can select appropriate algorithms
- [ ] Can apply design patterns
- [ ] Can consider trade-offs
- [ ] Can plan implementation steps

### **Implementation** ✅
- [ ] Can write clean, readable code
- [ ] Can handle errors gracefully
- [ ] Can implement incrementally
- [ ] Can optimize performance

### **Testing and Validation** ✅
- [ ] Can write test cases
- [ ] Can validate solutions
- [ ] Can handle edge cases
- [ ] Can measure performance

## 🔗 **Related Topics**
- [Technical Interview Guide](./Technical_Interview_Guide.md)
- [Mock Interviews](./Mock_Interviews.md)
- [C Programming Interview](../Foundation_Level/C_Programming_Interview.md)
- [System Integration Interview](../Intermediate_Level/System_Integration_Interview.md)

## 📚 **Additional Resources**
- **Books**: "Problem Solving with Algorithms and Data Structures" by Brad Miller
- **Online**: [LeetCode Problem-Solving](https://leetcode.com/)
- **Practice**: [HackerRank Algorithms](https://www.hackerrank.com/domains/algorithms)
- **Communities**: [Stack Overflow Problem Solving](https://stackoverflow.com/questions/tagged/algorithm)
