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

#### 4. Asynchronous Resource Monitoring with Cbeam

To enable true dynamic resource management without blocking systemd's core operations, we integrate [Cbeam's message_manager](https://cbeam.org/message_manager) for asynchronous processing:

```cpp
class Sparrow6AsyncMonitor {
private:
    cbeam::concurrency::message_manager<ResourceCheckRequest> monitor;
    Sparrow6Extension sparrow;
    
    static constexpr size_t RESOURCE_CHECK_ID = 1;
    static constexpr size_t TIMER_ID = 2;
    
public:
    void initialize() {
        // Resource check handler runs in background thread
        monitor.add_handler(RESOURCE_CHECK_ID, [this](ResourceCheckRequest req) {
            // Execute Sparrow6 task asynchronously
            auto result = sparrow.run_task("resource-monitor", req.plugin, req.params);
            
            // Apply changes through systemd API if needed
            if (result.requires_update && meets_change_threshold(req, result)) {
                update_service_properties(req.service_id, result.properties);
            }
        }, nullptr, nullptr, "sparrow6_resource_check");
        
        // Timer handler sends periodic check requests
        monitor.add_handler(TIMER_ID, [this](TimerTick tick) {
            for (auto& [service_id, config] : monitored_services) {
                if (should_check(service_id, tick)) {
                    monitor.send_message(RESOURCE_CHECK_ID, 
                        ResourceCheckRequest{service_id, config});
                }
            }
        }, nullptr, nullptr, "sparrow6_timer");
    }
    
    void register_service(const std::string& service_id, 
                         const MonitoringConfig& config) {
        monitored_services[service_id] = config;
    }
    
private:
    std::map<std::string, MonitoringConfig> monitored_services;
    
    bool meets_change_threshold(const ResourceCheckRequest& req,
                               const TaskResult& result) {
        // Implement hysteresis to prevent oscillation
        // Compare with configured thresholds
        return true; // simplified
    }
};
```

This asynchronous architecture provides:
- **Non-blocking Operation**: All Sparrow6 evaluations happen in background threads
- **Configurable Check Intervals**: Services can specify their monitoring frequency
- **Multi-core Utilization**: Checks distribute across available CPU cores
- **Zero systemd Latency**: Main systemd operations continue unimpeded

## Why This Would Be Revolutionary

This approach would solve systemd's biggest pain points:
- **Dynamic adaptation** instead of static configs - Resources adjust to runtime conditions
- **Reusable components** through Sparrow6 plugins
- **Testing built-in** via Sparrow6's test framework
- **Cross-distro compatibility** through Sparrow6's abstractions
- **Gradual adoption** - start with translation, evolve to runtime
- **True asynchronous monitoring** - System conditions evaluated continuously without blocking

The Sparrow6 ecosystem would provide immediate value through:
- Pre-built service definitions on SparrowHub
- Community-contributed health checks and monitors
- Battle-tested dependency management patterns
- Integration with existing Sparrow6 configuration management

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

## Contributing

This is a collaborative project between [@melezhik](https://github.com/melezhik) and [@acrion](https://github.com/acrion). We're looking for contributors interested in:
- Creating Sparrow6 plugins for common services
- Implementing the C++ integration layer
- Testing on different Linux distributions
- Documenting patterns and best practices
