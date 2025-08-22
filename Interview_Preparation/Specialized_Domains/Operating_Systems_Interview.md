# 🎯 Operating Systems Interview Preparation

## 🚀 **Quick Navigation**
- [Common Questions](#common-questions)
- [Problem-Solving Examples](#problem-solving-examples)
- [Practice Problems](#practice-problems)
- [Resources](#resources)

## 📚 **Quick Reference: Key Concepts**
- **Linux Kernel**: Kernel modules, device drivers, system calls, memory management
- **Device Drivers**: Character, block, network drivers, lifecycle management
- **Real-time Linux**: PREEMPT_RT, Xenomai, real-time extensions
- **Embedded Linux**: Buildroot, Yocto, custom distributions
- **System Programming**: POSIX APIs, process management, inter-process communication

## 🎯 **Common Interview Questions**

### **Question 1: Implement a character device driver with ioctl support**

**Why this matters**: Device drivers are fundamental to embedded Linux systems and demonstrate kernel programming skills.

**Problem**: Create a character device driver for a custom sensor that supports read, write, and ioctl operations.

**Requirements**:
- Support read/write operations
- Implement custom ioctl commands
- Handle multiple device instances
- Support poll/select operations

**Solution Design**:
```c
#include <linux/module.h>
#include <linux/fs.h>
#include <linux/uaccess.h>
#include <linux/device.h>
#include <linux/cdev.h>
#include <linux/slab.h>
#include <linux/poll.h>
#include <linux/wait.h>

#define DEVICE_NAME "custom_sensor"
#define MAX_DEVICES 4
#define BUFFER_SIZE 1024

// Custom ioctl commands
#define SENSOR_GET_STATUS _IOR('S', 0, int)
#define SENSOR_SET_MODE _IOW('S', 1, int)
#define SENSOR_GET_CALIBRATION _IOR('S', 2, struct sensor_calib)
#define SENSOR_SET_CALIBRATION _IOW('S', 3, struct sensor_calib)

// Sensor calibration structure
struct sensor_calib {
    int offset;
    int scale;
    int temperature;
};

// Device private data
struct sensor_dev {
    struct cdev cdev;
    struct device *device;
    char buffer[BUFFER_SIZE];
    size_t buffer_len;
    struct mutex lock;
    wait_queue_head_t read_queue;
    bool data_ready;
    int mode;
    struct sensor_calib calibration;
};

static struct sensor_dev *sensor_devices[MAX_DEVICES];
static int major_number;
static struct class *sensor_class;

// File operations
static int sensor_open(struct inode *inode, struct file *file) {
    struct sensor_dev *dev = container_of(inode->i_cdev, struct sensor_dev, cdev);
    file->private_data = dev;
    
    pr_info("Sensor device opened\n");
    return 0;
}

static int sensor_release(struct inode *inode, struct file *file) {
    pr_info("Sensor device closed\n");
    return 0;
}

static ssize_t sensor_read(struct file *file, char __user *buf, 
                          size_t count, loff_t *ppos) {
    struct sensor_dev *dev = file->private_data;
    ssize_t ret = 0;
    
    if (mutex_lock_interruptible(&dev->lock))
        return -ERESTARTSYS;
    
    // Wait for data to be ready
    while (!dev->data_ready && !signal_pending(current)) {
        mutex_unlock(&dev->lock);
        if (wait_event_interruptible(dev->read_queue, dev->data_ready))
            return -ERESTARTSYS;
        if (mutex_lock_interruptible(&dev->lock))
            return -ERESTARTSYS;
    }
    
    if (dev->data_ready) {
        count = min(count, dev->buffer_len);
        if (copy_to_user(buf, dev->buffer, count)) {
            ret = -EFAULT;
        } else {
            ret = count;
            dev->data_ready = false;
        }
    }
    
    mutex_unlock(&dev->lock);
    return ret;
}

static ssize_t sensor_write(struct file *file, const char __user *buf,
                           size_t count, loff_t *ppos) {
    struct sensor_dev *dev = file->private_data;
    ssize_t ret = 0;
    
    if (mutex_lock_interruptible(&dev->lock))
        return -ERESTARTSYS;
    
    count = min(count, BUFFER_SIZE);
    if (copy_from_user(dev->buffer, buf, count)) {
        ret = -EFAULT;
    } else {
        dev->buffer_len = count;
        dev->data_ready = true;
        wake_up_interruptible(&dev->read_queue);
        ret = count;
    }
    
    mutex_unlock(&dev->lock);
    return ret;
}

static long sensor_ioctl(struct file *file, unsigned int cmd, unsigned long arg) {
    struct sensor_dev *dev = file->private_data;
    long ret = 0;
    
    if (mutex_lock_interruptible(&dev->lock))
        return -ERESTARTSYS;
    
    switch (cmd) {
        case SENSOR_GET_STATUS:
            ret = put_user(dev->mode, (int __user *)arg);
            break;
            
        case SENSOR_SET_MODE:
            ret = get_user(dev->mode, (int __user *)arg);
            pr_info("Sensor mode set to %d\n", dev->mode);
            break;
            
        case SENSOR_GET_CALIBRATION:
            ret = copy_to_user((struct sensor_calib __user *)arg, 
                              &dev->calibration, sizeof(struct sensor_calib));
            if (ret)
                ret = -EFAULT;
            break;
            
        case SENSOR_SET_CALIBRATION:
            ret = copy_from_user(&dev->calibration, 
                                (struct sensor_calib __user *)arg, 
                                sizeof(struct sensor_calib));
            if (ret)
                ret = -EFAULT;
            else
                pr_info("Calibration updated: offset=%d, scale=%d\n", 
                       dev->calibration.offset, dev->calibration.scale);
            break;
            
        default:
            ret = -ENOTTY;
            break;
    }
    
    mutex_unlock(&dev->lock);
    return ret;
}

static unsigned int sensor_poll(struct file *file, poll_table *wait) {
    struct sensor_dev *dev = file->private_data;
    unsigned int mask = 0;
    
    poll_wait(file, &dev->read_queue, wait);
    
    if (mutex_lock_interruptible(&dev->lock))
        return -ERESTARTSYS;
    
    if (dev->data_ready)
        mask |= POLLIN | POLLRDNORM;
    
    mutex_unlock(&dev->lock);
    return mask;
}

static const struct file_operations sensor_fops = {
    .owner = THIS_MODULE,
    .open = sensor_open,
    .release = sensor_release,
    .read = sensor_read,
    .write = sensor_write,
    .unlocked_ioctl = sensor_ioctl,
    .poll = sensor_poll,
};

// Module initialization
static int __init sensor_init(void) {
    int ret, i;
    dev_t dev;
    
    // Allocate major number
    ret = alloc_chrdev_region(&dev, 0, MAX_DEVICES, DEVICE_NAME);
    if (ret < 0) {
        pr_err("Failed to allocate major number\n");
        return ret;
    }
    major_number = MAJOR(dev);
    
    // Create device class
    sensor_class = class_create(THIS_MODULE, DEVICE_NAME);
    if (IS_ERR(sensor_class)) {
        pr_err("Failed to create device class\n");
        unregister_chrdev_region(dev, MAX_DEVICES);
        return PTR_ERR(sensor_class);
    }
    
    // Initialize devices
    for (i = 0; i < MAX_DEVICES; i++) {
        sensor_devices[i] = kzalloc(sizeof(struct sensor_dev), GFP_KERNEL);
        if (!sensor_devices[i]) {
            ret = -ENOMEM;
            goto cleanup;
        }
        
        // Initialize device
        cdev_init(&sensor_devices[i]->cdev, &sensor_fops);
        sensor_devices[i]->cdev.owner = THIS_MODULE;
        
        // Add device
        dev = MKDEV(major_number, i);
        ret = cdev_add(&sensor_devices[i]->cdev, dev, 1);
        if (ret < 0) {
            pr_err("Failed to add device %d\n", i);
            goto cleanup;
        }
        
        // Create device file
        sensor_devices[i]->device = device_create(sensor_class, NULL, dev, 
                                                NULL, "%s%d", DEVICE_NAME, i);
        if (IS_ERR(sensor_devices[i]->device)) {
            pr_err("Failed to create device file %d\n", i);
            cdev_del(&sensor_devices[i]->cdev);
            goto cleanup;
        }
        
        // Initialize device data
        mutex_init(&sensor_devices[i]->lock);
        init_waitqueue_head(&sensor_devices[i]->read_queue);
        sensor_devices[i]->mode = 0;
        sensor_devices[i]->calibration.offset = 0;
        sensor_devices[i]->calibration.scale = 1000;
    }
    
    pr_info("Sensor driver loaded successfully\n");
    return 0;
    
cleanup:
    for (i = 0; i < MAX_DEVICES; i++) {
        if (sensor_devices[i]) {
            if (sensor_devices[i]->device)
                device_destroy(sensor_class, MKDEV(major_number, i));
            cdev_del(&sensor_devices[i]->cdev);
            kfree(sensor_devices[i]);
        }
    }
    class_destroy(sensor_class);
    unregister_chrdev_region(MKDEV(major_number, 0), MAX_DEVICES);
    return ret;
}

// Module cleanup
static void __exit sensor_exit(void) {
    int i;
    
    for (i = 0; i < MAX_DEVICES; i++) {
        if (sensor_devices[i]) {
            device_destroy(sensor_class, MKDEV(major_number, i));
            cdev_del(&sensor_devices[i]->cdev);
            kfree(sensor_devices[i]);
        }
    }
    
    class_destroy(sensor_class);
    unregister_chrdev_region(MKDEV(major_number, 0), MAX_DEVICES);
    
    pr_info("Sensor driver unloaded\n");
}

module_init(sensor_init);
module_exit(sensor_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Embedded Engineer");
MODULE_DESCRIPTION("Custom Sensor Driver");
```

**Driver Features**:
- **Multiple Instances**: Support for up to 4 devices
- **Ioctl Support**: Custom commands for configuration
- **Poll/Select**: Asynchronous I/O support
- **Proper Synchronization**: Mutex and wait queues

**Follow-up Questions**:
- How would you handle interrupt-driven I/O?
- What if you need to support DMA transfers?

### **Question 2: Design a real-time task scheduler using PREEMPT_RT**

**Problem**: Implement a real-time task scheduler that can handle periodic tasks with deadlines.

**Requirements**:
- Support periodic tasks
- Handle task priorities
- Implement deadline monitoring
- Support task preemption

**Solution Design**:
```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/sched.h>
#include <linux/sched/rt.h>
#include <linux/timer.h>
#include <linux/workqueue.h>
#include <linux/spinlock.h>
#include <linux/list.h>

#define MAX_TASKS 16
#define MAX_PRIORITY 99

// Task states
typedef enum {
    TASK_STATE_READY,
    TASK_STATE_RUNNING,
    TASK_STATE_WAITING,
    TASK_STATE_COMPLETED
} task_state_t;

// Real-time task structure
typedef struct rt_task {
    struct list_head list;
    int task_id;
    char name[32];
    void (*function)(void*);
    void *data;
    unsigned long period;      // Period in nanoseconds
    unsigned long deadline;    // Deadline in nanoseconds
    unsigned long last_release; // Last release time
    unsigned long next_release; // Next release time
    int priority;
    task_state_t state;
    unsigned long execution_time;
    unsigned long missed_deadlines;
    spinlock_t lock;
} rt_task_t;

// Scheduler context
typedef struct {
    struct list_head ready_queue;
    struct list_head waiting_queue;
    rt_task_t *current_task;
    spinlock_t scheduler_lock;
    struct timer_list scheduler_timer;
    unsigned long current_time;
    int num_tasks;
} rt_scheduler_t;

static rt_scheduler_t scheduler;

// Initialize real-time task
int rt_task_create(rt_task_t *task, const char *name, 
                   void (*function)(void*), void *data,
                   unsigned long period, unsigned long deadline, int priority) {
    if (!task || !function || period == 0 || deadline == 0)
        return -EINVAL;
    
    if (priority < 1 || priority > MAX_PRIORITY)
        return -EINVAL;
    
    // Initialize task
    task->task_id = scheduler.num_tasks++;
    strncpy(task->name, name, sizeof(task->name) - 1);
    task->function = function;
    task->data = data;
    task->period = period;
    task->deadline = deadline;
    task->priority = priority;
    task->state = TASK_STATE_READY;
    task->execution_time = 0;
    task->missed_deadlines = 0;
    
    // Set initial release time
    task->last_release = 0;
    task->next_release = ktime_get_ns();
    
    spin_lock_init(&task->lock);
    
    // Add to ready queue
    spin_lock(&scheduler.scheduler_lock);
    list_add_tail(&task->list, &scheduler.ready_queue);
    spin_unlock(&scheduler.scheduler_lock);
    
    pr_info("RT Task '%s' created with period %lu ns, deadline %lu ns\n", 
            name, period, deadline);
    
    return 0;
}

// Start real-time task
int rt_task_start(rt_task_t *task) {
    unsigned long flags;
    
    spin_lock_irqsave(&task->lock, flags);
    
    if (task->state != TASK_STATE_READY) {
        spin_unlock_irqrestore(&task->lock, flags);
        return -EINVAL;
    }
    
    task->state = TASK_STATE_WAITING;
    task->next_release = ktime_get_ns();
    
    spin_unlock_irqrestore(&task->lock, flags);
    
    pr_info("RT Task '%s' started\n", task->name);
    return 0;
}

// Stop real-time task
int rt_task_stop(rt_task_t *task) {
    unsigned long flags;
    
    spin_lock_irqsave(&task->lock, flags);
    
    if (task->state == TASK_STATE_RUNNING) {
        // Preempt current task
        if (scheduler.current_task == task) {
            scheduler.current_task = NULL;
        }
    }
    
    task->state = TASK_STATE_READY;
    
    spin_unlock_irqrestore(&task->lock, flags);
    
    pr_info("RT Task '%s' stopped\n", task->name);
    return 0;
}

// Set task priority
int rt_task_set_priority(rt_task_t *task, int priority) {
    unsigned long flags;
    
    if (priority < 1 || priority > MAX_PRIORITY)
        return -EINVAL;
    
    spin_lock_irqsave(&task->lock, flags);
    task->priority = priority;
    spin_unlock_irqrestore(&task->lock, flags);
    
    return 0;
}

// Scheduler timer callback
static void scheduler_timer_callback(struct timer_list *t) {
    unsigned long flags;
    rt_task_t *task, *next_task = NULL;
    unsigned long current_time = ktime_get_ns();
    
    spin_lock_irqsave(&scheduler.scheduler_lock, flags);
    
    // Check for tasks that need to be released
    list_for_each_entry(task, &scheduler.waiting_queue, list) {
        if (current_time >= task->next_release) {
            // Move to ready queue
            list_del(&task->list);
            list_add_tail(&task->list, &scheduler.ready_queue);
            task->state = TASK_STATE_READY;
            task->last_release = task->next_release;
            task->next_release += task->period;
            
            pr_debug("Task '%s' released at %lu\n", task->name, current_time);
        }
    }
    
    // Select next task to run (highest priority first)
    if (!scheduler.current_task || scheduler.current_task->state != TASK_STATE_RUNNING) {
        list_for_each_entry(task, &scheduler.ready_queue, list) {
            if (!next_task || task->priority > next_task->priority) {
                next_task = task;
            }
        }
        
        if (next_task) {
            // Preempt current task if needed
            if (scheduler.current_task && 
                next_task->priority > scheduler.current_task->priority) {
                scheduler.current_task->state = TASK_STATE_READY;
                pr_debug("Task '%s' preempted by '%s'\n", 
                        scheduler.current_task->name, next_task->name);
            }
            
            scheduler.current_task = next_task;
            next_task->state = TASK_STATE_RUNNING;
            
            // Execute task
            spin_unlock_irqrestore(&scheduler.scheduler_lock, flags);
            next_task->function(next_task->data);
            spin_lock_irqsave(&scheduler.scheduler_lock, flags);
            
            next_task->state = TASK_STATE_WAITING;
            scheduler.current_task = NULL;
            
            // Check deadline
            if (ktime_get_ns() > next_task->last_release + next_task->deadline) {
                next_task->missed_deadlines++;
                pr_warn("Task '%s' missed deadline\n", next_task->name);
            }
        }
    }
    
    spin_unlock_irqrestore(&scheduler.scheduler_lock, flags);
    
    // Reschedule timer
    mod_timer(&scheduler.scheduler_timer, jiffies + 1);
}

// Initialize scheduler
static int __init rt_scheduler_init(void) {
    // Initialize scheduler
    INIT_LIST_HEAD(&scheduler.ready_queue);
    INIT_LIST_HEAD(&scheduler.waiting_queue);
    scheduler.current_task = NULL;
    scheduler.num_tasks = 0;
    spin_lock_init(&scheduler.scheduler_lock);
    
    // Setup scheduler timer
    timer_setup(&scheduler.scheduler_timer, scheduler_timer_callback, 0);
    mod_timer(&scheduler.scheduler_timer, jiffies + 1);
    
    pr_info("Real-time scheduler initialized\n");
    return 0;
}

// Cleanup scheduler
static void __exit rt_scheduler_exit(void) {
    del_timer_sync(&scheduler.scheduler_timer);
    pr_info("Real-time scheduler unloaded\n");
}

module_init(rt_scheduler_init);
module_exit(rt_scheduler_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Embedded Engineer");
MODULE_DESCRIPTION("Real-time Task Scheduler");
```

**Scheduler Features**:
- **Priority-based Scheduling**: Higher priority tasks run first
- **Deadline Monitoring**: Track missed deadlines
- **Task Preemption**: Higher priority tasks can preempt lower ones
- **Periodic Execution**: Support for periodic real-time tasks

### **Question 3: Implement a custom init system for embedded Linux**

**Problem**: Create a lightweight init system that can start and manage system services.

**Requirements**:
- Service dependency management
- Service lifecycle control
- Configuration file parsing
- Logging and monitoring

**Solution Design**:
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <signal.h>
#include <sys/wait.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <errno.h>
#include <time.h>

#define MAX_SERVICES 32
#define MAX_DEPENDENCIES 8
#define MAX_LINE_LENGTH 256
#define CONFIG_FILE "/etc/init.conf"

// Service states
typedef enum {
    SERVICE_STATE_STOPPED,
    SERVICE_STATE_STARTING,
    SERVICE_STATE_RUNNING,
    SERVICE_STATE_STOPPING,
    SERVICE_STATE_FAILED
} service_state_t;

// Service structure
typedef struct {
    char name[64];
    char command[256];
    char dependencies[MAX_DEPENDENCIES][64];
    int num_dependencies;
    pid_t pid;
    service_state_t state;
    int restart_count;
    int max_restarts;
    bool auto_start;
    time_t start_time;
    time_t last_restart;
} service_t;

// Init system context
typedef struct {
    service_t services[MAX_SERVICES];
    int num_services;
    bool running;
    int log_fd;
} init_system_t;

static init_system_t init_system;

// Logging functions
void init_log(const char *format, ...) {
    va_list args;
    char timestamp[64];
    time_t now = time(NULL);
    
    strftime(timestamp, sizeof(timestamp), "%Y-%m-%d %H:%M:%S", localtime(&now));
    
    dprintf(init_system.log_fd, "[%s] ", timestamp);
    
    va_start(args, format);
    vdprintf(init_system.log_fd, format, args);
    va_end(args);
    
    dprintf(init_system.log_fd, "\n");
}

// Parse configuration file
int parse_config_file(const char *filename) {
    FILE *file = fopen(filename, "r");
    if (!file) {
        perror("Failed to open config file");
        return -1;
    }
    
    char line[MAX_LINE_LENGTH];
    service_t *current_service = NULL;
    
    while (fgets(line, sizeof(line), file)) {
        // Skip comments and empty lines
        if (line[0] == '#' || line[0] == '\n')
            continue;
        
        // Remove newline
        line[strcspn(line, "\n")] = 0;
        
        if (strncmp(line, "service ", 8) == 0) {
            // New service definition
            if (init_system.num_services >= MAX_SERVICES) {
                init_log("Too many services, maximum is %d", MAX_SERVICES);
                break;
            }
            
            current_service = &init_system.services[init_system.num_services++];
            memset(current_service, 0, sizeof(service_t));
            
            // Parse service name
            char *name = line + 8;
            strncpy(current_service->name, name, sizeof(current_service->name) - 1);
            current_service->max_restarts = 3;
            current_service->auto_start = true;
            
            init_log("Parsing service: %s", current_service->name);
            
        } else if (current_service && strncmp(line, "  command ", 11) == 0) {
            // Service command
            char *command = line + 11;
            strncpy(current_service->command, command, sizeof(current_service->command) - 1);
            
        } else if (current_service && strncmp(line, "  depends ", 10) == 0) {
            // Service dependencies
            char *deps = line + 10;
            char *token = strtok(deps, " ");
            
            while (token && current_service->num_dependencies < MAX_DEPENDENCIES) {
                strncpy(current_service->dependencies[current_service->num_dependencies], 
                       token, 63);
                current_service->num_dependencies++;
                token = strtok(NULL, " ");
            }
            
        } else if (current_service && strncmp(line, "  max_restarts ", 15) == 0) {
            // Maximum restart count
            current_service->max_restarts = atoi(line + 15);
            
        } else if (current_service && strncmp(line, "  auto_start ", 13) == 0) {
            // Auto-start setting
            if (strstr(line, "true"))
                current_service->auto_start = true;
            else
                current_service->auto_start = false;
        }
    }
    
    fclose(file);
    return 0;
}

// Check if service dependencies are met
bool check_dependencies(service_t *service) {
    for (int i = 0; i < service->num_dependencies; i++) {
        bool found = false;
        
        for (int j = 0; j < init_system.num_services; j++) {
            if (strcmp(service->dependencies[i], init_system.services[j].name) == 0) {
                if (init_system.services[j].state != SERVICE_STATE_RUNNING) {
                    return false;  // Dependency not running
                }
                found = true;
                break;
            }
        }
        
        if (!found) {
            init_log("Service %s depends on unknown service %s", 
                    service->name, service->dependencies[i]);
            return false;
        }
    }
    
    return true;
}

// Start a service
int start_service(service_t *service) {
    if (service->state != SERVICE_STATE_STOPPED && 
        service->state != SERVICE_STATE_FAILED) {
        return -1;  // Service already running or starting
    }
    
    // Check dependencies
    if (!check_dependencies(service)) {
        init_log("Service %s dependencies not met", service->name);
        return -1;
    }
    
    service->state = SERVICE_STATE_STARTING;
    init_log("Starting service: %s", service->name);
    
    // Fork and exec service
    pid_t pid = fork();
    if (pid == 0) {
        // Child process
        close(init_system.log_fd);
        
        // Redirect output to log file
        int log_fd = open("/var/log/services.log", O_WRONLY | O_APPEND | O_CREAT, 0644);
        if (log_fd >= 0) {
            dup2(log_fd, STDOUT_FILENO);
            dup2(log_fd, STDERR_FILENO);
            close(log_fd);
        }
        
        // Execute service command
        execl("/bin/sh", "sh", "-c", service->command, NULL);
        exit(1);
        
    } else if (pid > 0) {
        // Parent process
        service->pid = pid;
        service->start_time = time(NULL);
        service->state = SERVICE_STATE_RUNNING;
        
        init_log("Service %s started with PID %d", service->name, pid);
        return 0;
        
    } else {
        // Fork failed
        service->state = SERVICE_STATE_FAILED;
        init_log("Failed to start service %s: fork error", service->name);
        return -1;
    }
}

// Stop a service
int stop_service(service_t *service) {
    if (service->state != SERVICE_STATE_RUNNING) {
        return -1;  // Service not running
    }
    
    service->state = SERVICE_STATE_STOPPING;
    init_log("Stopping service: %s", service->name);
    
    // Send SIGTERM to service
    if (kill(service->pid, SIGTERM) == 0) {
        // Wait for service to terminate
        int status;
        pid_t result = waitpid(service->pid, &status, WNOHANG);
        
        if (result == 0) {
            // Service still running, send SIGKILL after timeout
            sleep(5);
            if (kill(service->pid, SIGKILL) == 0) {
                waitpid(service->pid, &status, 0);
            }
        }
    }
    
    service->pid = 0;
    service->state = SERVICE_STATE_STOPPED;
    init_log("Service %s stopped", service->name);
    
    return 0;
}

// Restart a service
int restart_service(service_t *service) {
    init_log("Restarting service: %s", service->name);
    
    stop_service(service);
    sleep(1);  // Brief delay between stop and start
    return start_service(service);
}

// Monitor services
void monitor_services() {
    int status;
    pid_t pid;
    
    while ((pid = waitpid(-1, &status, WNOHANG)) > 0) {
        // Find service by PID
        for (int i = 0; i < init_system.num_services; i++) {
            if (init_system.services[i].pid == pid) {
                service_t *service = &init_system.services[i];
                
                if (WIFEXITED(status)) {
                    int exit_code = WEXITSTATUS(status);
                    init_log("Service %s exited with code %d", service->name, exit_code);
                    
                    if (exit_code != 0 && service->restart_count < service->max_restarts) {
                        service->restart_count++;
                        service->last_restart = time(NULL);
                        init_log("Restarting service %s (attempt %d/%d)", 
                                service->name, service->restart_count, service->max_restarts);
                        
                        start_service(service);
                    } else {
                        service->state = SERVICE_STATE_FAILED;
                        init_log("Service %s failed, not restarting", service->name);
                    }
                } else if (WIFSIGNALED(status)) {
                    int signal = WTERMSIG(status);
                    init_log("Service %s killed by signal %d", service->name, signal);
                    service->state = SERVICE_STATE_STOPPED;
                }
                
                service->pid = 0;
                break;
            }
        }
    }
}

// Signal handler
void signal_handler(int sig) {
    if (sig == SIGTERM || sig == SIGINT) {
        init_log("Received shutdown signal, stopping all services");
        init_system.running = false;
    }
}

// Main init system loop
int main(int argc, char *argv[]) {
    // Open log file
    init_system.log_fd = open("/var/log/init.log", O_WRONLY | O_APPEND | O_CREAT, 0644);
    if (init_system.log_fd < 0) {
        perror("Failed to open log file");
        return 1;
    }
    
    init_log("Custom init system starting");
    
    // Parse configuration
    if (parse_config_file(CONFIG_FILE) < 0) {
        init_log("Failed to parse configuration file");
        return 1;
    }
    
    init_log("Loaded %d services", init_system.num_services);
    
    // Set up signal handlers
    signal(SIGTERM, signal_handler);
    signal(SIGINT, signal_handler);
    signal(SIGCHLD, SIG_IGN);  // Let waitpid handle child processes
    
    // Start auto-start services
    for (int i = 0; i < init_system.num_services; i++) {
        if (init_system.services[i].auto_start) {
            start_service(&init_system.services[i]);
        }
    }
    
    // Main loop
    init_system.running = true;
    while (init_system.running) {
        monitor_services();
        usleep(100000);  // 100ms sleep
    }
    
    // Shutdown: stop all services
    for (int i = 0; i < init_system.num_services; i++) {
        if (init_system.services[i].state == SERVICE_STATE_RUNNING) {
            stop_service(&init_system.services[i]);
        }
    }
    
    init_log("Init system shutdown complete");
    close(init_system.log_fd);
    
    return 0;
}
```

**Init System Features**:
- **Configuration Parsing**: Parse service definitions from config file
- **Dependency Management**: Start services in dependency order
- **Service Monitoring**: Track service status and restart failed services
- **Graceful Shutdown**: Stop all services on shutdown signal

## 🧪 **Practice Problems**

### **Problem 1: Kernel Module Memory Management**

**Scenario**: Analyze memory management in a kernel module.

**Question**: What happens when a kernel module allocates memory and how should it be managed?

**Expected Analysis**:
```
1. Memory Allocation Methods:
   - kmalloc: For small allocations, returns physically contiguous memory
   - vmalloc: For large allocations, may not be physically contiguous
   - get_free_pages: For page-aligned allocations
   - kmem_cache_alloc: For frequently allocated objects

2. Memory Management:
   - Always free allocated memory in module_exit
   - Use appropriate allocation flags (GFP_KERNEL, GFP_ATOMIC)
   - Check return values for allocation failures
   - Consider memory fragmentation and alignment

3. Common Issues:
   - Memory leaks from unfreed allocations
   - Using wrong allocation flags in interrupt context
   - Not handling allocation failures
```

### **Problem 2: Real-time Performance Analysis**

**Scenario**: Analyze real-time performance of a Linux system.

**Question**: How would you measure and improve real-time performance?

**Expected Solution**:
```
1. Performance Measurement:
   - Use cyclictest for latency measurement
   - Monitor kernel preemption with /proc/interrupts
   - Check PREEMPT_RT patch status
   - Use ftrace for scheduling analysis

2. Performance Improvement:
   - Enable PREEMPT_RT patch
   - Use real-time scheduling policies (SCHED_FIFO, SCHED_RR)
   - Minimize interrupt handling time
   - Use high-resolution timers

3. Analysis Tools:
   - cyclictest, ftrace, perf
   - Real-time monitoring tools
   - Kernel configuration analysis
```

### **Problem 3: Device Driver Error Handling**

**Scenario**: Design error handling for a device driver.

**Question**: How would you implement robust error handling?

**Expected Solution**:
```
1. Error Categories:
   - Hardware errors (device not responding)
   - Software errors (invalid parameters)
   - Resource errors (memory allocation failure)
   - Communication errors (I/O failures)

2. Error Handling Strategies:
   - Return appropriate error codes
   - Log errors with sufficient detail
   - Implement retry mechanisms
   - Provide user-space error information

3. Recovery Mechanisms:
   - Device reset on critical errors
   - Graceful degradation
   - Automatic retry with backoff
   - User notification of errors
```

## ✅ **Self-Assessment Checklist**

### **Linux Kernel Programming** ✅
- [ ] Can write kernel modules
- [ ] Understand device driver architecture
- [ ] Can implement system calls
- [ ] Know kernel memory management

### **Device Drivers** ✅
- [ ] Can implement character drivers
- [ ] Understand driver lifecycle
- [ ] Can handle interrupts and DMA
- [ ] Know driver debugging techniques

### **Real-time Linux** ✅
- [ ] Can configure PREEMPT_RT
- [ ] Understand real-time scheduling
- [ ] Can measure real-time performance
- [ ] Know real-time best practices

### **Embedded Linux** ✅
- [ ] Can build custom distributions
- [ ] Understand build systems
- [ ] Can optimize for embedded use
- [ ] Know deployment strategies

## 🔗 **Related Topics**
- [Linux Kernel Programming](../Operating_System/Linux_Kernel_Programming.md)
- [Device Drivers](../Operating_System/Device_Drivers.md)
- [Real-time Linux](../Operating_System/Real_time_Linux.md)
- [Embedded Linux](../Operating_System/Embedded_Linux.md)
- [System Programming](../Operating_System/System_Programming.md)

## 📚 **Additional Resources**
- **Books**: "Linux Device Drivers" by Jonathan Corbet
- **Online**: [Linux Kernel Documentation](https://www.kernel.org/doc/)
- **Practice**: [Linux Kernel Newbies](https://kernelnewbies.org/)
- **Standards**: [POSIX Standards](https://pubs.opengroup.org/onlinepubs/9699919799/)
