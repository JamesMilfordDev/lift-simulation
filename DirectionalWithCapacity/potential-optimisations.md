# Potential Optimisations

This document outlines potential optimisations that were considered during development but intentionally not implemented. In most cases, the decision not to implement reflects a conscious tradeoff between performance, readability, and conceptual clarity.

I see four potential areas of optimisation.

- Syntax
- Application Efficiency
- Lift Algorithm Efficiency
- Reporting

## Syntax

I at times defined and employed helper methods in `Main()` to simplify the code found in the body of the simulation. For example, I created the `AddLog()` method that handled the creation of `LogEntry` objects and their addition to the `List<LogEntry>` log. Further, I created associated methods, `StateChange()`, `FloorArrival()`, `PassengerCollection()`, and `PassengerDrop()`, to handle different kinds of events that are to be logged.

I could use helper methods more extensively. For example, if we take `OpportunisticFIFOWithCapacity`, I often repeat code blocks, such as when the lift moves into the `DroppingExtra` state. Creating a helper method `MoveToDroppingExtra()` would reduce the total lines of code. 

However, the benefit here is marginal, and the cost is reduced local readability: explicitly writing out this block of code in full at each appropriate point is not particularly noisy, and ensures that the reader understands exactly what is happening without having to navigate back up to the helper method. After all, multiple separate actions are executed. In contrast, it is much more obvious what calling the  `AddLog()` helper method does. Without this helper, the code would not be any clearer to a reader, but would be littered with noisy `LogEntry` instance creations.

## Application Efficiency

I ensured that my logic is correct, that is, each application correctly simulates the lift algorithm I want it to simulate. However, there is room to improve the efficiency of the applications themselves. Here are two potential areas of improvement.

### Handling Passenger Calls

The first area concerns how the lift manages active passengers. 

Currently, `PassengerCalls` is a simple `List<Passenger>` object that is updated each time the while loop runs by calling `AddRange()` on it, passing as argument a list of all the passengers that appeared at `currentTime`. Candidate passengers are then identified by filtering this list at decision points within the simulation loop using LINQ.

A more efficient alternative would be to use a more specialised data structure to hold live `Passenger` objects. I chose not to implement this strategy as it introduces additional code complexity and reduces local readability. With the current approach, the decision logic is explicit and easy to reason about, which was especially important given the primary goal was to implement and compare multiple different algorithms.

### Handling the Passage of Time

The second area concerns how many times the overall while loop needs to run.

Currently, the while loop runs once for each second of the simulation. Each time the while loop runs, we increment `currentTime` by 1, check what state the lift is in, and check whether it has now finished performing its action (if it was in a non-idle state). If it is idle at `currentTime`, or has finished performing an action, we decide the next action for the lift. Otherwise, we decrement the time to the completion of its current action by 1, and continue.

For example, let us consider `OpportunisticFIFOWithCapacity` again. Suppose the while loops runs, and after incrementing `currentTime`, the lift is in the `MovingToCollect` state, but `lift.TimeToNextFloor == 0`. That is, the lift has just finished moving to the next floor on its way to collecting the target passenger. Suppose that there is an internal passenger whose destination floor is the new current floor. Then the lift moves into the `DroppingExtra` state. In the current code, we adjust `lift.TimeToDropEnd` to `lift.DropTime` (5 seconds by default), decrement this by 1 second, then continue. Now, the while block runs four additional times when `lift.TimeToDropEnd` starts as a positive int (namely, when it starts as 4, then 3, then 2, then 1). In each of these runs, we will check `lift.State`, check `lift.TimeToDropEnd`, decrement `lift.TimeToDropEnd` by 1, then merely continue. It is only when `lift.TimeToDropEnd == 0` that the next lift state is calculated.

Call this the **Simple Tick Approach**.

Given that we know in advance how long each action takes to complete, an alternative simulation approach is available. Put simply, we allow time skips. The loop runs at exactly those `currentTime` values when the lift has finished completing an action (collecting / dropping / travelling to a new floor) and must decide what next to do, or is idle and so must decide what next to do. Rather than incrementing `currentTime` by 1 at the start of each loop, we only increment `currentTime` when we have decided what the lift’s next action is starting at `currentTime`. If the lift is to remain idle, we increment `currentTime` by 1. If the lift is to begin performing an action, we increment `currentTime` by the amount of time it takes for the lift to perform that action.

With this approach, the simulation is completed with fewer while loop runs. Further, we no longer ever need our logic to check whether the lift has finished its current action.

Call this the **Time Skip Approach**.

I chose to avoid the Time Skip Approach in my implementations. Although this approach is more efficient, the more gamified nature of the time skip simulation is less intuitive, and harder to reason about and modify. Again, with the current approach, the decision logic is explicit and easy to reason about, which was especially important given the primary goal was to implement and compare multiple different algorithms.

### Lift Algorithm Efficiency

In my implementations, I assumed that each floor has exactly one external call button, which is non-directed. However, given that each `Passenger` object contains values for both its start floor and destination floor, we could alternatively model each floor as having a pair of directed external call buttons: one for up and one for down. Whether a passenger presses up or down can be inferred from their start and destination floors.

One way to incorporate this extra information into my `DirectionalWithCapacity` algorithm would be to update the collection rule: the lift only collects an external passenger p at its current floor if both (i) it has capacity, and (ii) it is travelling in the direction of the destination floor of p.

This rule only partially specifies the algorithm; we still need to say under what circumstances the lift changes direction. We might be tempted to say: the lift changes direction when there are neither internal calls in the lift’s current direction, nor external calls in the lift’s current direction that (i) request travel in the same direction as the lift and (ii) the lift has capacity for.

However, this would result in external call starvation at the extremities. For example, if the lift is travelling up and there is an external passenger call from floor 10 (for whom the start floor and destination floor are distinct and therefore by definition the passenger wants to move down), then with this approach, that passenger call would only ever be served if there is an internal passenger that wants to be dropped off at floor 10.

A better complement is therefore: the lift changes direction when there are neither internal calls in the lift’s current direction, nor external calls in the lift’s current direction which the lift has capacity for (regardless of which direction that external passenger wants to go).

In our case of the lift travelling up and there being an external passenger at floor 10, assuming the lift has capacity, it will continue to floor 10. At that point, it will change direction, and then collect the passenger.

### Reporting

The main area in which my applications could be optimised is improving their reporting.

Currently, each application outputs to the console the total time taken for the lift to serve all passenger calls. I also generate a CSV file containing a log of every recorded event from the simulation. From that log, one can infer additional information, such as the total travel distance of the lift, the external wait time of each passenger, and the internal wait time of each passenger.

However, it would be helpful if that information was calculated and reported by the application. After all, our metrics for a good algorithm may include, amongst other things, total time to serve all passengers, total distance travelled, external call starvation, and internal call starvation.

Many of these reporting improvements can be implemented in a straightforward fashion. For example:
- To handle total travel distance: before the simulation begins, we initialise an int variable `travelDistance` as 0, then increment this by 1 each time the lift arrives at a new floor. This could then be printed to the console alongside the total time to serve all passengers. 

- To handle starvation, for each passenger, we could add `externalWait` and `internalWait` as int? values that start as `null`. `externalWait` is adjusted to the relevant int when collection of that passenger begins; `internalWait` is adjusted to the relevant int when dropping off of that passenger begins. This information could then be reported in an additional CSV. 

- We could also calculate the average external and internal wait times, using the `externalWait` and `internalWait` values, and print these to the console alongside total time to serve all passengers and total distance travelled.