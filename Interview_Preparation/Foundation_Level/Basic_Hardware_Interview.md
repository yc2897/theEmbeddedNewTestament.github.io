# 🎯 **Basic Hardware Interview Preparation**

> **Master Hardware Fundamentals for Embedded Systems Interviews**  
> GPIO configuration, interrupts, timers, communication protocols, and hardware-software interaction

---

## 📋 **Quick Navigation**
- [Common Questions](#common-interview-questions)
- [Problem-Solving Examples](#problem-solving-examples)
- [Practice Problems](#practice-problems)
- [Self-Assessment](#self-assessment-checklist)
- [Resources](#additional-resources)

---

## 🚀 **Quick Reference: Key Concepts**

- **GPIO**: Input/output configuration, pull-up/down resistors, drive strength, interrupt triggering
- **Interrupts**: Edge/level triggered, priority management, ISR design, interrupt safety
- **Timers**: Configuration, prescaler, period, input capture, output compare, PWM
- **Communication**: UART, SPI, I2C, CAN - configuration, error handling, timing
- **ADC/DAC**: Resolution, sampling, signal integrity, calibration
- **Hardware-Software Interface**: Register access, bit manipulation, hardware abstraction

---

## 🎯 **Common Interview Questions**

### **Question 1: Configure GPIO for different modes and explain the trade-offs**

**Problem**: Configure a GPIO pin for different modes (input, output, alternate function) and explain when to use each.

**Why this matters**: GPIO configuration is fundamental to embedded systems and demonstrates understanding of hardware-software interaction.

**Answer Structure**:
```c
// GPIO configuration structure
typedef struct {
    uint8_t mode;           // Input/Output/Alternate/Analog
    uint8_t type;           // Push-pull/Open-drain
    uint8_t speed;          // Low/Medium/High speed
    uint8_t pull_up_down;   // No pull/Up/Down
} gpio_config_t;

// Configure GPIO pin
void configure_gpio_pin(uint8_t pin, gpio_config_t *config) {
    // Set pin mode
    set_pin_mode(pin, config->mode);
    
    // Configure output type and speed if output
    if (config->mode == GPIO_MODE_OUTPUT) {
        set_output_type(pin, config->type);
        set_output_speed(pin, config->speed);
    }
    
    // Set pull-up/pull-down
    set_pull_up_down(pin, config->pull_up_down);
}
```

**Configuration Examples**:

**Digital Input with Pull-up**:
```c
gpio_config_t button_config = {
    .mode = GPIO_MODE_INPUT,
    .type = GPIO_TYPE_PUSH_PULL,      // Not used for input
    .speed = GPIO_SPEED_LOW,          // Not used for input
    .pull_up_down = GPIO_PULL_UP      // Enable pull-up resistor
};
// Use case: Button input, active low when pressed
```

**Digital Output for LED**:
```c
gpio_config_t led_config = {
    .mode = GPIO_MODE_OUTPUT,
    .type = GPIO_TYPE_PUSH_PULL,      // Strong drive capability
    .speed = GPIO_SPEED_LOW,          // Low speed for LED
    .pull_up_down = GPIO_PULL_NONE    // No pull needed for output
};
// Use case: LED control, simple on/off
```

**High-Speed Output for Communication**:
```c
gpio_config_t comm_config = {
    .mode = GPIO_MODE_OUTPUT,
    .type = GPIO_TYPE_PUSH_PULL,      // Fast switching
    .speed = GPIO_SPEED_HIGH,         // High speed for fast signals
    .pull_up_down = GPIO_PULL_NONE    // No pull needed
};
// Use case: SPI clock, UART TX, fast digital signals
```

**Follow-up Questions**:
- When would you use open-drain vs push-pull?
- How does GPIO speed affect power consumption?
- What happens if you don't configure pull-up/down for input pins?

**Key Points**:
- **Input Mode**: Configure pull-up/down based on signal characteristics
- **Output Mode**: Choose speed based on signal requirements and power constraints
- **Alternate Function**: Use for communication protocols and specialized functions
- **Analog Mode**: For ADC/DAC connections

---

### **Question 2: Design an interrupt-driven system for button input**

**Problem**: Design a system that detects button presses using interrupts and debounces the input.

**Why this matters**: Interrupt-driven input handling is essential for responsive embedded systems and demonstrates understanding of real-time programming.

**Solution**:
```c
// Button state and debouncing
typedef struct {
    volatile bool button_pressed;      // Current button state
    volatile uint32_t press_time;      // Timestamp of press
    volatile uint8_t debounce_count;   // Debounce counter
    uint8_t pin;                       // GPIO pin number
    uint32_t debounce_time_ms;         // Debounce time
} button_t;

// Button configuration
button_t button = {
    .button_pressed = false,
    .press_time = 0,
    .debounce_count = 0,
    .pin = BUTTON_PIN,
    .debounce_time_ms = 50
};

// GPIO interrupt handler
void EXTI0_IRQHandler(void) {
    // Clear interrupt flag
    EXTI->PR = (1 << 0);
    
    // Read button state (active low)
    bool button_state = !(GPIOA->IDR & (1 << button.pin));
    
    if (button_state) {
        // Button pressed, start debouncing
        button.debounce_count++;
        button.press_time = get_system_tick();
    } else {
        // Button released
        button.debounce_count = 0;
    }
}

// Main loop processes debounced button state
void main_loop(void) {
    static uint32_t last_button_check = 0;
    uint32_t current_time = get_system_tick();
    
    // Check button state every 10ms
    if (current_time - last_button_check >= 10) {
        last_button_check = current_time;
        
        // Process button if debouncing is complete
        if (button.debounce_count > 0 && 
            (current_time - button.press_time) >= button.debounce_time_ms) {
            
            if (button.debounce_count >= 3) {  // Require 3 consecutive reads
                button.button_pressed = true;
                handle_button_press();
            }
            button.debounce_count = 0;
        }
    }
    
    // Process other tasks
    process_system_tasks();
}
```

**Follow-up Questions**:
- How would you handle multiple buttons with different priorities?
- What happens if the interrupt occurs too frequently?
- How can you implement different debounce strategies?

**Key Points**:
- **Interrupt Safety**: Use volatile for shared variables
- **Debouncing**: Combine hardware and software debouncing
- **Priority Management**: Consider interrupt priorities for multiple inputs
- **Resource Efficiency**: Balance responsiveness with system overhead

---

### **Question 3: Configure a timer for PWM generation with variable duty cycle**

**Problem**: Configure a timer to generate PWM signals with configurable frequency and duty cycle.

**Why this matters**: PWM generation is fundamental for motor control, LED dimming, and analog signal generation.

**Solution**:
```c
// PWM configuration structure
typedef struct {
    uint32_t frequency_hz;     // PWM frequency
    uint8_t duty_cycle_percent; // Duty cycle (0-100%)
    uint32_t timer_clock_hz;   // Timer clock frequency
    uint8_t timer_channel;     // Timer channel number
} pwm_config_t;

// Configure timer for PWM
void configure_pwm_timer(pwm_config_t *config) {
    // Calculate timer period and prescaler
    uint32_t timer_period = (config->timer_clock_hz / config->frequency_hz) - 1;
    uint32_t prescaler = 0;
    
    // Adjust prescaler if period exceeds timer maximum
    while (timer_period > 0xFFFF && prescaler < 0xFFFF) {
        prescaler++;
        timer_period = (config->timer_clock_hz / (config->frequency_hz * (prescaler + 1))) - 1;
    }
    
    // Configure timer
    TIM2->PSC = prescaler;
    TIM2->ARR = timer_period;
    
    // Configure PWM mode
    TIM2->CCMR1 &= ~(0x7 << (config->timer_channel * 8));
    TIM2->CCMR1 |= (0x6 << (config->timer_channel * 8));  // PWM mode 1
    
    // Enable output
    TIM2->CCER |= (1 << (config->timer_channel * 4));
    
    // Start timer
    TIM2->CR1 |= TIM_CR1_CEN;
}

// Set PWM duty cycle
void set_pwm_duty_cycle(uint8_t duty_cycle_percent) {
    if (duty_cycle_percent > 100) duty_cycle_percent = 100;
    
    // Calculate compare value
    uint32_t compare_value = (TIM2->ARR * duty_cycle_percent) / 100;
    
    // Set compare value for channel 1
    TIM2->CCR1 = compare_value;
}

// Example usage
void setup_led_pwm(void) {
    pwm_config_t led_pwm = {
        .frequency_hz = 1000,      // 1kHz PWM
        .duty_cycle_percent = 50,   // 50% duty cycle
        .timer_clock_hz = 84000000, // 84MHz system clock
        .timer_channel = 1          // Channel 1
    };
    
    configure_pwm_timer(&led_pwm);
    set_pwm_duty_cycle(50);
}
```

**Follow-up Questions**:
- How would you implement smooth duty cycle transitions?
- What's the relationship between frequency and resolution?
- How can you generate multiple PWM signals with different frequencies?

**Key Points**:
- **Frequency vs Resolution**: Higher frequency reduces resolution
- **Prescaler Selection**: Balance frequency range with resolution
- **Duty Cycle Calculation**: Use compare value relative to period
- **Channel Configuration**: Multiple channels for different outputs

---

### **Question 4: Implement UART communication with interrupt handling**

**Problem**: Implement UART communication using interrupts for both transmission and reception.

**Why this matters**: UART is fundamental for debugging, communication, and demonstrates understanding of interrupt-driven I/O.

**Solution**:
```c
// UART buffer structure
typedef struct {
    uint8_t rx_buffer[UART_RX_BUFFER_SIZE];
    uint8_t tx_buffer[UART_TX_BUFFER_SIZE];
    volatile uint16_t rx_head;
    volatile uint16_t rx_tail;
    volatile uint16_t tx_head;
    volatile uint16_t tx_tail;
    volatile bool tx_busy;
} uart_buffers_t;

uart_buffers_t uart_buffers = {0};

// UART initialization
void uart_init(void) {
    // Configure GPIO for UART (TX: PA9, RX: PA10)
    GPIOA->MODER &= ~(0x3 << (9 * 2));  // Clear mode
    GPIOA->MODER |= (0x2 << (9 * 2));   // Set alternate function
    
    // Configure UART peripheral
    UART1->BRR = 0x1A1;  // 115200 baud at 42MHz
    UART1->CR1 = UART_CR1_TE | UART_CR1_RE | UART_CR1_RXNEIE;
    UART1->CR1 |= UART_CR1_UART_EN;
    
    // Enable UART interrupt
    NVIC_EnableIRQ(UART1_IRQn);
    NVIC_SetPriority(UART1_IRQn, 1);
}

// UART interrupt handler
void UART1_IRQHandler(void) {
    // Handle receive interrupt
    if (UART1->SR & UART_SR_RXNE) {
        uint8_t data = UART1->DR;
        
        // Add to receive buffer
        uint16_t next_head = (uart_buffers.rx_head + 1) % UART_RX_BUFFER_SIZE;
        if (next_head != uart_buffers.rx_tail) {
            uart_buffers.rx_buffer[uart_buffers.rx_head] = data;
            uart_buffers.rx_head = next_head;
        }
    }
    
    // Handle transmit interrupt
    if (UART1->SR & UART_SR_TXE) {
        if (uart_buffers.tx_head != uart_buffers.tx_tail) {
            // Send next byte
            UART1->DR = uart_buffers.tx_buffer[uart_buffers.tx_tail];
            uart_buffers.tx_tail = (uart_buffers.tx_tail + 1) % UART_TX_BUFFER_SIZE;
        } else {
            // No more data to send
            uart_buffers.tx_busy = false;
            UART1->CR1 &= ~UART_CR1_TXEIE;  // Disable TX interrupt
        }
    }
}

// Send data via UART
void uart_send(const uint8_t *data, uint16_t length) {
    // Wait if transmit buffer is full
    while (((uart_buffers.tx_head + 1) % UART_TX_BUFFER_SIZE) == uart_buffers.tx_tail) {
        // Buffer full, wait
    }
    
    // Add data to transmit buffer
    for (uint16_t i = 0; i < length; i++) {
        uart_buffers.tx_buffer[uart_buffers.tx_head] = data[i];
        uart_buffers.tx_head = (uart_buffers.tx_head + 1) % UART_TX_BUFFER_SIZE;
    }
    
    // Enable transmit interrupt if not already busy
    if (!uart_buffers.tx_busy) {
        uart_buffers.tx_busy = true;
        UART1->CR1 |= UART_CR1_TXEIE;
    }
}

// Receive data from UART
uint16_t uart_receive(uint8_t *data, uint16_t max_length) {
    uint16_t received = 0;
    
    while (uart_buffers.rx_head != uart_buffers.rx_tail && received < max_length) {
        data[received] = uart_buffers.rx_buffer[uart_buffers.rx_tail];
        uart_buffers.rx_tail = (uart_buffers.rx_tail + 1) % UART_TX_BUFFER_SIZE;
        received++;
    }
    
    return received;
}
```

**Follow-up Questions**:
- How would you implement flow control (XON/XOFF)?
- What happens if the receive buffer overflows?
- How can you implement different baud rates dynamically?

**Key Points**:
- **Buffer Management**: Use circular buffers for efficient memory usage
- **Interrupt Safety**: Protect shared variables with volatile
- **Flow Control**: Implement buffer overflow protection
- **Error Handling**: Check for framing errors and buffer conditions

---

## 🧪 **Practice Problems**

### **Problem 1: Multi-Button Interrupt System**

**Scenario**: Design a system that handles multiple buttons using a single interrupt line and GPIO port.

**Requirements**:
- Support up to 8 buttons on GPIO port A
- Use external interrupt on any pin change
- Implement debouncing for each button
- Different actions for different buttons

**Solution Approach**:
1. Configure all pins as inputs with pull-up
2. Enable external interrupt on port A
3. Use GPIO IDR to read all button states
4. Implement individual debouncing counters
5. Handle different button actions

**Key Learning Points**:
- Single interrupt for multiple inputs
- Efficient debouncing implementation
- Button state tracking and action mapping

---

### **Problem 2: Timer-Based ADC Sampling**

**Scenario**: Implement ADC sampling using a timer interrupt at a fixed frequency.

**Requirements**:
- Sample ADC at 1kHz
- Store samples in circular buffer
- Process data when buffer is half full
- Handle buffer overflow gracefully

**Solution Approach**:
1. Configure timer for 1kHz interrupt
2. Configure ADC for continuous conversion
3. Use DMA or interrupt to store samples
4. Implement circular buffer management
5. Process data in main loop

**Key Learning Points**:
- Timer-ADC synchronization
- Buffer management for continuous data
- Interrupt-driven data acquisition

---

## ✅ **Self-Assessment Checklist**

### **Basic Understanding** ✅
- [ ] **GPIO Configuration**: Can configure pins for different modes
- [ ] **Interrupt Handling**: Understand interrupt setup and ISR design
- [ ] **Timer Configuration**: Can configure timers for various applications
- [ ] **Communication Protocols**: Understand UART, SPI, I2C basics

### **Problem Solving** ✅
- [ ] **Hardware-Software Interface**: Can design register-based interfaces
- [ ] **Interrupt Safety**: Can implement interrupt-safe code
- [ ] **Buffer Management**: Can implement circular buffers and FIFOs
- [ ] **Error Handling**: Can handle hardware errors and edge cases

### **Advanced Concepts** ✅
- [ ] **Protocol Implementation**: Can implement communication protocols
- [ ] **Real-Time Programming**: Understand timing constraints and deadlines
- [ ] **Hardware Abstraction**: Can design portable hardware interfaces
- [ ] **Performance Optimization**: Can optimize for speed and power

---

## 🔗 **Related Learning Modules**

- **[GPIO Configuration](../Hardware_Fundamentals/GPIO_Configuration.md)** - GPIO modes, configuration, electrical characteristics
- **[External Interrupts](../Hardware_Fundamentals/External_Interrupts.md)** - Edge/level triggered interrupts, debouncing
- **[Timer/Counter Programming](../Hardware_Fundamentals/Timer_Counter_Programming.md)** - Timer configuration, PWM generation
- **[Analog I/O](../Hardware_Fundamentals/Analog_IO.md)** - ADC/DAC configuration and signal processing
- **[Communication Protocols](../Communication_Protocols/README.md)** - UART, SPI, I2C, CAN implementation

---

## 📚 **Additional Resources**

### **Books**
- "Making Embedded Systems" by Elecia White
- "Embedded Systems: Introduction to ARM Cortex-M Microcontrollers" by Jonathan Valvano
- "The Definitive Guide to ARM Cortex-M3 and Cortex-M4 Processors" by Joseph Yiu

### **Online Resources**
- [ARM Cortex-M Technical Reference Manual](https://developer.arm.com/documentation/ddi0337/)
- [STM32 Reference Manuals](https://www.st.com/en/microcontrollers-microprocessors/stm32-32-bit-arm-cortex-mcus.html)
- [Embedded.com Hardware Articles](https://www.embedded.com/)

### **Practice Platforms**
- [STM32CubeMX](https://www.st.com/en/development-tools/stm32cubemx.html) - Hardware configuration
- [ARM Mbed](https://os.mbed.com/) - Online IDE and examples
- [GitHub STM32 Projects](https://github.com/topics/stm32) - Open-source examples

---

## 🎯 **Interview Success Tips**

### **Before the Interview**
- **Review Hardware Fundamentals**: Understand GPIO, interrupts, timers, and communication
- **Practice Configuration**: Be able to configure peripherals from scratch
- **Understand Trade-offs**: Know when to use different approaches
- **Review Datasheets**: Be comfortable reading and interpreting hardware specifications

### **During the Interview**
- **Think Systematically**: Start with requirements, then design the solution
- **Consider Constraints**: Think about timing, power, and resource limitations
- **Explain Trade-offs**: Discuss why you chose specific approaches
- **Handle Edge Cases**: Consider error conditions and boundary cases

### **Common Pitfalls to Avoid**
- **Ignoring Interrupt Safety**: Always consider shared variable protection
- **Buffer Overflow**: Implement proper buffer management
- **Timing Issues**: Consider real-time constraints and deadlines
- **Hardware Assumptions**: Don't assume specific hardware behavior

---

**Next Topic**: [Problem-Solving Approach](./Problem_Solving_Approach.md) → [Real-Time Systems Interview](../Intermediate_Level/Real_Time_Systems_Interview.md)
