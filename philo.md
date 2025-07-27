# Philosophers Project - Complete Manual Testing Guide

## Project Overview
The Philosophers project simulates the classic dining philosophers problem using threads and mutexes. Each philosopher alternates between eating, thinking, and sleeping, while sharing forks with adjacent philosophers.

## Program Usage
```bash
./philo number_of_philosophers time_to_die time_to_eat time_to_sleep [number_of_times_each_philosopher_must_eat]
```

## Pre-Test Setup Checklist

### ✅ Compilation Tests
1. **Make compilation**: `make` should compile without errors or warnings
2. **Clean compilation**: `make clean` should remove object files
3. **Full clean**: `make fclean` should remove all generated files
4. **Recompilation**: `make re` should clean and recompile

### ✅ Code Quality Checks
1. **Norm compliance**: Check with `norminette` (if available)
2. **No global variables** (except for program termination flags if needed)
3. **Proper header files** and function organization
4. **Memory leaks**: Test with `valgrind --tool=helgrind ./philo [args]`

## Test Categories

## 1. Basic Functionality Tests

### Test 1.1: Valid Input Parsing
```bash
# Test basic valid inputs
./philo 4 410 200 200
./philo 5 800 200 200 7
./philo 2 400 100 100
```
**Expected**: Program should start and run without immediate errors.

### Test 1.2: Philosopher Actions
```bash
./philo 4 410 200 200
```
**Check for**:
- Philosophers take forks (should show "has taken a fork" twice per philosopher)
- Philosophers eat (should show "is eating")
- Philosophers sleep (should show "is sleeping")
- Philosophers think (should show "is thinking")
- Proper timestamp formatting

### Test 1.3: Fork Management
```bash
./philo 3 410 200 200
```
**Verify**:
- Each philosopher picks up exactly 2 forks before eating
- Forks are released after eating
- No philosopher eats with less than 2 forks

## 2. Error Handling Tests

### Test 2.1: Invalid Arguments
```bash
# Too few arguments
./philo 4 410 200

# Too many arguments
./philo 4 410 200 200 7 extra

# Invalid numbers
./philo -1 410 200 200
./philo 4 -410 200 200
./philo 4 410 -200 200
./philo 0 410 200 200
./philo abc 410 200 200
```
**Expected**: Program should display error message and exit gracefully.

### Test 2.2: Edge Case Arguments
```bash
# Very large numbers
./philo 1000 410 200 200
./philo 4 999999 200 200

# Very small positive numbers
./philo 1 1 1 1
./philo 2 60 1 1
```
**Expected**: Should handle gracefully or provide appropriate error messages.

## 3. Death Detection Tests

### Test 3.1: Single Philosopher Death
```bash
# Philosopher should die (can't eat alone)
./philo 1 400 200 200
```
**Expected**: Should print death message within ~400ms and terminate.

### Test 3.2: Multiple Philosophers Death Scenarios
```bash
# Designed to cause death
./philo 4 310 200 200
./philo 2 300 400 100
```
**Expected**: 
- Death should be detected within time_to_die ms after last meal
- Program should terminate immediately after death
- Death message format: "timestamp_in_ms X died"

### Test 3.3: Death Timing Precision
```bash
./philo 4 410 200 200
```
**Manual Check**:
- Monitor timestamps carefully
- Death should occur at timestamp_of_last_meal + time_to_die (±10ms tolerance)
- No actions should be printed after death

## 4. Synchronization Tests

### Test 4.1: No Data Races
```bash
# Run with race detection tools
valgrind --tool=helgrind ./philo 4 410 200 200
# or if available:
# ./philo 4 410 200 200 | grep -E "(died|eating)" | sort -n
```
**Expected**: No data race warnings, clean execution.

### Test 4.2: Deadlock Prevention
```bash
# Even number of philosophers (higher deadlock risk)
./philo 4 800 200 200
./philo 6 800 200 200
./philo 8 800 200 200

# Run for extended periods
timeout 30s ./philo 4 800 200 200
```
**Expected**: Should run without hanging or deadlocking.

### Test 4.3: Message Ordering
```bash
./philo 3 600 200 200
```
**Check**:
- No overlapping messages (messages should not be interleaved)
- Timestamps should be in chronological order
- Fork actions should be paired correctly

## 5. Performance and Stress Tests

### Test 5.1: Large Number of Philosophers
```bash
./philo 100 800 200 200
./philo 200 800 200 200
```
**Expected**: Should handle large numbers without crashing.

### Test 5.2: Long Running Tests
```bash
# Should run indefinitely without dying
./philo 5 800 200 200

# Let it run for several minutes, then Ctrl+C
# Check for memory leaks or degraded performance
```

### Test 5.3: Quick Timing Tests
```bash
# Very fast timing
./philo 5 410 1 1
./philo 4 800 1 1 50
```
**Expected**: Should handle rapid state changes correctly.

## 6. Optional Argument Tests

### Test 6.1: Limited Meals
```bash
# Each philosopher eats exactly 7 times
./philo 4 410 200 200 7
./philo 5 800 200 200 3
```
**Expected**: 
- Program should terminate when all philosophers have eaten the required number of times
- Should print completion or exit gracefully
- No philosopher should eat more than the specified number

### Test 6.2: Edge Cases with Meal Limits
```bash
# Single meal
./philo 4 410 200 200 1

# Zero meals (if allowed)
./philo 4 410 200 200 0
```

## 7. Output Format Verification

### Test 7.1: Message Format
Check that all messages follow the exact format:
```
timestamp_in_ms X has taken a fork
timestamp_in_ms X is eating
timestamp_in_ms X is sleeping
timestamp_in_ms X is thinking
timestamp_in_ms X died
```

### Test 7.2: Timestamp Accuracy
```bash
./philo 2 410 200 200
```
**Verify**:
- Timestamps start from 0 or program start time
- Timestamps are in milliseconds
- Timestamps increase monotonically
- Actions occur at expected intervals

## 8. Memory and Resource Tests

### Test 8.1: Memory Leak Detection
```bash
valgrind --leak-check=full ./philo 4 410 200 200
```
**Expected**: No memory leaks reported.

### Test 8.2: Thread Cleanup
```bash
# Ensure proper thread termination
./philo 4 310 200 200  # Let a philosopher die
# Check that all threads terminate properly
```

## 9. Real-world Scenario Tests

### Test 9.1: Subject Example Tests
Run the specific examples from the subject:
```bash
# Should not die
./philo 5 800 200 200

# Should not die  
./philo 5 800 200 200 7

# Should die
./philo 4 310 200 200

# Should not die
./philo 4 410 200 200

# One should die
./philo 1 800 200 200
```

### Test 9.2: Boundary Condition Tests
```bash
# Right at the edge of death
./philo 4 400 200 200  # time_to_die = time_to_eat * 2
./philo 2 201 200 1    # Very tight timing
```

## Testing Checklist Summary

### ✅ Mandatory Checks
- [ ] Compiles without warnings
- [ ] Handles all error cases gracefully
- [ ] No data races (verified with helgrind)
- [ ] No deadlocks during extended runs
- [ ] Proper death detection and timing
- [ ] Correct output format
- [ ] No memory leaks
- [ ] Passes all subject examples

### ✅ Advanced Verification
- [ ] Works with large numbers of philosophers (100+)
- [ ] Handles edge case timings correctly
- [ ] Proper thread synchronization
- [ ] Clean program termination
- [ ] Optional meal counting works correctly

## Common Issues to Look For

1. **Race Conditions**: Messages appearing out of order or overlapping
2. **Deadlocks**: Program hanging indefinitely
3. **Death Detection**: Late or early death detection, continuing after death
4. **Memory Issues**: Leaks, double frees, accessing freed memory
5. **Thread Issues**: Threads not terminating, zombie threads
6. **Timing Issues**: Incorrect timestamps, wrong intervals
7. **Fork Management**: Taking wrong number of forks, not releasing forks

## 10. Advanced Thread Issues Testing

### Test 10.1: Thread Creation and Destruction
```bash
# Monitor thread creation
ps -eLf | grep philo  # While program is running
pstree -p $(pgrep philo)  # See thread hierarchy

# Test multiple quick runs
for i in {1..10}; do ./philo 4 310 200 200; done
```
**Check for**:
- Threads are created properly
- All threads terminate when program ends
- No zombie threads remain after program exit

### Test 10.2: Thread Synchronization Stress Test
```bash
# Stress test with many philosophers and tight timing
./philo 20 500 100 100 &
PID=$!
# Monitor for 30 seconds
sleep 30
kill $PID

# Check for thread issues
ps aux | grep philo  # Should be empty after kill
```

### Test 10.3: Mutex Contention Testing
```bash
# High contention scenario
./philo 10 600 50 50

# Monitor mutex wait times and context switches
# Run in background and monitor
./philo 8 800 100 100 &
PID=$!
# Check context switches (high number may indicate poor synchronization)
while kill -0 $PID 2>/dev/null; do
    cat /proc/$PID/status | grep voluntary_ctxt_switches
    sleep 1
done
```

## 11. Detailed Timing Issues Tests

### Test 11.1: Precise Death Timing
```bash
# Create a script to test exact death timing
./philo 2 1000 400 200 | while read line; do
    echo "$(date '+%s%3N'): $line"
done
```
**Manual verification**:
- Record timestamp of last "is eating" message
- Death should occur exactly time_to_die ms later (±10ms tolerance)
- No messages should appear after death

### Test 11.2: Eating Duration Verification
```bash
# Test that eating takes exactly time_to_eat ms
./philo 3 800 300 100 | grep -E "(is eating|is sleeping)" | head -20
```
**Check**:
- Time between "is eating" and "is sleeping" should be ~300ms
- Use timestamps to verify accurate duration

### Test 11.3: Sleep Duration Verification
```bash
# Test that sleeping takes exactly time_to_sleep ms
./philo 3 800 100 250 | grep -E "(is sleeping|is thinking)" | head -20
```
**Check**:
- Time between "is sleeping" and "is thinking" should be ~250ms

### Test 11.4: Microsecond Precision Test
```bash
# Test with very precise timing
./philo 4 1000 1 1  # 1ms eat/sleep times
```
**Expected**: Should handle microsecond-level precision without errors.

## 12. Enhanced Death Detection Tests

### Test 12.1: Death During Different States
```bash
# Death while waiting for forks
./philo 3 250 300 100

# Death while eating (impossible but test the logic)
./philo 4 150 200 100

# Death while sleeping
./philo 4 350 200 200
```

### Test 12.2: Concurrent Death Detection
```bash
# Multiple philosophers might die simultaneously
./philo 6 400 500 100
```
**Check**:
- Only one death message should be printed
- Program should terminate immediately after first death
- No race condition in death detection

### Test 12.3: Death Message Timing Accuracy
```bash
# Script to verify death message timing
./philo 2 500 200 100 | awk '
/is eating/ { last_eat[$2] = $1 }
/died/ { 
    death_time = $1
    philo = $2
    if (last_eat[philo] > 0) {
        diff = death_time - last_eat[philo]
        print "Philosopher " philo " died " diff "ms after last meal (expected: 500ms)"
    }
}'
```

## 13. Comprehensive Deadlock Testing

### Test 13.1: Classic Deadlock Scenarios
```bash
# Even number of philosophers (higher deadlock risk)
timeout 60s ./philo 4 800 200 200
timeout 60s ./philo 6 800 200 200
timeout 60s ./philo 8 800 200 200

# If any timeout, there might be a deadlock
```

### Test 13.2: Resource Starvation Test
```bash
# One philosopher might be starved
./philo 5 1200 400 400 | grep "is eating" | sort -n | awk '{print $2}' | uniq -c
```
**Check**: Each philosopher should eat roughly the same number of times.

### Test 13.3: Deadlock Detection Script
```bash
# Monitor for deadlock signs
./philo 6 800 200 200 &
PID=$!
sleep 5

# Check if all threads are blocked
THREADS=$(ps -T -p $PID | wc -l)
echo "Thread count: $THREADS"

# Check for system deadlock detection
if command -v strace >/dev/null; then
    timeout 10s strace -p $PID 2>&1 | grep -E "(futex|mutex)" || echo "Possible deadlock"
fi

kill $PID 2>/dev/null
```

## 14. Race Condition Detection Tests

### Test 14.1: Message Interleaving Detection
```bash
# Check for message corruption
./philo 10 800 100 100 | head -100 | grep -E "^[0-9]+ [0-9]+ (has taken a fork|is eating|is sleeping|is thinking|died)$" | wc -l
# Count should match total lines if no corruption
```

### Test 14.2: Shared Resource Access Test
```bash
# Monitor for race conditions in fork access
./philo 4 800 200 200 | grep "has taken a fork" | head -20
```
**Check**:
- No philosopher should take the same fork simultaneously
- Forks should be taken in pairs before eating

### Test 14.3: Race Condition Stress Test
```bash
# Run multiple instances to stress test
for i in {1..5}; do
    ./philo 4 310 200 200 &
done
wait  # Wait for all to complete
```

## 15. Fork Management Deep Testing

### Test 15.1: Fork Availability Tracking
```bash
# Create a custom monitor for fork usage
./philo 4 800 200 200 | awk '
/has taken a fork/ { 
    forks_taken[$2]++
    total_forks++
    print $0 " (Total forks in use: " total_forks ")"
}
/is eating/ { 
    if (forks_taken[$2] != 2) {
        print "ERROR: Philosopher " $2 " eating with " forks_taken[$2] " forks!"
    }
}
/is sleeping/ { 
    total_forks -= forks_taken[$2]
    forks_taken[$2] = 0
    print $0 " (Total forks in use: " total_forks ")"
}'
```

### Test 15.2: Fork Deadlock Prevention
```bash
# Test with philosophers that should create fork conflicts
./philo 3 600 300 100  # Odd number with long eating time
```
**Monitor**: Ensure no circular waiting for forks.

### Test 15.3: Fork Release Verification
```bash
# Ensure forks are always released after eating
./philo 4 800 200 200 | grep -A1 "is eating" | grep -B1 "is sleeping"
```
**Check**: Every "is eating" should be followed by "is sleeping" (fork release).

## 16. System Resource Monitoring

### Test 16.1: CPU Usage Monitoring
```bash
# Monitor CPU usage during execution
./philo 10 800 200 200 &
PID=$!

# Continuous monitoring
while kill -0 $PID 2>/dev/null; do
    echo "=== $(date) ==="
    # CPU usage per thread
    ps -T -p $PID -o pid,tid,pcpu,pmem,comm
    
    # System CPU usage
    top -p $PID -n 1 | grep philo
    
    # Context switches (high number = poor synchronization)
    cat /proc/$PID/status | grep -E "(voluntary_ctxt_switches|nonvoluntary_ctxt_switches)"
    
    sleep 2
done
```

### Test 16.2: Memory Usage Monitoring
```bash
# Monitor memory usage over time
./philo 20 1200 300 300 &
PID=$!

while kill -0 $PID 2>/dev/null; do
    echo "=== Memory Usage $(date) ==="
    
    # Memory usage
    ps -p $PID -o pid,ppid,pmem,rss,vsz,comm
    
    # Detailed memory breakdown
    cat /proc/$PID/status | grep -E "(VmSize|VmRSS|VmHWM|VmStk|Threads)"
    
    # Memory maps (for leak detection)
    wc -l /proc/$PID/maps
    
    sleep 3
done
```

### Test 16.3: File Descriptor and Thread Monitoring
```bash
# Monitor resource usage
./philo 15 800 200 200 &
PID=$!

while kill -0 $PID 2>/dev/null; do
    echo "=== Resources $(date) ==="
    
    # File descriptors
    ls /proc/$PID/fd | wc -l
    
    # Thread count
    cat /proc/$PID/status | grep Threads
    
    # Memory locks (mutexes)
    cat /proc/$PID/status | grep VmLck
    
    sleep 2
done
```

### Test 16.4: Performance Regression Test
```bash
# Baseline performance test
echo "Testing performance with different philosopher counts..."

for count in 5 10 20 50; do
    echo "Testing with $count philosophers:"
    
    time timeout 30s ./philo $count 800 200 200 50 >/dev/null 2>&1
    
    # Check peak memory usage
    ./philo $count 800 200 200 &
    PID=$!
    sleep 5
    peak_mem=$(cat /proc/$PID/status | grep VmHWM | awk '{print $2}')
    echo "Peak memory: ${peak_mem} kB"
    kill $PID 2>/dev/null
    wait $PID 2>/dev/null
    
    echo "---"
done
```

## Debug Tips

1. **Add debug mode**: Compile with debug flags for more detailed output
2. **Use smaller numbers**: Test with 2-4 philosophers first
3. **System monitoring**: Use the resource monitoring scripts above during all tests
4. **Log analysis**: Pipe output to files for detailed analysis
   ```bash
   ./philo 4 410 200 200 > test_output.log 2>&1
   ```
5. **Incremental testing**: Start with basic functionality, add complexity gradually
6. **Profiling tools**: Use `gprof`, `perf`, or `valgrind --tool=callgrind` for performance analysis
7. **Thread debugging**: 
   ```bash
   gdb ./philo
   (gdb) set scheduler-locking on  # Debug one thread at a time
   (gdb) info threads             # List all threads
   (gdb) thread 2                 # Switch to thread 2
   ```
8. **Real-time monitoring**: Use `htop` or `atop` to monitor system resources during tests
9. **Stress testing**: Run tests under system load:
   ```bash
   stress --cpu 4 --timeout 60s &  # Create CPU load
   ./philo 10 800 200 200           # Run test under load
   ```

Remember to test both the mandatory part (threads + mutexes) and bonus part (processes + semaphores) if implemented!