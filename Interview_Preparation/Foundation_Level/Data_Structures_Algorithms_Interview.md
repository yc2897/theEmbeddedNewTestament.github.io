# 🧮 Data Structures & Algorithms Interview Preparation

## 🚀 **Quick Navigation**
- [Data Structures Fundamentals](#data-structures-fundamentals)
- [Algorithm Analysis](#algorithm-analysis)
- [Common Algorithms](#common-algorithms)
- [Embedded-Specific Considerations](#embedded-specific-considerations)
- [Optimization Techniques](#optimization-techniques)

## 📚 **Quick Reference: Key Concepts**
- **Data Structures**: Arrays, linked lists, stacks, queues, trees, hash tables
- **Algorithm Complexity**: Time complexity (O notation), space complexity
- **Search Algorithms**: Linear, binary, hash-based searching
- **Sort Algorithms**: Bubble, insertion, selection, merge, quick, heap sort
- **Memory Efficiency**: Cache-friendly structures, minimal memory usage

## 🧮 **Data Structures Fundamentals**

### **Arrays and Memory Layout**

**Array Implementation**:
```c
// Static array
int static_array[100];

// Dynamic array
typedef struct {
    int *data;
    size_t size;
    size_t capacity;
} dynamic_array_t;

// Initialize dynamic array
bool init_dynamic_array(dynamic_array_t *arr, size_t initial_capacity) {
    if (!arr || initial_capacity == 0) return false;
    
    arr->data = malloc(initial_capacity * sizeof(int));
    if (!arr->data) return false;
    
    arr->size = 0;
    arr->capacity = initial_capacity;
    return true;
}

// Add element to dynamic array
bool add_element(dynamic_array_t *arr, int value) {
    if (!arr) return false;
    
    // Check if resize needed
    if (arr->size >= arr->capacity) {
        size_t new_capacity = arr->capacity * 2;
        int *new_data = realloc(arr->data, new_capacity * sizeof(int));
        
        if (!new_data) return false;
        
        arr->data = new_data;
        arr->capacity = new_capacity;
    }
    
    // Add element
    arr->data[arr->size] = value;
    arr->size++;
    
    return true;
}

// Cache-friendly array access (row-major order)
void matrix_multiply_cache_friendly(float *A, float *B, float *C, int N) {
    for (int i = 0; i < N; i++) {
        for (int j = 0; j < N; j++) {
            float sum = 0.0f;
            for (int k = 0; k < N; k++) {
                // Access elements in row-major order for better cache performance
                sum += A[i * N + k] * B[k * N + j];
            }
            C[i * N + j] = sum;
        }
    }
}
```

**Memory Layout Considerations**:
```
1. Cache Line Size
   - Typical cache line: 64 bytes
   - Align data structures to cache lines
   - Minimize cache misses

2. Memory Access Patterns
   - Sequential access is faster than random access
   - Row-major vs. column-major order
   - Structure of arrays vs. array of structures

3. Alignment
   - Natural alignment for performance
   - Padding considerations
   - Memory waste vs. performance trade-off
```

### **Linked Lists**

**Singly Linked List Implementation**:
```c
typedef struct node {
    int data;
    struct node *next;
} node_t;

typedef struct {
    node_t *head;
    node_t *tail;
    size_t size;
} linked_list_t;

// Create new node
node_t* create_node(int data) {
    node_t *new_node = malloc(sizeof(node_t));
    if (new_node) {
        new_node->data = data;
        new_node->next = NULL;
    }
    return new_node;
}

// Insert at beginning
bool insert_at_beginning(linked_list_t *list, int data) {
    if (!list) return false;
    
    node_t *new_node = create_node(data);
    if (!new_node) return false;
    
    new_node->next = list->head;
    list->head = new_node;
    
    if (!list->tail) {
        list->tail = new_node;
    }
    
    list->size++;
    return true;
}

// Insert at end
bool insert_at_end(linked_list_t *list, int data) {
    if (!list) return false;
    
    node_t *new_node = create_node(data);
    if (!new_node) return false;
    
    if (!list->head) {
        list->head = new_node;
        list->tail = new_node;
    } else {
        list->tail->next = new_node;
        list->tail = new_node;
    }
    
    list->size++;
    return true;
}

// Find element
node_t* find_element(linked_list_t *list, int data) {
    if (!list) return NULL;
    
    node_t *current = list->head;
    while (current) {
        if (current->data == data) {
            return current;
        }
        current = current->next;
    }
    
    return NULL;
}

// Delete element
bool delete_element(linked_list_t *list, int data) {
    if (!list || !list->head) return false;
    
    node_t *current = list->head;
    node_t *prev = NULL;
    
    // Find element to delete
    while (current && current->data != data) {
        prev = current;
        current = current->next;
    }
    
    if (!current) return false;  // Element not found
    
    // Update list structure
    if (prev) {
        prev->next = current->next;
        if (!prev->next) {
            list->tail = prev;
        }
    } else {
        list->head = current->next;
        if (!list->head) {
            list->tail = NULL;
        }
    }
    
    // Free memory
    free(current);
    list->size--;
    
    return true;
}
```

**Doubly Linked List**:
```c
typedef struct dnode {
    int data;
    struct dnode *prev;
    struct dnode *next;
} dnode_t;

typedef struct {
    dnode_t *head;
    dnode_t *tail;
    size_t size;
} doubly_linked_list_t;

// Insert in sorted order (for ordered list)
bool insert_sorted(doubly_linked_list_t *list, int data) {
    if (!list) return false;
    
    dnode_t *new_node = malloc(sizeof(dnode_t));
    if (!new_node) return false;
    
    new_node->data = data;
    
    // Empty list
    if (!list->head) {
        new_node->prev = NULL;
        new_node->next = NULL;
        list->head = new_node;
        list->tail = new_node;
        list->size++;
        return true;
    }
    
    // Find insertion position
    dnode_t *current = list->head;
    while (current && current->data < data) {
        current = current->next;
    }
    
    // Insert before current
    if (current) {
        new_node->next = current;
        new_node->prev = current->prev;
        
        if (current->prev) {
            current->prev->next = new_node;
        } else {
            list->head = new_node;
        }
        
        current->prev = new_node;
    } else {
        // Insert at end
        new_node->next = NULL;
        new_node->prev = list->tail;
        list->tail->next = new_node;
        list->tail = new_node;
    }
    
    list->size++;
    return true;
}
```

### **Stacks and Queues**

**Stack Implementation**:
```c
typedef struct {
    int *data;
    size_t top;
    size_t capacity;
} stack_t;

// Initialize stack
bool init_stack(stack_t *stack, size_t capacity) {
    if (!stack || capacity == 0) return false;
    
    stack->data = malloc(capacity * sizeof(int));
    if (!stack->data) return false;
    
    stack->top = 0;
    stack->capacity = capacity;
    return true;
}

// Push element
bool push(stack_t *stack, int value) {
    if (!stack || stack->top >= stack->capacity) return false;
    
    stack->data[stack->top++] = value;
    return true;
}

// Pop element
bool pop(stack_t *stack, int *value) {
    if (!stack || !value || stack->top == 0) return false;
    
    *value = stack->data[--stack->top];
    return true;
}

// Peek top element
bool peek(stack_t *stack, int *value) {
    if (!stack || !value || stack->top == 0) return false;
    
    *value = stack->data[stack->top - 1];
    return true;
}

// Check if stack is empty
bool is_stack_empty(stack_t *stack) {
    return (!stack || stack->top == 0);
}

// Check if stack is full
bool is_stack_full(stack_t *stack) {
    return (stack && stack->top >= stack->capacity);
}
```

**Queue Implementation**:
```c
typedef struct {
    int *data;
    size_t front;
    size_t rear;
    size_t size;
    size_t capacity;
} queue_t;

// Initialize queue
bool init_queue(queue_t *queue, size_t capacity) {
    if (!queue || capacity == 0) return false;
    
    queue->data = malloc(capacity * sizeof(int));
    if (!queue->data) return false;
    
    queue->front = 0;
    queue->rear = 0;
    queue->size = 0;
    queue->capacity = capacity;
    return true;
}

// Enqueue element
bool enqueue(queue_t *queue, int value) {
    if (!queue || queue->size >= queue->capacity) return false;
    
    queue->data[queue->rear] = value;
    queue->rear = (queue->rear + 1) % queue->capacity;
    queue->size++;
    
    return true;
}

// Dequeue element
bool dequeue(queue_t *queue, int *value) {
    if (!queue || !value || queue->size == 0) return false;
    
    *value = queue->data[queue->front];
    queue->front = (queue->front + 1) % queue->capacity;
    queue->size--;
    
    return true;
}

// Peek front element
bool peek_front(queue_t *queue, int *value) {
    if (!queue || !value || queue->size == 0) return false;
    
    *value = queue->data[queue->front];
    return true;
}

// Check if queue is empty
bool is_queue_empty(queue_t *queue) {
    return (!queue || queue->size == 0);
}

// Check if queue is full
bool is_queue_full(queue_t *queue) {
    return (queue && queue->size >= queue->capacity);
}
```

## 🧮 **Algorithm Analysis**

### **Time Complexity Analysis**

**Big O Notation Examples**:
```c
// O(1) - Constant time
int get_first_element(int arr[], int size) {
    if (size > 0) return arr[0];
    return -1;
}

// O(n) - Linear time
int find_element(int arr[], int size, int target) {
    for (int i = 0; i < size; i++) {
        if (arr[i] == target) return i;
    }
    return -1;
}

// O(n²) - Quadratic time
void bubble_sort(int arr[], int size) {
    for (int i = 0; i < size - 1; i++) {
        for (int j = 0; j < size - i - 1; j++) {
            if (arr[j] > arr[j + 1]) {
                // Swap elements
                int temp = arr[j];
                arr[j] = arr[j + 1];
                arr[j + 1] = temp;
            }
        }
    }
}

// O(log n) - Logarithmic time
int binary_search(int arr[], int size, int target) {
    int left = 0;
    int right = size - 1;
    
    while (left <= right) {
        int mid = left + (right - left) / 2;
        
        if (arr[mid] == target) return mid;
        if (arr[mid] < target) left = mid + 1;
        else right = mid - 1;
    }
    
    return -1;
}

// O(n log n) - Linearithmic time
void merge_sort(int arr[], int left, int right) {
    if (left < right) {
        int mid = left + (right - left) / 2;
        
        merge_sort(arr, left, mid);
        merge_sort(arr, mid + 1, right);
        merge(arr, left, mid, right);
    }
}
```

**Complexity Comparison**:
```
Algorithm          | Best Case | Average Case | Worst Case | Space
------------------|-----------|--------------|------------|-------
Linear Search     | O(1)      | O(n)         | O(n)       | O(1)
Binary Search     | O(1)      | O(log n)     | O(log n)   | O(1)
Bubble Sort       | O(n)      | O(n²)        | O(n²)      | O(1)
Insertion Sort    | O(n)      | O(n²)        | O(n²)      | O(1)
Merge Sort        | O(n log n)| O(n log n)   | O(n log n) | O(n)
Quick Sort        | O(n log n)| O(n log n)   | O(n²)      | O(log n)
Heap Sort         | O(n log n)| O(n log n)   | O(n log n) | O(1)
```

### **Space Complexity Analysis**

**Memory Usage Examples**:
```c
// O(1) - Constant space
int sum_array(int arr[], int size) {
    int sum = 0;
    for (int i = 0; i < size; i++) {
        sum += arr[i];
    }
    return sum;
}

// O(n) - Linear space
int* copy_array(int arr[], int size) {
    int *copy = malloc(size * sizeof(int));
    if (!copy) return NULL;
    
    for (int i = 0; i < size; i++) {
        copy[i] = arr[i];
    }
    return copy;
}

// O(n²) - Quadratic space
int** create_matrix(int rows, int cols) {
    int **matrix = malloc(rows * sizeof(int*));
    if (!matrix) return NULL;
    
    for (int i = 0; i < rows; i++) {
        matrix[i] = malloc(cols * sizeof(int));
        if (!matrix[i]) {
            // Clean up on failure
            for (int j = 0; j < i; j++) {
                free(matrix[j]);
            }
            free(matrix);
            return NULL;
        }
    }
    
    return matrix;
}

// Recursive space complexity
int factorial_recursive(int n) {
    if (n <= 1) return 1;
    return n * factorial_recursive(n - 1);
    // Space complexity: O(n) due to call stack
}

int factorial_iterative(int n) {
    int result = 1;
    for (int i = 2; i <= n; i++) {
        result *= i;
    }
    return result;
    // Space complexity: O(1)
}
```

## 🧮 **Common Algorithms**

### **Search Algorithms**

**Linear Search with Optimizations**:
```c
// Basic linear search
int linear_search(int arr[], int size, int target) {
    for (int i = 0; i < size; i++) {
        if (arr[i] == target) return i;
    }
    return -1;
}

// Linear search with sentinel (eliminates boundary check)
int linear_search_sentinel(int arr[], int size, int target) {
    // Store original last element
    int last = arr[size - 1];
    
    // Set sentinel
    arr[size - 1] = target;
    
    int i = 0;
    while (arr[i] != target) {
        i++;
    }
    
    // Restore original element
    arr[size - 1] = last;
    
    // Check if target was found
    if (i < size - 1 || last == target) {
        return i;
    }
    
    return -1;
}

// Linear search with early termination
int linear_search_early_termination(int arr[], int size, int target) {
    for (int i = 0; i < size; i++) {
        if (arr[i] == target) return i;
        
        // Early termination if we've passed where target should be
        if (arr[i] > target) break;
    }
    return -1;
}
```

**Binary Search Variations**:
```c
// Standard binary search
int binary_search(int arr[], int size, int target) {
    int left = 0;
    int right = size - 1;
    
    while (left <= right) {
        int mid = left + (right - left) / 2;
        
        if (arr[mid] == target) return mid;
        if (arr[mid] < target) left = mid + 1;
        else right = mid - 1;
    }
    
    return -1;
}

// Binary search with first occurrence
int binary_search_first(int arr[], int size, int target) {
    int left = 0;
    int right = size - 1;
    int result = -1;
    
    while (left <= right) {
        int mid = left + (right - left) / 2;
        
        if (arr[mid] == target) {
            result = mid;
            right = mid - 1;  // Continue searching left
        } else if (arr[mid] < target) {
            left = mid + 1;
        } else {
            right = mid - 1;
        }
    }
    
    return result;
}

// Binary search with last occurrence
int binary_search_last(int arr[], int size, int target) {
    int left = 0;
    int right = size - 1;
    int result = -1;
    
    while (left <= right) {
        int mid = left + (right - left) / 2;
        
        if (arr[mid] == target) {
            result = mid;
            left = mid + 1;  // Continue searching right
        } else if (arr[mid] < target) {
            left = mid + 1;
        } else {
            right = mid - 1;
        }
    }
    
    return result;
}

// Binary search for insertion point
int binary_search_insertion_point(int arr[], int size, int target) {
    int left = 0;
    int right = size;
    
    while (left < right) {
        int mid = left + (right - left) / 2;
        
        if (arr[mid] < target) {
            left = mid + 1;
        } else {
            right = mid;
        }
    }
    
    return left;
}
```

### **Sort Algorithms**

**Insertion Sort with Optimizations**:
```c
// Basic insertion sort
void insertion_sort(int arr[], int size) {
    for (int i = 1; i < size; i++) {
        int key = arr[i];
        int j = i - 1;
        
        while (j >= 0 && arr[j] > key) {
            arr[j + 1] = arr[j];
            j--;
        }
        
        arr[j + 1] = key;
    }
}

// Binary insertion sort (uses binary search for insertion point)
void binary_insertion_sort(int arr[], int size) {
    for (int i = 1; i < size; i++) {
        int key = arr[i];
        
        // Find insertion point using binary search
        int left = 0;
        int right = i;
        
        while (left < right) {
            int mid = left + (right - left) / 2;
            if (arr[mid] <= key) {
                left = mid + 1;
            } else {
                right = mid;
            }
        }
        
        // Shift elements to make room
        for (int j = i; j > left; j--) {
            arr[j] = arr[j - 1];
        }
        
        arr[left] = key;
    }
}

// Shell sort (improved insertion sort)
void shell_sort(int arr[], int size) {
    // Generate gap sequence
    for (int gap = size / 2; gap > 0; gap /= 2) {
        for (int i = gap; i < size; i++) {
            int key = arr[i];
            int j = i;
            
            while (j >= gap && arr[j - gap] > key) {
                arr[j] = arr[j - gap];
                j -= gap;
            }
            
            arr[j] = key;
        }
    }
}
```

**Quick Sort Implementation**:
```c
// Partition function
int partition(int arr[], int low, int high) {
    int pivot = arr[high];
    int i = low - 1;
    
    for (int j = low; j < high; j++) {
        if (arr[j] <= pivot) {
            i++;
            // Swap arr[i] and arr[j]
            int temp = arr[i];
            arr[i] = arr[j];
            arr[j] = temp;
        }
    }
    
    // Swap arr[i+1] and arr[high] (pivot)
    int temp = arr[i + 1];
    arr[i + 1] = arr[high];
    arr[high] = temp;
    
    return i + 1;
}

// Quick sort with optimization for small subarrays
void quick_sort(int arr[], int low, int high) {
    if (low < high) {
        // Use insertion sort for small subarrays
        if (high - low + 1 <= 10) {
            insertion_sort(&arr[low], high - low + 1);
            return;
        }
        
        int pi = partition(arr, low, high);
        
        quick_sort(arr, low, pi - 1);
        quick_sort(arr, pi + 1, high);
    }
}

// Quick sort with median-of-three pivot selection
int median_of_three(int arr[], int low, int high) {
    int mid = low + (high - low) / 2;
    
    // Sort low, mid, high
    if (arr[low] > arr[mid]) {
        int temp = arr[low];
        arr[low] = arr[mid];
        arr[mid] = temp;
    }
    if (arr[low] > arr[high]) {
        int temp = arr[low];
        arr[low] = arr[high];
        arr[high] = temp;
    }
    if (arr[mid] > arr[high]) {
        int temp = arr[mid];
        arr[mid] = arr[high];
        arr[high] = temp;
    }
    
    // Move pivot to high-1
    int temp = arr[mid];
    arr[mid] = arr[high - 1];
    arr[high - 1] = temp;
    
    return arr[high - 1];
}
```

## 🧮 **Embedded-Specific Considerations**

### **Memory Constraints**

**Memory-Efficient Data Structures**:
```c
// Compact linked list node (minimal overhead)
typedef struct compact_node {
    uint16_t data;           // 2 bytes
    uint16_t next_offset;    // 2 bytes (offset from start of pool)
} compact_node_t;

// Memory pool for compact nodes
typedef struct {
    compact_node_t *pool;
    uint16_t free_list_head;
    uint16_t pool_size;
} compact_list_pool_t;

// Initialize pool
bool init_compact_pool(compact_list_pool_t *pool, uint16_t size) {
    pool->pool = malloc(size * sizeof(compact_node_t));
    if (!pool->pool) return false;
    
    pool->pool_size = size;
    
    // Initialize free list
    for (uint16_t i = 0; i < size - 1; i++) {
        pool->pool[i].next_offset = i + 1;
    }
    pool->pool[size - 1].next_offset = 0xFFFF;  // End marker
    pool->free_list_head = 0;
    
    return true;
}

// Allocate node from pool
uint16_t allocate_node(compact_list_pool_t *pool) {
    if (pool->free_list_head == 0xFFFF) return 0xFFFF;  // Pool full
    
    uint16_t node_index = pool->free_list_head;
    pool->free_list_head = pool->pool[node_index].next_offset;
    
    return node_index;
}

// Free node back to pool
void free_node(compact_list_pool_t *pool, uint16_t node_index) {
    if (node_index >= pool->pool_size) return;
    
    pool->pool[node_index].next_offset = pool->free_list_head;
    pool->free_list_head = node_index;
}
```

**Cache-Optimized Data Structures**:
```c
// Structure of arrays (SoA) for better cache performance
typedef struct {
    float *temperatures;     // All temperatures together
    float *humidities;       // All humidities together
    uint32_t *timestamps;    // All timestamps together
    size_t size;
    size_t capacity;
} sensor_data_soa_t;

// Array of structures (AoS) - traditional approach
typedef struct {
    float temperature;
    float humidity;
    uint32_t timestamp;
} sensor_data_aos_t;

// SoA initialization
bool init_sensor_data_soa(sensor_data_soa_t *data, size_t capacity) {
    data->temperatures = malloc(capacity * sizeof(float));
    data->humidities = malloc(capacity * sizeof(float));
    data->timestamps = malloc(capacity * sizeof(uint32_t));
    
    if (!data->temperatures || !data->humidities || !data->timestamps) {
        // Clean up on failure
        free(data->temperatures);
        free(data->humidities);
        free(data->timestamps);
        return false;
    }
    
    data->size = 0;
    data->capacity = capacity;
    return true;
}

// Cache-friendly processing
void process_sensor_data_soa(sensor_data_soa_t *data) {
    // Process temperatures (sequential memory access)
    for (size_t i = 0; i < data->size; i++) {
        data->temperatures[i] = apply_calibration(data->temperatures[i]);
    }
    
    // Process humidities (sequential memory access)
    for (size_t i = 0; i < data->size; i++) {
        data->humidities[i] = apply_filter(data->humidities[i]);
    }
}
```

### **Real-Time Considerations**

**Deterministic Algorithms**:
```c
// Deterministic sorting with fixed time complexity
void deterministic_sort(int arr[], int size) {
    // Use counting sort for bounded integer values
    // Time complexity: O(n + k) where k is range of values
    // Space complexity: O(k)
    
    const int MAX_VALUE = 1000;  // Known upper bound
    int count[MAX_VALUE + 1] = {0};
    
    // Count occurrences
    for (int i = 0; i < size; i++) {
        count[arr[i]]++;
    }
    
    // Reconstruct sorted array
    int index = 0;
    for (int i = 0; i <= MAX_VALUE; i++) {
        while (count[i] > 0) {
            arr[index++] = i;
            count[i]--;
        }
    }
}

// Bounded search with timeout
int bounded_search(int arr[], int size, int target, uint32_t max_time_ms) {
    uint32_t start_time = get_system_time();
    
    for (int i = 0; i < size; i++) {
        // Check timeout
        if (get_system_time() - start_time > max_time_ms) {
            return -2;  // Timeout indicator
        }
        
        if (arr[i] == target) return i;
    }
    
    return -1;  // Not found
}
```

## 🧮 **Optimization Techniques**

### **Algorithm Optimization**

**Loop Optimization**:
```c
// Unroll loops for better performance
void matrix_multiply_unrolled(float *A, float *B, float *C, int N) {
    for (int i = 0; i < N; i++) {
        for (int j = 0; j < N; j += 4) {  // Process 4 elements at once
            float sum0 = 0.0f, sum1 = 0.0f, sum2 = 0.0f, sum3 = 0.0f;
            
            for (int k = 0; k < N; k++) {
                float a_val = A[i * N + k];
                sum0 += a_val * B[k * N + j];
                sum1 += a_val * B[k * N + j + 1];
                sum2 += a_val * B[k * N + j + 2];
                sum3 += a_val * B[k * N + j + 3];
            }
            
            C[i * N + j] = sum0;
            C[i * N + j + 1] = sum1;
            C[i * N + j + 2] = sum2;
            C[i * N + j + 3] = sum3;
        }
    }
}

// Cache-blocked matrix multiplication
void matrix_multiply_blocked(float *A, float *B, float *C, int N) {
    const int BLOCK_SIZE = 32;  // Optimize for cache size
    
    for (int i0 = 0; i0 < N; i0 += BLOCK_SIZE) {
        for (int j0 = 0; j0 < N; j0 += BLOCK_SIZE) {
            for (int k0 = 0; k0 < N; k0 += BLOCK_SIZE) {
                // Process block
                for (int i = i0; i < MIN(i0 + BLOCK_SIZE, N); i++) {
                    for (int j = j0; j < MIN(j0 + BLOCK_SIZE, N); j++) {
                        float sum = 0.0f;
                        for (int k = k0; k < MIN(k0 + BLOCK_SIZE, N); k++) {
                            sum += A[i * N + k] * B[k * N + j];
                        }
                        C[i * N + j] += sum;
                    }
                }
            }
        }
    }
}
```

**Memory Access Optimization**:
```c
// Prefetch data for better cache performance
void prefetch_optimized_processing(int *data, int size) {
    for (int i = 0; i < size; i++) {
        // Prefetch next cache line
        if (i + 16 < size) {
            __builtin_prefetch(&data[i + 16], 0, 3);  // Read, high locality
        }
        
        // Process current element
        process_element(data[i]);
    }
}

// Aligned memory allocation
void* aligned_allocate(size_t size, size_t alignment) {
    void *ptr = NULL;
    
    #ifdef _MSC_VER
        ptr = _aligned_malloc(size, alignment);
    #else
        if (posix_memalign(&ptr, alignment, size) != 0) {
            ptr = NULL;
        }
    #endif
    
    return ptr;
}

// SIMD-optimized operations
void vector_add_simd(float *a, float *b, float *result, int size) {
    // Ensure alignment for SIMD operations
    assert(((uintptr_t)a & 0xF) == 0);
    assert(((uintptr_t)b & 0xF) == 0);
    assert(((uintptr_t)result & 0xF) == 0);
    
    for (int i = 0; i < size; i += 4) {
        // Load vectors
        __m128 va = _mm_load_ps(&a[i]);
        __m128 vb = _mm_load_ps(&b[i]);
        
        // Add vectors
        __m128 sum = _mm_add_ps(va, vb);
        
        // Store result
        _mm_store_ps(&result[i], sum);
    }
}
```

## 🧪 **Common Interview Questions**

### **Question 1: Find Missing Number**

**Problem**: Given an array of n-1 integers in the range [1, n], find the missing number.

**Solution Approach**:
```
1. Mathematical approach: Sum of first n numbers minus array sum
2. XOR approach: XOR all numbers from 1 to n with array elements
3. Sorting approach: Sort and find gap
4. Hash table approach: Mark present numbers
```

**Optimal Solution**:
```c
// Mathematical approach - O(n) time, O(1) space
int find_missing_number_math(int arr[], int size) {
    int n = size + 1;
    int expected_sum = n * (n + 1) / 2;
    int actual_sum = 0;
    
    for (int i = 0; i < size; i++) {
        actual_sum += arr[i];
    }
    
    return expected_sum - actual_sum;
}

// XOR approach - O(n) time, O(1) space
int find_missing_number_xor(int arr[], int size) {
    int xor_result = 0;
    
    // XOR all numbers from 1 to n
    for (int i = 1; i <= size + 1; i++) {
        xor_result ^= i;
    }
    
    // XOR with array elements
    for (int i = 0; i < size; i++) {
        xor_result ^= arr[i];
    }
    
    return xor_result;
}
```

### **Question 2: Reverse Linked List**

**Problem**: Reverse a singly linked list.

**Solution Approach**:
```
1. Iterative approach: Use three pointers
2. Recursive approach: Recursively reverse rest of list
3. Stack approach: Push all nodes, then pop to reverse
```

**Iterative Solution**:
```c
node_t* reverse_linked_list_iterative(node_t *head) {
    node_t *prev = NULL;
    node_t *current = head;
    node_t *next = NULL;
    
    while (current != NULL) {
        // Store next node
        next = current->next;
        
        // Reverse current node's pointer
        current->next = prev;
        
        // Move pointers one position ahead
        prev = current;
        current = next;
    }
    
    return prev;  // New head
}

// Recursive solution
node_t* reverse_linked_list_recursive(node_t *head) {
    // Base case: empty list or single node
    if (head == NULL || head->next == NULL) {
        return head;
    }
    
    // Recursively reverse rest of list
    node_t *rest = reverse_linked_list_recursive(head->next);
    
    // Reverse current node
    head->next->next = head;
    head->next = NULL;
    
    return rest;
}
```

### **Question 3: Detect Cycle in Linked List**

**Problem**: Detect if a linked list has a cycle.

**Solution Approach**:
```
1. Hash table approach: Store visited nodes
2. Floyd's cycle-finding algorithm (tortoise and hare)
3. Marking approach: Mark visited nodes
```

**Floyd's Algorithm**:
```c
bool has_cycle_floyd(node_t *head) {
    if (!head || !head->next) return false;
    
    node_t *slow = head;
    node_t *fast = head->next;
    
    while (slow != fast) {
        if (!fast || !fast->next) return false;
        
        slow = slow->next;        // Move one step
        fast = fast->next->next;  // Move two steps
    }
    
    return true;  // Cycle detected
}

// Find cycle start node
node_t* find_cycle_start(node_t *head) {
    if (!head || !head->next) return NULL;
    
    // Find meeting point
    node_t *slow = head;
    node_t *fast = head;
    
    do {
        if (!fast || !fast->next) return NULL;
        slow = slow->next;
        fast = fast->next->next;
    } while (slow != fast);
    
    // Find cycle start
    slow = head;
    while (slow != fast) {
        slow = slow->next;
        fast = fast->next;
    }
    
    return slow;
}
```

## 🧪 **Practice Problems**

### **Problem 1: Efficient String Matching**

**Scenario**: Implement a function to find the first occurrence of a substring in a string.

**Question**: Design an efficient algorithm with O(n+m) time complexity.

**Expected Analysis**:
```
1. Algorithm Selection:
   - Naive approach: O(n*m) time
   - KMP algorithm: O(n+m) time, O(m) space
   - Boyer-Moore: O(n/m) average case

2. Implementation:
   - Preprocess pattern for KMP
   - Use failure function for efficient matching
   - Handle edge cases (empty strings, no match)

3. Optimization:
   - Early termination
   - Memory-efficient preprocessing
   - Cache-friendly access patterns
```

### **Problem 2: Memory-Efficient Queue**

**Scenario**: Design a queue that can handle millions of elements with minimal memory overhead.

**Question**: Implement a circular buffer with automatic resizing.

**Expected Analysis**:
```
1. Data Structure Design:
   - Circular buffer with dynamic sizing
   - Efficient memory allocation strategy
   - Minimal memory fragmentation

2. Performance Considerations:
   - O(1) enqueue/dequeue operations
   - Efficient resizing (amortized O(1))
   - Memory usage optimization

3. Implementation Details:
   - Handle wrap-around conditions
   - Efficient capacity doubling
   - Memory cleanup on resize
```

## ✅ **Self-Assessment Checklist**

### **Data Structures** ✅
- [ ] Can implement arrays and dynamic arrays
- [ ] Can implement linked lists (singly and doubly)
- [ ] Can implement stacks and queues
- [ ] Can implement trees and hash tables

### **Algorithm Analysis** ✅
- [ ] Can analyze time complexity using Big O notation
- [ ] Can analyze space complexity
- [ ] Can compare algorithm efficiency
- [ ] Can optimize algorithms for better performance

### **Search and Sort** ✅
- [ ] Can implement linear and binary search
- [ ] Can implement basic sorting algorithms
- [ ] Can optimize search and sort for specific cases
- [ ] Can handle edge cases and error conditions

### **Embedded Optimization** ✅
- [ ] Can design memory-efficient data structures
- [ ] Can optimize for cache performance
- [ ] Can implement deterministic algorithms
- [ ] Can balance memory usage vs. performance

### **Problem Solving** ✅
- [ ] Can break down complex problems
- [ ] Can select appropriate data structures
- [ ] Can implement efficient solutions
- [ ] Can optimize and refactor code

## 🔗 **Related Topics**
- [C Programming Interview](./C_Programming_Interview.md)
- [RTOS Interview](./RTOS_Interview.md)
- [Performance Optimization Interview](../Advanced_Level/Performance_Optimization_Interview.md)
- [System Integration Interview](../Intermediate_Level/System_Integration_Interview.md)

## 📚 **Additional Resources**
- **Algorithm Visualization**: [VisuAlgo](https://visualgo.net/)
- **Practice Problems**: [LeetCode](https://leetcode.com/), [HackerRank](https://www.hackerrank.com/)
- **Algorithm Analysis**: [Big-O Cheat Sheet](https://www.bigocheatsheet.com/)
- **Books**: "Introduction to Algorithms" by CLRS, "Algorithms" by Sedgewick
