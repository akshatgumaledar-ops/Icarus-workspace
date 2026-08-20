OBC Simulation Project Report

Overview
This purpose of this project was testing the flight OBC(On Board Computing) software for bugs.Space software need to run for a long time without crashing.The goal was to find and fix bugs like data overflowing and pointers going out of bounds.By adding safety limits and fixing how data flowed loops,I made the software stable enough to run till completion.In the final run the software completed all 2250 ticks without any segmentation faults or bugs mentioned above.

Resolved Issues & Bug Fixes

1.Problem:Missing Static Library Link Error (-lobc_physics)
The compiler was not able to locate the file directly during the build phase.It couldn't locate lobc_physics because the pre-compiled physics archive file was not in the same file directory
error message:Missing Static Library Link Error (-lobc_physics)
fix: I created the missing directory and moved the -lobs_physics file so I could be accessed by the compiler.It could now execute the build sequnece properly.

error code(in bash(cli for mac)):/usr/bin/ld: cannot find -lobc_physics: No such file or directory
collect2: error: ld returned 1 exit status
fix(in bash): mkdir -p lib/static
mv /path/to/libobc_physics.a lib/static/libobc_physics.a

2.Problem:Sensor Copy Buffer Overflow (Canary Corruption at Tick 16)
At the 16th tick the temperature sensor reads -2.Since the code calculates the memory dynamically, the reading caused a mathamatical error in size allotment for copying.It copied 34 bytes instead of 24,which caused overwriting of the next box since one box can only hold 24 bytes.The box next to it was an alarm called Canary which got triggerred due to said overwrite.
error message:`[WARN] Sensor copy canary modified at tick 16`.
fix:before handing the copy length to memcpy, I made a check so that canary doesn't overflow.sizeof(frame.dest) was my limit check.The condition was so that the calculated size didn't exceed sizeof(frame.dest),if it did it was shortened to the max length.

error code:memcpy(frame.dest, source_data, copy_len);
corrected code:size_t safe_len = copy_len < sizeof(frame.dest) ? copy_len : sizeof(frame.dest);
memcpy(frame.dest, source_data, safe_len);

3.Problem: Telemetry Ring Buffer Segmentation Fault (Tick 1026)
At the 1026th tick the system crashed ,because in src/drivers/telemetry_ingest.c the system utilised an array with a capacity of 1024 elements,when the system passed out of bounds(after 1024th element) there was a segmentation fault.Pointers were used to progress elements.
error message:[FATAL] Segmentation fault at tick 1026
Fix:Implemented a conditional logic that checks if the cursor reached the end of queque capacity(maximum number of elements in an array) and if it did,it'll reset it to the start(&state->shared.queue[0]) so data is overwritten in previous elements rather than going in forbidden elements and causing segmentation faults.

error code:static void cbuf_insert(AppState *state, const TelemetryFrame *frame) {
    WRITE_SLOT(state->telemetry_cursor, frame);
    state->telemetry_cursor++; 
    state->telemetry_frames++;
}
corrected code:static void cbuf_insert(AppState *state, const TelemetryFrame *frame) {
    WRITE_SLOT(state->telemetry_cursor, frame);
    state->telemetry_frames++;

    state->telemetry_cursor++;
    if (state->telemetry_cursor >= &state->shared.queue[1024]) {
        state->telemetry_cursor = &state->shared.queue[0];
    }
}

4.Problem:Thermal counter logic error
At the 1700th tick Therm_stale became 1 meaning that the system is using old data and not getting fresh data.This happened because the 16 bit counter broke down and overflowed so the timing math broke down
error message:no error message(logic error)
Fix:I replaced the 16 bit counter with a 36 bit counter

error code:
typedef struct {
    uint16_t runtime_ms;  
} ThermalTaskState;

typedef struct {
    uint16_t runtime_ms; 
} RefreshTaskState;
corrected code:
typedef struct {
    uint32_t runtime_ms; 
} ThermalTaskState;

typedef struct {
    uint32_t runtime_ms;  
} RefreshTaskState;
Testing and verification
To make sure everything was working properly.The code was tested on terminal using
make clean && make.This was more important for the physics library.Every fix worked and the final output was of
TICK:2250 | ORBIT:15 | TEMP:-15 | VBAT:2.438 | SAFE:1 | THERM_STALE: 0

Conclusion
In the end I had to fix 4 major bugs:
1.solve and build environment issues:physics library was in the wrong directory
2.Stopped memory crashes:stopped data overflowing in the 16th bit
3.Handled queque limits:ring buffer wrap-around logic to prevent segmentation fault at 1024 bit
4.Solved timing overflows:changed the 16 bit counter to 32 bit so that mission could run all 2250 flags with updating data.

Fault injection part-
I created a seperate branch from solution to make sure core bug fixes don't change when testing failure code.
In src/main.c I added a tick at 1700 that invokes cmd_set_actuators(0xFFFF).I also updated the fault_manifest.json file with this.Injected abnormal actuator values (0xFFFF) will disrupt system operations.If invalid values are injected it'll cause a sort of safe mode transition at tick 1700

