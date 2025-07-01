# systemd-sparrow

Extending [systemd](https://github.com/systemd/systemd) with Sparrow6's power for dynamic, intelligent service management.

## Vision

This project explores integrating [Sparrow6](https://github.com/melezhik/Sparrow6) with systemd to address systemd's biggest pain points:
- Static configuration for dynamic requirements
- Duplicate code between similar services
- Complex dependency management
- Missing abstractions for common patterns

## Two Implementation Approaches

### 1. Translation Mode (Proposed by [@melezhik](https://github.com/melezhik))

Alexey's original concept: Use Raku/Sparrow6 to generate native systemd unit files.

```raku
#!raku
# File: /etc/systemd/system/multi-user.target.wants/beans.service

task-run "beans", "systemd", %(
    after-service => ["nginx", "mysql"],
    after-target => ["syslog", "network"],
    env => %(:RACK_ENV<production>),
    description => "Cool Beans",
    start => "/usr/local/bin/beans start -f",
    stop => "/usr/local/bin/beans stop",
    reload => "kill -s HUP /var/run/beans.pid",
);
```

**Flow:**
1. Systemd detects `#!raku` shebang
2. Spawns: `raku -MSparrow6::DSL -e /path/to/service`
3. Generates `/tmp/beans.unit.out` with native systemd format
4. Systemd uses the generated unit file

**Advantages:**
- ✅ Easy to implement (just spawn raku)
- ✅ Works with existing systemd
- ✅ Immediate configuration management wins
- ✅ Full Raku/Sparrow6 power at generation time

### 2. Runtime Integration Mode (Extended vision)

Going beyond static generation to true runtime dynamism with asynchronous processing.

```raku
#!raku
use Sparrow6::DSL;

task-run "my-webapp", "systemd-dynamic", %(
    description => "Dynamic Web Application",
    exec-start => "/usr/bin/webapp",
    
    # Dynamic resource allocation
    hooks => %(
        resource-check => sub {
            my %metrics = task-run "system-metrics", "resource-monitor";
            return %(
                memory-max => %metrics<free-memory> < 4.GB ?? "512M" !! "2G",
                cpu-quota => %metrics<cpu-load> > 80 ?? "50%" !! "200%",
            );
        },
        
        # Health monitoring with Sparrow6
        health-check => sub {
            task-run "webapp-health", "http-check", %(
                url => "http://localhost:8080/health",
                expect => %(status => 200)
            );
        }
    ),
    
    # Sparrow6 orchestration
    orchestration => %(
        startup-sequence => q:to/SEQUENCE/,
            begin:
                task-run "network-ready", "network-check"
                task-run "storage-mount", "mount-check", %( path => "/data" )
                task-run "config-sync", "consul-fetch", %( key => "webapp/config" )
            end:
        SEQUENCE
    )
);
```

## Comparison: Translation vs Runtime Integration

### Translation Approach (Phase 1)
- ✅ Easy C++ integration (just spawn `raku -e`)
- ✅ Works with existing systemd
- ✅ Immediate win for configuration management
- ❌ Static compilation - no runtime dynamism
- ❌ Can't react to changing system conditions
- ❌ Requires regeneration for changes

### Runtime Integration (Full vision)
- ✅ True dynamic behavior during service lifecycle
- ✅ Can adapt to system load, hardware changes, etc.
- ✅ Live system introspection and adaptation
- ✅ Non-blocking asynchronous monitoring
- ❌ Requires deeper systemd modifications
- ❌ Much more complex C++ integration

### Example of the Difference

**Translation approach generates static configs:**
```ini
[Service]
ExecStart=/usr/bin/myapp
MemoryMax=2048K  # Static value calculated at unit creation
```

**Runtime integration could evaluate dynamically:**
```raku
# Evaluated periodically in background threads
MemoryMax={ system-memory() < 4.GB ?? "512M" !! "2G" }
CPUQuota={ current-load() > 80 ?? "50%" !! "200%" }
```

## Most Promising Approach (Sparrow6-Powered)

From a pure Sparrow6 capability standpoint, here's what would create the most excitement:

```raku
#!raku
use Sparrow6::DSL;

# Define a service with Sparrow6's full orchestration power
task-run "my-webapp", "systemd-dynamic", %(
    # Basic service definition
    description => "Dynamic Web Application",
    exec-start => "/usr/bin/webapp",
    
    # Sparrow6 hooks for lifecycle management
    hooks => %(
        # Pre-start hook using Sparrow6 tasks
        pre-start => sub {
            # Check dependencies with custom logic
            task-run "check-db", "database-health", %( 
                timeout => 30,
                retry => 5 
            );
            
            # Conditional environment setup
            if task-run("detect-env", "environment-probe")<env> eq 'production' {
                task-run "setup-prod", "production-init";
            }
        },
        
        # Dynamic resource allocation hook
        resource-check => sub {
            my %metrics = task-run "system-metrics", "resource-monitor";
            
            return %(
                memory-max => %metrics<free-memory> < 4.GB ?? "512M" !! "2G",
                cpu-quota => %metrics<cpu-load> > 80 ?? "50%" !! "200%",
                io-weight => %metrics<io-pressure> ?? 100 !! 1000
            );
        },
        
        # Health monitoring with Sparrow6 tasks
        health-check => sub {
            my %health = task-run "webapp-health", "http-check", %(
                url => "http://localhost:8080/health",
                expect => %(status => 200, body => /'ok'/)
            );
            
            if !%health<success> {
                task-run "alert", "notification", %( 
                    message => "Service degraded", 
                    severity => "warning" 
                );
            }
            
            return %health<success>;
        }
    ),
    
    # Sparrow6 orchestration for complex dependencies
    orchestration => %(
        # Use Sparrow6's within/sequence expressions
        startup-sequence => q:to/SEQUENCE/,
            begin:
                task-run "network-ready", "network-check"
                task-run "storage-mount", "mount-check", %( path => "/data" )
                task-run "config-sync", "consul-fetch", %( key => "webapp/config" )
            end:
        SEQUENCE
        
        # Dynamic dependency resolution
        requires => sub {
            my @deps = <network.target>;
            
            # Add dependencies based on system state
            if task-run("check-feature", "feature-flags")<use-redis> {
                @deps.push: "redis.service";
            }
            
            if os() ~~ /centos/ {
                @deps.push: "firewalld.service";
            }
            
            return @deps;
        }
    ),
    
    # Asynchronous monitoring configuration
    monitoring => %(
        interval => 60,  # Check every 60 seconds
        priority => "background"  # Don't block systemd operations
    )
);
```

This approach leverages Sparrow6's strengths:
- **Task orchestration** for complex startup sequences
- **Plugin ecosystem** for reusable components
- **DSL power** for expressing dependencies
- **Cross-platform** abstractions (os() detection)
- **Built-in testing** through Sparrow6's test framework

## Key Benefits

### 1. Dynamic Configuration
```raku
# Instead of static values:
MemoryMax=2048K

# Dynamic evaluation:
MemoryMax => { system-memory() < 4.GB ?? "512M" !! "2G" }
```

### 2. Reusable Service Definitions
```raku
# Simple, reusable service definitions
task-run "nginx service", "systemd-nginx";
task-run "mariadb service", "systemd-mariadb";
```

### 3. Complex Orchestration
Using Sparrow6's task orchestration capabilities for sophisticated startup sequences, health checks, and dependency management.

### 4. Cross-Platform Abstractions
```raku
# Sparrow6's os() detection for platform-specific logic
if os() ~~ /centos/ {
    @deps.push: "firewalld.service";
}
```

### 5. Non-Blocking Asynchronous Operations
All Sparrow6 evaluations run in background threads, ensuring systemd's core operations continue at full speed.

## Implementation Roadmap

### Phase 1: Translation Mode
- [ ] Create `systemd` Sparrow6 plugin
- [ ] Implement basic unit file generation
- [ ] Add systemd hook for `#!raku` detection
- [ ] Create plugins for common services (nginx, postgresql, etc.)

### Phase 2: Enhanced Translation
- [ ] Template support for service families
- [ ] Conditional logic compilation
- [ ] SparrowHub integration for shared service definitions

### Phase 3: Hybrid Mode
- [ ] Runtime property evaluation hooks
- [ ] Dynamic health check integration
- [ ] Resource limit adaptation
- [ ] Integrate Cbeam for asynchronous monitoring

### Phase 4: Full Runtime Integration
- [ ] Complete Sparrow6 runtime bridge
- [ ] Event-driven task orchestration
- [ ] Live system adaptation
- [ ] Full asynchronous architecture with Cbeam

## Technical Design

### C++ Integration (Phase 1)
```cpp
// Simple approach: detect and translate Sparrow6 units
bool is_sparrow6_unit(const std::filesystem::path& path) {
    std::ifstream file(path);
    std::string first_line;
    std::getline(file, first_line);
    return first_line == "#!raku" || first_line == "#!sparrow6";
}

if (is_sparrow6_unit(unit_path)) {
    system("raku -MSparrow6::DSL -e " + unit_path);
    // Use generated unit file...
}
```

### C++ Integration Design (Full Runtime)

#### 1. Sparrow6 Runtime Bridge
```cpp
// Sparrow6-specific runtime integration
class Sparrow6Extension {
private:
    std::unique_ptr<RakuRuntime> raku;
    std::filesystem::path sparrow_cache;
    
public:
    // Initialize Sparrow6 environment
    void initialize() {
        // Set up Sparrow6 environment variables
        setenv("SP6_REPO", "https://sparrowhub.io", 1);
        setenv("SP6_TASK_ROOT", "/etc/systemd/sparrow-tasks", 1);
        
        // Initialize Raku with Sparrow6 modules
        raku = std::make_unique<RakuRuntime>();
        raku->use_module("Sparrow6::DSL");
    }
    
    // Execute Sparrow6 task and return results
    TaskResult run_task(const std::string& task_name, 
                       const std::string& plugin_name,
                       const ServiceContext& context) {
        
        // Build parameters from service context
        auto params = build_sparrow_params(context);
        
        // Execute through Sparrow6 API
        std::string code = fmt::format(
            R"(
            my %result = task-run "{}", "{}", {};
            %result
            )",
            task_name, plugin_name, params
        );
        
        return raku->evaluate<TaskResult>(code);
    }
    
    // Hook for dynamic property evaluation
    std::string evaluate_property(const std::string& property_code,
                                 const ServiceContext& context) {
        // Make context available to Raku code
        inject_context(context);
        
        return raku->evaluate_string(property_code);
    }
};
```

#### 2. Systemd Service Extension
```cpp
// Extend systemd's Service class to support Sparrow6 hooks
class Sparrow6Service : public Service {
private:
    Sparrow6Extension sparrow;
    std::map<std::string, std::string> sparrow_hooks;
    
public:
    // Override lifecycle methods to call Sparrow6 hooks
    Result<void> start() override {
        // Run pre-start Sparrow6 tasks
        if (sparrow_hooks.contains("pre-start")) {
            auto result = sparrow.run_inline_task(
                sparrow_hooks["pre-start"], 
                get_context()
            );
            
            if (!result.success) {
                return Error("Pre-start hook failed: " + result.error);
            }
        }
        
        // Continue with normal start
        return Service::start();
    }
    
    // Dynamic resource limit evaluation
    uint64_t get_memory_limit() override {
        if (sparrow_hooks.contains("resource-check")) {
            auto resources = sparrow.run_inline_task(
                sparrow_hooks["resource-check"],
                get_context()
            );
            
            return parse_memory(resources["memory-max"]);
        }
        
        return Service::get_memory_limit();
    }
    
    // Health check through Sparrow6
    bool is_healthy() override {
        if (sparrow_hooks.contains("health-check")) {
            auto health = sparrow.run_inline_task(
                sparrow_hooks["health-check"],
                get_context()
            );
            
            return health["success"].to_bool();
        }
        
        return Service::is_healthy();
    }
};
```

#### 3. Unit File Parser Extension
```cpp
// Extend unit file parsing to support Sparrow6 expressions
class Sparrow6UnitParser : public UnitParser {
public:
    std::unique_ptr<Unit> parse(const std::filesystem::path& path) override {
        // Check for Sparrow6 shebang
        if (is_sparrow6_unit(path)) {
            return parse_sparrow6_unit(path);
        }
        
        // Fall back to standard parsing
        return UnitParser::parse(path);
    }
    
private:
    bool is_sparrow6_unit(const std::filesystem::path& path) {
        std::ifstream file(path);
        std::string first_line;
        std::getline(file, first_line);
        return first_line == "#!raku" || first_line == "#!sparrow6";
    }
    
    std::unique_ptr<Unit> parse_sparrow6_unit(const std::filesystem::path& path) {
        // Execute Sparrow6 code to generate unit definition
        Sparrow6Extension sparrow;
        
        // For translation mode: generate static unit
        if (translation_mode) {
            auto unit_data = sparrow.generate_unit(path);
            return parse_generated_unit(unit_data);
        }
        
        // For runtime mode: create dynamic unit
        return std::make_unique<Sparrow6Service>(path, sparrow);
    }
};
```

#### 4. Kernel-Based Resource Monitoring

To achieve truly efficient resource monitoring without polling, we leverage Linux kernel capabilities for event-driven notifications:

##### PSI (Pressure Stall Information) Integration
Modern kernels (4.20+) provide pressure metrics that we can monitor efficiently:

```cpp
// Message types for kernel events
struct PSIEvent {
    std::string service_id;
    PSIMetrics metrics;
    int psi_fd;
    int epoll_fd;
};

struct EpollWatchRequest {
    std::string service_id;
    std::string pressure_file;
    PSIConfig config;
};

class PSIMonitor {
private:
    cbeam::concurrency::message_manager<std::variant<PSIEvent, EpollWatchRequest>> monitor;
    
    static constexpr size_t EPOLL_WATCHER_ID = 1;
    static constexpr size_t PSI_PROCESSOR_ID = 2;
    static constexpr size_t SPARROW_TRIGGER_ID = 3;
    
public:
    PSIMonitor() {
        // Handler that sets up epoll watching for PSI files
        monitor.add_handler(EPOLL_WATCHER_ID, [this](auto msg) {
            std::visit([this](auto&& event) {
                using T = std::decay_t<decltype(event)>;
                if constexpr (std::is_same_v<T, EpollWatchRequest>) {
                    setup_psi_watch(event);
                }
            }, msg);
        }, nullptr, nullptr, "psi_epoll_watcher");
        
        // Handler that processes PSI events
        monitor.add_handler(PSI_PROCESSOR_ID, [this](auto msg) {
            std::visit([this](auto&& event) {
                using T = std::decay_t<decltype(event)>;
                if constexpr (std::is_same_v<T, PSIEvent>) {
                    process_psi_event(event);
                }
            }, msg);
        }, nullptr, nullptr, "psi_processor");
    }
    
    void monitor_service(const std::string& service_id, const PSIConfig& config) {
        // Send request to set up monitoring
        monitor.send_message(EPOLL_WATCHER_ID, 
            EpollWatchRequest{service_id, "/proc/pressure/memory", config});
    }
    
private:
    void setup_psi_watch(const EpollWatchRequest& req) {
        // Set up PSI monitoring
        int psi_fd = open(req.pressure_file.c_str(), O_RDWR | O_NONBLOCK);
        
        // Configure threshold
        std::string threshold = fmt::format("{} {} {}", 
            req.config.level,           // "some" or "full"
            req.config.threshold_us,    // stall time threshold
            req.config.window_us        // time window
        );
        
        write(psi_fd, threshold.c_str(), threshold.length());
        
        // Set up epoll
        int epoll_fd = epoll_create1(EPOLL_CLOEXEC);
        struct epoll_event ev;
        ev.events = EPOLLPRI;
        epoll_ctl(epoll_fd, EPOLL_CTL_ADD, psi_fd, &ev);
        
        // Poll in a loop and send events as messages
        struct epoll_event events[1];
        while (true) {
            int nfds = epoll_wait(epoll_fd, events, 1, -1);
            
            if (nfds > 0) {
                PSIMetrics metrics = read_psi_metrics(psi_fd);
                monitor.send_message(PSI_PROCESSOR_ID, 
                    PSIEvent{req.service_id, metrics, psi_fd, epoll_fd});
            }
        }
    }
    
    void process_psi_event(const PSIEvent& event) {
        // Forward to Sparrow6 handler
        monitor.send_message(SPARROW_TRIGGER_ID, event);
    }
    
    PSIMetrics read_psi_metrics(int fd) {
        char buffer[256];
        pread(fd, buffer, sizeof(buffer), 0);
        // Parse PSI format: "some avg10=0.00 avg60=0.00 avg300=0.00 total=0"
        return parse_psi_output(buffer);
    }
};
```

##### cgroups v2 Memory Events
For fine-grained memory monitoring per service:

```cpp
// Message types for memory monitoring
struct MemoryWatchRequest {
    std::string service_id;
    MemoryConfig config;
};

struct MemoryEvent {
    std::string service_id;
    uint64_t current_bytes;
    uint64_t limit_bytes;
    bool is_oom_imminent;
};

class CgroupMemoryMonitor {
private:
    cbeam::concurrency::message_manager<std::variant<MemoryWatchRequest, MemoryEvent>> monitor;
    
    static constexpr size_t MEMORY_WATCHER_ID = 1;
    static constexpr size_t MEMORY_EVENT_ID = 2;
    
public:
    CgroupMemoryMonitor() {
        // Handler that sets up memory monitoring for a service
        monitor.add_handler(MEMORY_WATCHER_ID, [this](auto msg) {
            std::visit([this](auto&& req) {
                using T = std::decay_t<decltype(req)>;
                if constexpr (std::is_same_v<T, MemoryWatchRequest>) {
                    setup_memory_watch(req);
                }
            }, msg);
        }, nullptr, nullptr, "memory_watcher");
    }
    
    void monitor_service_memory(const std::string& service_id, const MemoryConfig& config) {
        monitor.send_message(MEMORY_WATCHER_ID, 
            MemoryWatchRequest{service_id, config});
    }
    
private:
    void setup_memory_watch(const MemoryWatchRequest& req) {
        // Get service's cgroup path
        std::string cgroup_path = fmt::format("/sys/fs/cgroup/system.slice/{}.service", 
                                             req.service_id);
        
        // Monitor memory.events for OOM and high pressure
        int mem_events_fd = open((cgroup_path + "/memory.events").c_str(), O_RDONLY);
        int event_fd = eventfd(0, EFD_CLOEXEC);
        
        // Set up memory.high threshold monitoring
        if (req.config.high_threshold) {
            std::string threshold_path = cgroup_path + "/memory.pressure";
            int threshold_fd = open(threshold_path.c_str(), O_WRONLY);
            
            // Configure pressure monitoring
            std::string config_str = fmt::format("some {} {}", 
                req.config.high_threshold, req.config.window_us);
            write(threshold_fd, config_str.c_str(), config_str.length());
            close(threshold_fd);
        }
        
        // Set up epoll for event notification
        int epoll_fd = epoll_create1(EPOLL_CLOEXEC);
        struct epoll_event ev;
        ev.events = EPOLLIN;
        ev.data.fd = event_fd;
        epoll_ctl(epoll_fd, EPOLL_CTL_ADD, event_fd, &ev);
        
        // Monitor in epoll loop
        struct epoll_event events[1];
        while (true) {
            int nfds = epoll_wait(epoll_fd, events, 1, -1);
            
            if (nfds > 0) {
                uint64_t counter;
                read(event_fd, &counter, sizeof(counter));
                
                // Read current memory usage
                auto stats = read_memory_stats(cgroup_path);
                
                // Send memory event message
                monitor.send_message(MEMORY_EVENT_ID, 
                    MemoryEvent{
                        req.service_id, 
                        stats.current, 
                        stats.limit,
                        stats.current > stats.limit * 0.9
                    });
            }
        }
    }
    
    MemoryStats read_memory_stats(const std::string& cgroup_path) {
        // Read memory.current and memory.max
        MemoryStats stats;
        std::ifstream current(cgroup_path + "/memory.current");
        std::ifstream limit(cgroup_path + "/memory.max");
        current >> stats.current;
        limit >> stats.limit;
        return stats;
    }
};
```

##### Integrated Monitoring Architecture
Combining kernel events with Sparrow6 hooks using Cbeam's message-driven architecture:

```cpp
// Unified message type for all kernel events
using KernelEvent = std::variant<
    PSIEvent,
    MemoryEvent,
    ResourceAdjustmentRequest,
    Sparrow6TaskRequest
>;

class KernelEventMonitor {
private:
    cbeam::concurrency::message_manager<KernelEvent> event_manager;
    PSIMonitor psi_monitor;
    CgroupMemoryMonitor cgroup_monitor;
    Sparrow6Extension sparrow;
    
    static constexpr size_t KERNEL_EVENT_ID = 1;
    static constexpr size_t SPARROW_EXECUTOR_ID = 2;
    static constexpr size_t RESOURCE_ADJUSTER_ID = 3;
    
public:
    KernelEventMonitor() {
        // Handler for all kernel events
        event_manager.add_handler(KERNEL_EVENT_ID, [this](KernelEvent event) {
            std::visit([this](auto&& e) {
                handle_kernel_event(e);
            }, event);
        }, nullptr, nullptr, "kernel_event_handler");
        
        // Handler for Sparrow6 task execution
        event_manager.add_handler(SPARROW_EXECUTOR_ID, [this](KernelEvent event) {
            std::visit([this](auto&& e) {
                using T = std::decay_t<decltype(e)>;
                if constexpr (std::is_same_v<T, Sparrow6TaskRequest>) {
                    execute_sparrow6_task(e);
                }
            }, event);
        }, nullptr, nullptr, "sparrow6_executor");
        
        // Handler for resource adjustments
        event_manager.add_handler(RESOURCE_ADJUSTER_ID, [this](KernelEvent event) {
            std::visit([this](auto&& e) {
                using T = std::decay_t<decltype(e)>;
                if constexpr (std::is_same_v<T, ResourceAdjustmentRequest>) {
                    apply_resource_adjustment(e);
                }
            }, event);
        }, nullptr, nullptr, "resource_adjuster");
    }
    
    void setup_monitoring(const std::string& service_id, const MonitoringConfig& config) {
        // Configure PSI monitoring
        if (config.enable_psi) {
            psi_monitor.monitor_service(service_id, config.psi_config);
        }
        
        // Configure cgroup-specific memory monitoring
        if (config.enable_cgroup_memory) {
            cgroup_monitor.monitor_service_memory(service_id, config.memory_config);
        }
    }
    
private:
    template<typename T>
    void handle_kernel_event(const T& event) {
        if constexpr (std::is_same_v<T, PSIEvent>) {
            // PSI pressure event - create Sparrow6 task request
            event_manager.send_message(SPARROW_EXECUTOR_ID,
                Sparrow6TaskRequest{
                    "pressure-response",
                    "resource-adjust",
                    %(
                        service => event.service_id,
                        pressure => event.metrics,
                        type => "memory"
                    )
                });
        } else if constexpr (std::is_same_v<T, MemoryEvent>) {
            // Memory event - create Sparrow6 task request
            event_manager.send_message(SPARROW_EXECUTOR_ID,
                Sparrow6TaskRequest{
                    "memory-pressure",
                    "oom-prevention",
                    %(
                        service => event.service_id,
                        current_usage => event.current_bytes,
                        limit => event.limit_bytes,
                        oom_imminent => event.is_oom_imminent
                    )
                });
        }
    }
    
    void execute_sparrow6_task(const Sparrow6TaskRequest& req) {
        // Execute Sparrow6 task and get adjustment
        auto result = sparrow.run_task(req.task_name, req.plugin_name, req.params);
        
        // Send adjustment request if needed
        if (result.requires_adjustment) {
            event_manager.send_message(RESOURCE_ADJUSTER_ID,
                ResourceAdjustmentRequest{
                    req.params["service"],
                    result.adjustments
                });
        }
    }
    
    void apply_resource_adjustment(const ResourceAdjustmentRequest& req) {
        // Apply systemd resource adjustments
        auto& service = get_service(req.service_id);
        
        if (req.adjustments.contains("memory-max")) {
            service.set_memory_limit(parse_memory(req.adjustments["memory-max"]));
        }
        
        if (req.adjustments.contains("cpu-quota")) {
            service.set_cpu_quota(parse_percentage(req.adjustments["cpu-quota"]));
        }
        
        // Log the adjustment
        log_resource_adjustment(req);
    }
};
```

This architecture provides true message-driven kernel event processing:
- **No Manual Thread Creation**: All threading handled by `message_manager`
- **Clean Separation**: Different handlers for different responsibilities
- **Scalable**: Add more handlers to distribute load across cores
- **Type-Safe**: `std::variant` ensures compile-time safety
- **Testable**: Each handler can be tested independently

This kernel-based approach provides:
- **Zero CPU overhead**: No polling, kernel wakes us only when thresholds are exceeded
- **Microsecond latency**: Direct kernel notifications vs. periodic checks
- **System-aware decisions**: PSI provides holistic system pressure information
- **Per-service precision**: cgroups v2 gives exact per-service metrics
- **Proactive management**: Act before OOM killer or service degradation

#### 5. Asynchronous Resource Monitoring with Cbeam

To enable true dynamic resource management without blocking systemd's core operations, we use [Cbeam's message_manager](https://cbeam.org/message_manager) for asynchronous processing:

```cpp
// Message types for resource monitoring
struct ResourceCheckRequest {
    std::string service_id;
    std::string plugin;
    std::map<std::string, std::string> params;
};

struct ResourceUpdateResult {
    std::string service_id;
    bool requires_update;
    std::map<std::string, std::string> properties;
};

struct TimerTick {
    std::chrono::steady_clock::time_point timestamp;
};

class Sparrow6AsyncMonitor {
private:
    using MonitorMessage = std::variant<ResourceCheckRequest, ResourceUpdateResult, TimerTick>;
    cbeam::concurrency::message_manager<MonitorMessage> monitor;
    Sparrow6Extension sparrow;
    
    static constexpr size_t RESOURCE_CHECK_ID = 1;
    static constexpr size_t RESULT_PROCESSOR_ID = 2;
    static constexpr size_t TIMER_ID = 3;
    
public:
    void initialize() {
        // Handler for resource check requests - runs Sparrow6 tasks
        monitor.add_handler(RESOURCE_CHECK_ID, [this](MonitorMessage msg) {
            std::visit([this](auto&& req) {
                using T = std::decay_t<decltype(req)>;
                if constexpr (std::is_same_v<T, ResourceCheckRequest>) {
                    // Execute Sparrow6 task asynchronously
                    auto result = sparrow.run_task("resource-monitor", 
                                                  req.plugin, req.params);
                    
                    // Send result to processor
                    monitor.send_message(RESULT_PROCESSOR_ID,
                        ResourceUpdateResult{
                            req.service_id,
                            result.requires_update,
                            result.properties
                        });
                }
            }, msg);
        }, nullptr, nullptr, "sparrow6_resource_check");
        
        // Handler for processing results and applying updates
        monitor.add_handler(RESULT_PROCESSOR_ID, [this](MonitorMessage msg) {
            std::visit([this](auto&& result) {
                using T = std::decay_t<decltype(result)>;
                if constexpr (std::is_same_v<T, ResourceUpdateResult>) {
                    if (result.requires_update && 
                        meets_change_threshold(result.service_id, result)) {
                        update_service_properties(result.service_id, result.properties);
                    }
                }
            }, msg);
        }, nullptr, nullptr, "result_processor");
        
        // Timer handler sends periodic check requests
        monitor.add_handler(TIMER_ID, [this](MonitorMessage msg) {
            std::visit([this](auto&& tick) {
                using T = std::decay_t<decltype(tick)>;
                if constexpr (std::is_same_v<T, TimerTick>) {
                    for (auto& [service_id, config] : monitored_services) {
                        if (should_check(service_id, tick.timestamp)) {
                            monitor.send_message(RESOURCE_CHECK_ID, 
                                ResourceCheckRequest{service_id, config.plugin, 
                                                   config.params});
                        }
                    }
                }
            }, msg);
        }, nullptr, nullptr, "sparrow6_timer");
        
        // Start timer loop
        start_timer_loop();
    }
    
    void register_service(const std::string& service_id, 
                         const MonitoringConfig& config) {
        monitored_services[service_id] = config;
    }
    
private:
    std::map<std::string, MonitoringConfig> monitored_services;
    
    void start_timer_loop() {
        // Send timer ticks every second
        monitor.add_handler(TIMER_ID + 1, [this](MonitorMessage) {
            while (true) {
                std::this_thread::sleep_for(std::chrono::seconds(1));
                monitor.send_message(TIMER_ID, 
                    TimerTick{std::chrono::steady_clock::now()});
            }
        }, nullptr, nullptr, "timer_generator");
        
        // Kickstart the timer
        monitor.send_message(TIMER_ID + 1, TimerTick{});
    }
    
    bool meets_change_threshold(const std::string& service_id,
                               const ResourceUpdateResult& result) {
        // Implement hysteresis to prevent oscillation
        auto& config = monitored_services[service_id];
        // Compare with configured thresholds
        return true; // simplified
    }
};
```

This message-driven architecture provides:
- **Non-blocking Operation**: All Sparrow6 evaluations happen in handler threads managed by `message_manager`
- **No Manual Threading**: `message_manager` handles all thread lifecycle
- **Configurable Parallelism**: Add more handlers for the same ID to scale across cores
- **Message-Based Communication**: Clean decoupling between components
- **Zero systemd Latency**: Main systemd operations continue unimpeded

## Why This Would Be Revolutionary

This approach would solve systemd's biggest pain points:
- **Dynamic adaptation** instead of static configs - Resources adjust to runtime conditions
- **Reusable components** through Sparrow6 plugins
- **Testing built-in** via Sparrow6's test framework
- **Cross-distro compatibility** through Sparrow6's abstractions
- **Gradual adoption** - start with translation, evolve to runtime
- **True asynchronous monitoring** - System conditions evaluated continuously without blocking
- **Kernel-native efficiency** - Zero-overhead monitoring through PSI and cgroups v2

The Sparrow6 ecosystem would provide immediate value through:
- Pre-built service definitions on SparrowHub
- Community-contributed health checks and monitors
- Battle-tested dependency management patterns
- Integration with existing Sparrow6 configuration management

The integration of kernel monitoring enables:
- **Real-time responsiveness**: Microsecond-latency reactions to system pressure
- **Proactive resource management**: Prevent OOM and service degradation before they occur
- **System-wide awareness**: PSI provides holistic view of resource contention
- **Zero polling overhead**: Kernel notifies only when action is needed

The integration of Cbeam enables:
- **Advanced monitoring capabilities**: Pattern matching in logs, network condition monitoring, external API integration
- **Predictive scaling**: Use historical data to anticipate resource needs
- **Complex decision logic**: Implement sophisticated algorithms without performance penalties
- **Reliable change management**: Hysteresis and thresholds prevent service instability

## Examples

### Basic Service
```raku
#!raku
task-run "webserver", "systemd", %(
    description => "My Web Server",
    exec-start => "/usr/bin/myapp",
    restart => "always",
    after-service => ["network.target"],
);
```

### Service with Conditional Logic
```raku
#!raku
my $env = %*ENV<DEPLOYMENT_ENV> // 'development';

task-run "api-server", "systemd", %(
    description => "API Server ({$env})",
    exec-start => "/usr/bin/api-server",
    
    env => %(
        NODE_ENV => $env,
        WORKERS => $env eq 'production' ?? 8 !! 2,
        LOG_LEVEL => $env eq 'production' ?? 'error' !! 'debug',
    ),
    
    memory-max => $env eq 'production' ?? '4G' !! '1G',
    
    after-service => do {
        my @deps = ["network.target"];
        @deps.push("postgresql.service") if $env eq 'production';
        @deps.push("redis.service") if $env ~~ /staging|production/;
        @deps;
    },
);
```

### Service with Health Checks
```raku
#!raku
task-run "microservice", "systemd-with-health", %(
    exec-start => "/usr/bin/microservice",
    
    health-check => %(
        type => "http",
        url => "http://localhost:3000/health",
        interval => 30,
        timeout => 5,
        
        on-failure => sub {
            task-run "alert", "slack-notification", %(
                channel => "#ops",
                message => "Microservice health check failed"
            );
        }
    ),
);
```

### Service with Dynamic Memory Management
```raku
#!raku
task-run "webapp", "systemd-dynamic", %(
    description => "Web Application with Dynamic Resources",
    exec-start => "/usr/bin/webapp",
    
    # Initial static configuration
    memory-max => "2G",
    cpu-quota => "100%",
    
    # Asynchronous monitoring configuration
    monitoring => %(
        interval => 60,  # Check every 60 seconds
        
        resource-check => sub {
            my $available = get-available-memory();
            my $load = get-system-load();
            my $connections = get-active-connections();
            
            # Complex logic without blocking systemd
            my $optimal-memory = do {
                when $available < 2.GB { "512M" }
                when $available < 4.GB { "1G" }
                when $connections > 1000 { "4G" }
                default { "2G" }
            };
            
            return %(
                memory-max => $optimal-memory,
                cpu-quota => $load > 80 ?? "50%" !! "200%",
                io-weight => $connections > 500 ?? 100 !! 1000
            );
        },
        
        # Hysteresis to prevent oscillation
        change-threshold => %(
            memory-max => "100M",  # Only update if change > 100M
            cpu-quota => "20%"     # Only update if change > 20%
        )
    )
);
```

### Service with Kernel-Based Pressure Monitoring
```raku
#!raku
task-run "database-service", "systemd-pressure-aware", %(
    exec-start => "/usr/bin/postgres",
    
    # Enable kernel-based monitoring
    kernel-monitoring => %(
        # PSI monitoring for system-wide pressure
        psi => %(
            enabled => True,
            memory => %(
                level => "some",      # Trigger on some tasks stalled
                threshold => 150000,  # 150ms stall time
                window => 1000000     # in 1 second window
            ),
            cpu => %(
                level => "full",      # Trigger on all tasks stalled
                threshold => 100000,  # 100ms stall time
                window => 1000000     # in 1 second window
            )
        ),
        
        # cgroup v2 memory events
        cgroup-memory => %(
            enabled => True,
            high-threshold => "80%",  # Alert at 80% of limit
            oom-score => -500        # Protect from OOM killer
        )
    ),
    
    # Sparrow6 hooks triggered by kernel events
    pressure-response => sub ($event) {
        given $event<type> {
            when 'memory-pressure' {
                # Reduce cache size to free memory
                task-run "postgres-tune", "cache-adjust", %(
                    shared_buffers => "128MB",
                    effective_cache_size => "512MB"
                );
            }
            when 'cpu-pressure' {
                # Reduce concurrent connections
                task-run "postgres-tune", "connection-limit", %(
                    max_connections => 50
                );
            }
            when 'memory-high' {
                # Emergency memory reduction
                task-run "postgres-tune", "emergency-gc";
            }
        }
    }
);
```

### Service with Pattern-Based Monitoring
```raku
#!raku
task-run "security-service", "systemd-pattern-monitor", %(
    exec-start => "/usr/bin/security-app",
    
    monitoring => %(
        interval => 30,
        
        # Monitor logs for security patterns
        pattern-check => sub {
            my $log-content = slurp("/var/log/security-app/access.log");
            
            # Use Raku's powerful regex engine
            my $suspicious = $log-content ~~ m:g/
                '401' .* 'admin' |
                'DROP' .* 'TABLE' |
                '../' ** 3..*
            /;
            
            if $suspicious.elems > 10 {
                task-run "alert", "security-notification", %(
                    severity => "high",
                    matches => $suspicious.elems
                );
                
                # Activate defensive mode
                return %(
                    environment => %(
                        SECURITY_MODE => "restrictive",
                        RATE_LIMIT => "aggressive"
                    )
                );
            }
            
            return %();  # No changes needed
        }
    )
);
```

### Service with Adaptive Resource Management
```raku
#!raku
task-run "ml-inference", "systemd-adaptive", %(
    exec-start => "/usr/bin/ml-server",
    
    # Combine kernel monitoring with Sparrow6 intelligence
    adaptive-resources => %(
        # Let kernel tell us about pressure
        kernel-signals => True,
        
        # Sparrow6 decides how to respond
        adaptation-strategy => sub ($signals) {
            my %response;
            
            # Memory pressure detected by PSI
            if $signals<psi><memory><avg10> > 20 {
                %response<memory-strategy> = do {
                    # Check what's using memory
                    my $model-size = task-run "ml-metrics", "model-memory";
                    my $cache-size = task-run "ml-metrics", "cache-memory";
                    
                    # Intelligent decision making
                    if $cache-size > $model-size * 0.5 {
                        # Cache is too large, reduce it
                        task-run "ml-config", "reduce-cache", %(
                            max-cache => $model-size * 0.3
                        );
                        "cache-reduced"
                    } else {
                        # Offload to disk if possible
                        task-run "ml-config", "enable-offloading";
                        "offloading-enabled"
                    }
                };
            }
            
            # CPU pressure detected
            if $signals<psi><cpu><avg10> > 30 {
                %response<cpu-strategy> = do {
                    # Reduce batch size for lower latency
                    my $current-batch = task-run "ml-metrics", "batch-size";
                    task-run "ml-config", "set-batch-size", %(
                        size => max(1, $current-batch / 2)
                    );
                    "batch-size-reduced"
                };
            }
            
            return %response;
        }
    )
);
```

## Contributing

This is a collaborative project between [@melezhik](https://github.com/melezhik) and [@acrion](https://github.com/acrion). We're looking for contributors interested in:
- Creating Sparrow6 plugins for common services
- Implementing the C++ integration layer (especially kernel event handling)
- Testing on different Linux distributions
- Documenting patterns and best practices
- Developing kernel monitoring integrations
