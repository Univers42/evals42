# My Philosophers Testing Notes

## What I'm Testing
So I'm working on the philosophers project and need to make sure everything works properly. The program simulates philosophers eating, sleeping, and thinking while sharing forks. Here's how I've been testing it systematically.

## Program Usage (reminder for myself)
```bash
./philo number_of_philosophers time_to_die time_to_eat time_to_sleep [number_of_times_each_philosopher_must_eat]
```

## My Testing Setup

### Compilation Check
First thing I always do:
1. `make` - should work without warnings
2. `make clean` and `make fclean` - cleanup works
3. `make re` - full recompilation 
4. Check with norminette if I have it

### Basic Stuff I Test

## 1. Does It Even Work?

### Simple runs to see if it starts
```bash
./philo 4 410 200 200
./philo 5 800 200 200 7
./philo 2 400 100 100
```
I just let these run and watch to see if philosophers are actually doing their thing - taking forks, eating, sleeping, thinking.

### Making sure the output looks right
I check that I see messages like:
- "X has taken a fork" (should appear twice before eating)
- "X is eating"
- "X is sleeping" 
- "X is thinking"
- Timestamps should make sense and increase

## 2. Error Handling (Important!)

### Bad arguments that should fail
```bash
# Too few/many args
./philo 4 410 200
./philo 4 410 200 200 7 extra

# Negative or zero values
./philo -1 410 200 200
./philo 4 -410 200 200
./philo 0 410 200 200

# Non-numbers
./philo abc 410 200 200
```
These should all give me error messages and exit cleanly.

### Edge cases I found tricky
```bash
# Really big numbers
./philo 1000 410 200 200

# Really small numbers  
./philo 2 60 1 1
```

## 3. Death Detection (This Was Hard!)

### Single philosopher (should always die)
```bash
./philo 1 400 200 200
```
This one can't eat because there's only one fork, so should die around 400ms.

### Testing death timing
```bash
./philo 4 310 200 200  # Should die
./philo 2 300 400 100  # Should die (eating takes longer than time_to_die)
```

I've been checking that:
- Death happens at the right time (last meal + time_to_die)
- Program stops immediately when someone dies
- No more messages after death

### My timing verification trick
```bash
./philo 2 1000 400 200 | while read line; do
    echo "$(date '+%s%3N'): $line"
done
```
This helps me see exactly when things happen.

## 4. Threading Issues (The Nightmare Part)

### Checking for data races
```bash
valgrind --tool=helgrind ./philo 4 410 200 200
```
If I get warnings here, I know I have race conditions to fix.

### Deadlock testing
```bash
# These should run forever without hanging
timeout 30s ./philo 4 800 200 200
timeout 30s ./philo 6 800 200 200

# If timeout kills it, probably no deadlock
# If it hangs before timeout, I have a deadlock problem
```

### Thread monitoring while running
```bash
./philo 8 800 200 200 &
PID=$!
ps -eLf | grep philo  # Shows me all threads
kill $PID
```

## 5. My CPU and Memory Monitoring Setup

### CPU usage check
```bash
./philo 10 800 200 200 &
PID=$!

# I run this in another terminal while my program runs
while kill -0 $PID 2>/dev/null; do
    echo "=== $(date) ==="
    ps -T -p $PID -o pid,tid,pcpu,pmem,comm
    cat /proc/$PID/status | grep voluntary_ctxt_switches
    sleep 2
done
```

### Memory monitoring
```bash
./philo 20 1200 300 300 &
PID=$!

while kill -0 $PID 2>/dev/null; do
    echo "Memory usage:"
    ps -p $PID -o pid,pmem,rss,vsz,comm
    cat /proc/$PID/status | grep -E "(VmSize|VmRSS|Threads)"
    sleep 3
done
```

High CPU or memory that keeps growing = problem!

## 6. Fork Management Testing

### My fork tracking script
```bash
./philo 4 800 200 200 | awk '
/has taken a fork/ { 
    forks_taken[$2]++
    total_forks++
    print $0 " (Forks in use: " total_forks ")"
}
/is eating/ { 
    if (forks_taken[$2] != 2) {
        print "ERROR: Philosopher " $2 " eating with " forks_taken[$2] " forks!"
    }
}
/is sleeping/ { 
    total_forks -= forks_taken[$2]
    forks_taken[$2] = 0
    print $0 " (Forks in use: " total_forks ")"
}'
```

This helps me catch philosophers eating without proper forks.

## 7. Timing Precision Tests

### Eating duration check
```bash
./philo 3 800 300 100 | grep -E "(is eating|is sleeping)" | head -20
```
I manually check timestamps - time between "eating" and "sleeping" should be ~300ms.

### Sleep duration check  
```bash
./philo 3 800 100 250 | grep -E "(is sleeping|is thinking)" | head -20
```
Time between "sleeping" and "thinking" should be ~250ms.

### Super precise timing
```bash
./philo 4 1000 1 1  # 1ms eat/sleep times
```
This really stresses the timing system.

## 8. Death Detection Deep Dive

### Death timing accuracy test
```bash
./philo 2 500 200 100 | awk '
/is eating/ { last_eat[$2] = $1 }
/died/ { 
    death_time = $1
    philo = $2
    if (last_eat[philo] > 0) {
        diff = death_time - last_eat[philo]
        print "Philosopher " philo " died " diff "ms after last meal (should be ~500ms)"
    }
}'
```

### Multiple death scenarios
```bash
./philo 6 400 500 100  # Multiple might die at once
```
Only first death should be printed, then program ends.

## 9. Race Condition Hunting

### Message corruption check
```bash
./philo 10 800 100 100 | head -100 | grep -E "^[0-9]+ [0-9]+ (has taken a fork|is eating|is sleeping|is thinking|died)$" | wc -l
```
If this count doesn't match total lines, messages are getting corrupted.

### Stress testing
```bash
# Run multiple instances
for i in {1..5}; do
    ./philo 4 310 200 200 &
done
wait
```

## 10. Memory Leak Testing
```bash
valgrind --leak-check=full ./philo 4 410 200 200
```
Should show "no leaks are possible" or I need to fix something.

## 11. The Subject's Required Tests

These are the exact examples from the subject I always test:
```bash
./philo 5 800 200 200      # Should not die
./philo 5 800 200 200 7    # Should not die, stop after eating 7 times
./philo 4 310 200 200      # Should die
./philo 4 410 200 200      # Should not die  
./philo 1 800 200 200      # Should die (only one philosopher)
```

## 12. Performance Testing

### Large numbers
```bash
./philo 100 800 200 200  # Can it handle many philosophers?
./philo 200 800 200 200
```

### Long running
```bash
./philo 5 800 200 200  # Let it run for minutes, check for degradation
```

### My performance comparison script
```bash
echo "Performance test with different counts:"
for count in 5 10 20 50; do
    echo "Testing $count philosophers:"
    time timeout 30s ./philo $count 800 200 200 50 >/dev/null 2>&1
    echo "---"
done
```

## 13. Optional Meal Counting

```bash
./philo 4 410 200 200 7   # Each should eat exactly 7 times
./philo 5 800 200 200 3   # Should stop when all ate 3 times
```

## 14. Extreme Edge Cases I Discovered

### Microsecond precision madness
```bash
# These broke my first implementation
./philo 2 100 50 1     # Very tight death timing
./philo 3 101 100 1    # Death exactly 1ms after eating should finish
./philo 4 200 199 1    # Should barely survive
./philo 5 300 299 1    # Should barely survive
```

### Integer overflow tests
```bash
# Maximum int values (these should either work or fail gracefully)
./philo 2147483647 410 200 200
./philo 4 2147483647 200 200  
./philo 4 410 2147483647 200
./philo 4 410 200 2147483647
./philo 4 410 200 200 2147483647

# Just under max int
./philo 4 2147483646 200 200
```

### Boundary value testing
```bash
# Exactly at boundaries
./philo 2 400 200 200    # time_to_die = 2 * time_to_eat exactly
./philo 3 300 300 0      # Zero sleep time
./philo 4 600 200 400 1  # Minimum meal count
./philo 1 1 1 1 1        # All minimum values

# Off-by-one testing
./philo 2 399 200 200    # Just under 2 * time_to_eat  
./philo 2 401 200 200    # Just over 2 * time_to_eat
```

### Weird timing combinations
```bash
# Eating longer than death time (impossible scenario)
./philo 3 100 200 50     # Should die before finishing eating

# Sleep longer than death time
./philo 3 200 50 300     # Should die during sleep

# Think time longer than death (no explicit think time, but derived)
./philo 3 500 100 100    # Long gaps between actions
```

### Stress testing with weird ratios
```bash
# Very unbalanced timings
./philo 10 10000 1 1     # Very long death time, very short actions
./philo 10 100 90 1      # Eating takes 90% of available time
./philo 5 1000 1 999     # Mostly sleeping

# Rapid cycling
./philo 20 2000 1 1 1000 # Fast eating, many meals
```

## 15. Stricter Validation Tests

### Output format nazi-level checking
```bash
# Check EXACT format compliance
./philo 3 800 200 200 | head -50 | while read line; do
    if ! echo "$line" | grep -qE "^[0-9]+ [0-9]+ (has taken a fork|is eating|is sleeping|is thinking|died)$"; then
        echo "INVALID FORMAT: $line"
    fi
done

# Timestamp validation (should be monotonic)
./philo 4 800 200 200 | head -100 | awk '{
    if ($1 < prev_time) {
        print "TIME WENT BACKWARDS: " prev_time " -> " $1
    }
    prev_time = $1
}'

# Check philosopher IDs are valid
./philo 5 800 200 200 | head -50 | awk '{
    if ($2 < 1 || $2 > 5) {
        print "INVALID PHILOSOPHER ID: " $2
    }
}'
```

### Death detection precision torture test
```bash
# Death should happen EXACTLY at time_to_die, not before, not after
./philo 2 1000 100 100 | awk '
BEGIN { death_time = 1000 }
/is eating/ { 
    last_meal[$2] = $1 
    expected_death[$2] = $1 + death_time
}
/died/ {
    actual_death = $1
    philo = $2
    expected = expected_death[philo]
    diff = actual_death - expected
    
    if (diff > 10 || diff < -10) {
        print "DEATH TIMING ERROR: Philosopher " philo
        print "  Expected death at: " expected
        print "  Actually died at: " actual_death  
        print "  Difference: " diff "ms (should be ±10ms max)"
    }
}'
```

### Fork logic stress testing
```bash
# Make sure no philosopher can cheat the fork system
./philo 5 800 200 200 | awk '
/has taken a fork/ {
    forks[$2]++
    if (forks[$2] > 2) {
        print "ERROR: Philosopher " $2 " took " forks[$2] " forks!"
        exit 1
    }
}
/is eating/ {
    if (forks[$2] < 2) {
        print "ERROR: Philosopher " $2 " eating with only " forks[$2] " forks!"
        exit 1
    }
}
/is sleeping/ {
    if (forks[$2] != 2) {
        print "ERROR: Philosopher " $2 " released forks incorrectly (had " forks[$2] ")"
        exit 1
    }
    forks[$2] = 0
}
END {
    for (p in forks) {
        if (forks[p] != 0) {
            print "ERROR: Philosopher " p " ended with " forks[p] " forks!"
        }
    }
}'
```

### Action sequence validation
```bash
# Philosophers must follow the exact sequence: take fork -> take fork -> eat -> sleep -> think
./philo 3 800 300 200 | awk '
/has taken a fork/ {
    if (state[$2] == "eating" || state[$2] == "sleeping") {
        print "ERROR: Philosopher " $2 " took fork while " state[$2]
    }
    fork_count[$2]++
}
/is eating/ {
    if (fork_count[$2] != 2) {
        print "ERROR: Philosopher " $2 " eating without exactly 2 forks (has " fork_count[$2] ")"
    }
    state[$2] = "eating"
    fork_count[$2] = 0
}
/is sleeping/ {
    if (state[$2] != "eating") {
        print "ERROR: Philosopher " $2 " sleeping without eating first (was " state[$2] ")"
    }
    state[$2] = "sleeping"
}
/is thinking/ {
    if (state[$2] != "sleeping") {
        print "ERROR: Philosopher " $2 " thinking without sleeping first (was " state[$2] ")"
    }
    state[$2] = "thinking"
}'
```

### Memory leak hunting with extreme cases
```bash
# Run with valgrind on edge cases that might leak
valgrind --leak-check=full --show-leak-kinds=all ./philo 1 800 200 200
valgrind --leak-check=full --show-leak-kinds=all ./philo 100 1000 1 1 1
valgrind --leak-check=full --show-leak-kinds=all ./philo 2 100 200 200  # Should die quickly

# Check for memory leaks on invalid input (should still clean up)
valgrind --leak-check=full ./philo 0 200 200 200
valgrind --leak-check=full ./philo -5 200 200 200
```

### Thread safety stress test
```bash
# Run the same test many times to catch race conditions
echo "Running 100 iterations to catch rare race conditions..."
for i in {1..100}; do
    echo -n "Test $i: "
    timeout 10s ./philo 4 310 200 200 >/dev/null 2>&1
    if [ $? -eq 124 ]; then
        echo "TIMEOUT (possible deadlock)"
        break
    elif [ $? -ne 0 ]; then
        echo "CRASHED"
        break
    else
        echo "OK"
    fi
done
```

### Signal handling tests
```bash
# Test behavior when interrupted
./philo 5 800 200 200 &
PID=$!
sleep 5
kill -INT $PID   # Should clean up properly
wait $PID
echo "Exit code: $?"

# Test with different signals
./philo 5 800 200 200 &
PID=$!
sleep 3
kill -TERM $PID  # Should clean up properly
wait $PID

# Test SIGKILL (can't be caught, but shouldn't leave zombies)
./philo 5 800 200 200 &
PID=$!
sleep 2
kill -KILL $PID
sleep 1
ps aux | grep philo | grep -v grep  # Should be empty
```

## 16. Performance Regression Testing

### CPU efficiency tests
```bash
# CPU usage should be reasonable
./philo 10 800 200 200 &
PID=$!
sleep 10

# Get average CPU usage
CPU_USAGE=$(ps -p $PID -o %cpu --no-headers)
echo "CPU usage: $CPU_USAGE%"

# Should be much less than 100% per core
if (( $(echo "$CPU_USAGE > 50" | bc -l) )); then
    echo "WARNING: High CPU usage detected!"
fi

kill $PID
```

### Memory efficiency tests  
```bash
# Memory usage should be stable
./philo 50 1200 300 300 &
PID=$!

for i in {1..10}; do
    MEM=$(ps -p $PID -o rss --no-headers)
    echo "Memory usage at ${i}0s: ${MEM} KB"
    sleep 10
done

kill $PID
```

### Scalability tests
```bash
# Test how it scales with philosopher count
echo "Scalability test:"
for count in 2 5 10 20 50 100 200; do
    echo -n "Testing $count philosophers: "
    
    # Time how long it takes to start and run briefly
    start_time=$(date +%s%N)
    timeout 5s ./philo $count 800 100 100 10 >/dev/null 2>&1
    end_time=$(date +%s%N)
    
    duration=$(( (end_time - start_time) / 1000000 ))  # Convert to ms
    echo "${duration}ms"
    
    # Check if it completed successfully
    if [ $? -eq 0 ]; then
        echo "  ✓ Completed successfully"
    elif [ $? -eq 124 ]; then  
        echo "  ⚠ Timed out (may be normal for large counts)"
    else
        echo "  ✗ Failed or crashed"
    fi
done
```

## 17. Concurrent Execution Testing

### Multiple program instances
```bash
# Run multiple instances simultaneously to test system limits
echo "Testing concurrent instances..."
for i in {1..5}; do
    ./philo 10 800 200 200 &
    PIDS[$i]=$!
done

# Monitor all instances
sleep 30

# Check if all are still running
for i in {1..5}; do
    if kill -0 ${PIDS[$i]} 2>/dev/null; then
        echo "Instance $i: Still running"
        kill ${PIDS[$i]}
    else
        echo "Instance $i: Terminated"
    fi
done

wait  # Wait for all background jobs to finish
```

### Resource contention testing
```bash
# Test under heavy system load
echo "Testing under system stress..."

# Create CPU load
stress --cpu $(nproc) --timeout 60s &
STRESS_PID=$!

# Create memory pressure  
stress --vm 2 --vm-bytes 512M --timeout 60s &
STRESS_MEM_PID=$!

# Run philosopher test under load
./philo 10 800 200 200 20

# Cleanup stress tests
kill $STRESS_PID $STRESS_MEM_PID 2>/dev/null
```

## 18. My Debug Tricks

### When I'm debugging threading issues:
```bash
gdb ./philo
(gdb) set scheduler-locking on
(gdb) run 4 410 200 200
(gdb) info threads  # See all threads
(gdb) thread 2      # Switch to specific thread
```

### Resource monitoring during problems:
```bash
htop  # Keep this open in another terminal
# or
top -p $(pgrep philo)
```

### When testing under load:
```bash
stress --cpu 4 --timeout 60s &  # Create system stress
./philo 10 800 200 200           # Run my program under stress
```

## Common Problems I've Found

1. **Messages overlapping** = race condition in printf
2. **Program hanging** = deadlock 
3. **Death detected too late/early** = timing bug in death checker
4. **Memory keeps growing** = memory leak
5. **High CPU usage** = busy waiting instead of proper sleeping
6. **Context switches through the roof** = poor mutex usage

### Automated torture testing script
```bash
# My ultimate stress test script
#!/bin/bash
echo "=== PHILOSOPHERS TORTURE TEST ==="

# Test 1: Edge case arguments
echo "Testing edge case arguments..."
edge_cases=(
    "1 800 200 200"     # Single philosopher
    "2 100 50 1"        # Tight timing
    "3 101 100 1"       # Barely survivable
    "4 200 199 1"       # Should just survive
    "100 1000 1 1 10"   # Many philosophers, fast actions
    "5 10000 1 1"       # Very long death time
    "3 100 200 50"      # Impossible: eat longer than death time
)

for test in "${edge_cases[@]}"; do
    echo -n "  Testing: ./philo $test ... "
    timeout 15s ./philo $test >/dev/null 2>&1
    case $? in
        0) echo "✓ PASSED" ;;
        124) echo "⚠ TIMEOUT" ;;
        *) echo "✗ FAILED" ;;
    esac
done

# Test 2: Race condition detection
echo "Hunting for race conditions (100 iterations)..."
failed=0
for i in {1..100}; do
    ./philo 4 410 200 200 2>/dev/null | head -50 | grep -qE "^[0-9]+ [0-9]+ (has taken a fork|is eating|is sleeping|is thinking|died)$"
    if [ $? -ne 0 ]; then
        echo "  ✗ Race condition detected in iteration $i"
        ((failed++))
    fi
done
echo "  Race condition failures: $failed/100"

# Test 3: Memory leak check on edge cases
echo "Memory leak testing..."
if command -v valgrind >/dev/null; then
    valgrind --leak-check=full --error-exitcode=1 ./philo 1 800 200 200 >/dev/null 2>&1
    [ $? -eq 0 ] && echo "  ✓ No leaks on single philosopher" || echo "  ✗ Memory leak detected"
    
    valgrind --leak-check=full --error-exitcode=1 ./philo 2 100 200 200 >/dev/null 2>&1  
    [ $? -eq 0 ] && echo "  ✓ No leaks on quick death" || echo "  ✗ Memory leak on death scenario"
else
    echo "  ⚠ Valgrind not available"
fi

# Test 4: Performance regression
echo "Performance testing..."
start_time=$(date +%s%N)
timeout 30s ./philo 20 800 200 200 50 >/dev/null 2>&1
end_time=$(date +%s%N)
duration=$(( (end_time - start_time) / 1000000 ))

if [ $duration -lt 30000 ]; then
    echo "  ✓ Performance acceptable (${duration}ms)"
else
    echo "  ⚠ Performance slow (${duration}ms)"
fi

echo "=== TORTURE TEST COMPLETE ==="
```

### My strictest validation script
```bash
#!/bin/bash
# Ultra-strict output validation
./philo 4 800 200 200 | head -200 > test_output.txt

echo "=== STRICT VALIDATION ==="

# Check 1: Perfect format compliance
echo -n "Format validation: "
invalid_lines=$(grep -vE "^[0-9]+ [0-9]+ (has taken a fork|is eating|is sleeping|is thinking|died)$" test_output.txt | wc -l)
[ $invalid_lines -eq 0 ] && echo "✓ PASSED" || echo "✗ FAILED ($invalid_lines invalid lines)"

# Check 2: Timestamp monotonicity
echo -n "Timestamp ordering: "
timestamp_errors=$(awk 'NR>1 && $1<prev {print "ERROR"} {prev=$1}' test_output.txt | wc -l)
[ $timestamp_errors -eq 0 ] && echo "✓ PASSED" || echo "✗ FAILED ($timestamp_errors ordering errors)"

# Check 3: Valid philosopher IDs
echo -n "Philosopher ID validation: "
invalid_ids=$(awk '$2<1 || $2>4 {print}' test_output.txt | wc -l)
[ $invalid_ids -eq 0 ] && echo "✓ PASSED" || echo "✗ FAILED ($invalid_ids invalid IDs)"

# Check 4: Fork logic validation
echo -n "Fork logic validation: "
awk '
/has taken a fork/ { forks[$2]++ }
/is eating/ { 
    if (forks[$2] != 2) { 
        print "ERROR: Philosopher " $2 " eating with " forks[$2] " forks"; 
        errors++ 
    } 
}
/is sleeping/ { forks[$2] = 0 }
END { 
    if (errors > 0) exit 1
}' test_output.txt
[ $? -eq 0 ] && echo "✓ PASSED" || echo "✗ FAILED (fork logic errors)"

# Check 5: Action sequence validation
echo -n "Action sequence validation: "
awk '
/has taken a fork/ {
    if (state[$2] == "eating") { print "ERROR: Taking fork while eating"; errors++ }
    fork_count[$2]++
}
/is eating/ {
    if (fork_count[$2] != 2) { print "ERROR: Eating without 2 forks"; errors++ }
    state[$2] = "eating"
    fork_count[$2] = 0
}
/is sleeping/ {
    if (state[$2] != "eating") { print "ERROR: Sleeping without eating"; errors++ }
    state[$2] = "sleeping"
}
/is thinking/ {
    if (state[$2] != "sleeping") { print "ERROR: Thinking without sleeping"; errors++ }
    state[$2] = "thinking"
}
END { if (errors > 0) exit 1 }' test_output.txt
[ $? -eq 0 ] && echo "✓ PASSED" || echo "✗ FAILED (sequence errors)"

rm test_output.txt
echo "=== VALIDATION COMPLETE ==="
```

## My Testing Checklist (Updated - Much Stricter!)

### Absolutely must pass (non-negotiable):
- [ ] Compiles with -Wall -Wextra -Werror
- [ ] No norminette errors
- [ ] All error cases handled gracefully
- [ ] No races (helgrind completely clean)
- [ ] No deadlocks in 5-minute stress test
- [ ] Death timing accurate (±5ms, not ±10ms!)
- [ ] Perfect message format (no exceptions)
- [ ] Zero memory leaks (even on errors)
- [ ] All subject examples work perfectly
- [ ] Single philosopher dies correctly
- [ ] Works with 200+ philosophers
- [ ] Handles microsecond precision (1ms timings)
- [ ] Fork logic 100% correct (my strict validation passes)
- [ ] Action sequences always correct
- [ ] No CPU usage above 30% total
- [ ] Passes 100-iteration race condition test
- [ ] Signal handling works (SIGINT/SIGTERM cleanup)

### Edge cases that destroyed my first attempts:
- [ ] `./philo 2 100 50 1` (super tight timing)
- [ ] `./philo 3 101 100 1` (barely survivable)  
- [ ] `./philo 1 1 1 1 1` (all minimum values)
- [ ] `./philo 100 1000 1 1 1000` (stress test)
- [ ] `./philo 3 100 200 50` (impossible timing)
- [ ] Integer overflow inputs handled
- [ ] Concurrent program instances work
- [ ] Performance under system stress

### The ultimate tests (if these pass, I'm confident):
- [ ] My torture test script passes 100%
- [ ] My strict validation script passes 100%
- [ ] Runs 1 hour without issues (`./philo 10 2000 500 500`)
- [ ] Works perfectly under `stress --cpu 8 --vm 4`
- [ ] No issues in 1000-iteration race condition hunt
- [ ] Valgrind helgrind shows zero warnings on all edge cases
- [ ] Performance scales reasonably up to 500 philosophers

## Notes to Myself

- Always test the single philosopher case first - easiest to debug
- Use small numbers (2-4) when debugging race conditions  
- helgrind is my best friend for finding data races
- timeout command saves me from infinite hangs
- When in doubt, add more logging and test timing manually
- The 42 subject examples are non-negotiable - they must all work exactly right

I keep this file updated as I find new edge cases or better testing methods!