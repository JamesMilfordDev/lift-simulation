# Lift Simulation

A set of C# console applications, each simulating a lift serving passenger calls in a ten-storey building, with detailed event-driven logging of passenger service.

## Purpose

This project explored lift scheduling from two complementary perspectives.

### A Theoretical Exercise

I considered the theoretical problem of how a single lift should serve passenger calls in a ten-storey building, beginning from first principles. My approach to the problem had two parts:

-	Identify evaluation metrics for a scheduling algorithm.
-	Create a range of algorithms, using my evaluation metrics as a guide.

### A Coding Exercise

For each of the main scheduling algorithms, I implemented a dedicated simulation.
The focus of this exercise was to build:

-	 A deterministic simulation of a complex system, with a clear simulation framework and decision logic that is straightforward to reason about and modify to enable multiple algorithms to be simulated.
-	A comprehensive logging system to enable algorithm evaluation.
-	A structured CSV input/output pipeline with comprehensive error handling.

To facilitate the development of multiple applications, differing only in the algorithm being simulated, I focused on clean architecture and modular code.

## Simulation Model

Each simulation is implemented as a time-based loop, employing an explicit event-driven lift state machine.

At each point in time, the lift occupies exactly one state from a finite collection of well-defined states, and a transition between states only occurs in response to a completed action or a new event.

In each iteration of the main simulation loop, we increment `currentTime` by 1, check what state the lift is in, and check whether it has now finished performing its action (if it was in a non-idle state). If it is idle at `currentTime`, or has finished performing an action, we decide the next action for the lift. Otherwise, we decrement the time to the completion of its current action by 1, and continue.

## Inputs

Each application takes as input exactly one string, corresponding to a CSV file detailing passenger calls for the lift to serve. In this CSV, each row represents a passenger. There are four columns:

- ID
- Start time
- Start floor
- Destination floor

## Outputs

Once the simulation has run, the program outputs to the console the total time taken for the lift to serve all passengers. It also generates a CSV log containing all recorded events that occurred during the simulation.

The following events are logged:

- A lift state change.
- The lift arriving at a floor.
- A passenger collection completion.
- A passenger drop off completion.

The exact nature of a log entry for a given application depends on the algorithm being simulated. However, for any application, each log entry includes at least:

- The time of the event.
- The floor of the lift at that time.
- The nature of the event.
- A list of the IDs of passengers in the lift.

## Architecture 

Each application has three sections:

1.	Input Handling
2.	Simulation Execution
3.	Output Generation

This uniform structure allows different algorithms to be compared without
introducing unrelated structural variation.

## Shared Assumptions

Each application assumes the lift: 

- Starts at floor 1.
- Takes 10 seconds to move between floors.
- Takes 5 seconds to collect a passenger.
- Takes 5 seconds to drop a passenger.
- Has a capacity of 8 (if capacity is modelled). 

Each of (i)-(v) can be adjusted by passing custom int values into the `Lift` constructor when the `Lift` object is created in `Main()`.

## Scheduling Algorithms Simulated

### OpportunisticFIFOWithoutCapacity

This is the most basic application in this repository. It simulates a lift that can accommodate multiple passengers at once, but which has no capacity. The lift algorithm sets a target passenger with a First In priority selection protocol, and focuses on serving the target passenger. 

However, on the way to collecting and dropping the target passenger, at each floor, the lift collects and drops extra passengers that need to be collected/dropped at that floor. 
Speaking broadly: at a new floor, the priority of actions is as follows (ordered from highest to lowest priority):

- Drop the target passenger (if they are inside the lift and their destination floor has been reached).
- Drop extra internal passengers.
- Collect the target passenger (if they are external and their start floor has been reached).
- Collect extra external passengers.

### OpportunisticFIFOWithCapacity

The algorithm simulated is the same as before, except now the lift has a capacity. Capacity comes into play at two points:

- The lift only collects extra external passengers if it has capacity.
- If (i) the target passenger is outside the lift, (ii) the lift arrives at the target passenger's start floor and would otherwise start collecting them, and (iii) the lift is at capacity, then the lift moves into an emergency state. In this state, the lift moves to the closest destination floor of a current internal passenger, then drops them off, then immediately returns to the target passenger's start floor, and collects the target passenger.

### DirectionalWithCapacity

This application simulates a lift that can accommodate multiple passengers at once, and which has a capacity. 

Broadly speaking, the lift serves all calls in one direction (all internal, and external when it has capacity). When there are no more internal calls, or external calls which the lift has capacity for, in its current direction, it changes direction. The application involves notable adjustments to my earlier `Lift` class, and the introduction of an additional enum.

## Suggested Reading Order

The applications were developed in the order presented above. Whilst each can be considered independently, they are most naturally read sequentially. Design choices in earlier work often informed later implementation decisions.

OpportunisticFIFOWithCapacity was the most complex algorithm to implement for me. For example, the `Lift` structure is more complex than with DirectionalWithCapacity, and it has the most comprehensive comments. I additionally included DirectionalWithCapacity to demonstrate an ability to create a different kind of algorithm. Interestingly, the two capacity algorithms have similar performance in some respects when using the CSV provided in this repository as input: the lift in OpportunisticFIFOWithCapacity serves all calls in 1145 seconds; the lift in DirectionalWithCapacity serves all calls in 1105 seconds (under default timing assumptions).
