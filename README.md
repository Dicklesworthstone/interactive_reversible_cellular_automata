# Interactive Reversible Cellular Automata

This project provides an interactive, browser-based environment to explore **one-dimensional reversible cellular automata** (CAs) with a uniquely dynamic timeline. It allows you to run the automaton forward or backward in time, adjust the underlying rule, highlight differences that emerge through reversibility, and experiment interactively with pattern configurations—all at high performance thanks to modern WebGPU acceleration where available. It was heavily influenced by the article [One Dimensional Reversible Automata](https://richiejp.com/1d-reversible-automata) by Richard Palethorpe.

## Overview

A cellular automaton is a discrete model often visualized as a row of cells that evolve step-by-step according to a simple rule. Traditional one-dimensional CAs (like Wolfram's elementary cellular automata) evolve in a single forward direction, where you input an initial row and watch the pattern grow more complex over time. Reversible cellular automata, in contrast, can be run forward or backward, allowing you to "rewind" the evolution and navigate the state space more freely. Not all CA rules are reversible, but by clever construction, we can implement a reversible-like process here: at each forward step, the next row is calculated based on the current row and the previous row. To reverse direction, we use the stored history and shift the calculation to go backward.

This project blends these concepts into a fully interactive environment that:

1. Lets you choose from predefined rules or even tweak the rule’s output bits manually.
2. Allows you to start, stop, step through states, or continuously animate the evolution.
3. Supports reversibility—at any point, you can reverse direction and roll the CA steps backward.
4. Renders an interactive timeline so you can jump directly to any stored historical state.
5. Includes an interface to visualize and modify the underlying "rule patterns" of the CA.
6. Highlights cells that differ when comparing the forward-evolved next state against a hypothetical "non-reversible" next state. This makes the nature of reversibility clearer.

What makes this project interesting is not just that it shows a standard CA pattern. Instead, it demonstrates a fully bidirectional evolution in a way that's accessible, intuitive, and visually engaging. By leveraging GPU acceleration through WebGPU (with a graceful fallback to CPU computation), we can handle large grids and achieve smooth interactivity at full-screen resolutions even on complex rules.

## Key Concepts

### Reversible Cellular Automata

Standard elementary cellular automata apply a rule (a mapping from a cell’s neighborhood to a new state) to step forward. They typically lose information at each step, making it impossible to reconstruct earlier states uniquely. Reversible cellular automata, however, are designed so that no information is lost and the earlier states can be recovered. In this project, reversibility is simulated by XOR-ing the newly computed row with previous states to ensure that going backward is as straightforward as going forward.

Because this project stores a full history of states, you can switch directions at will. When run in "forward" mode, each new line depends on the previous one and some reversible logic. When reversed, it can reconstruct past lines. This dynamic approach provides a window into the structure and constraints of reversible evolution.

### Rule Definition and Patterns

A "rule" in a 1D elementary cellular automaton typically comes from a lookup table mapping 3-bit patterns (left, center, right cells) to an output bit. There are 8 possible 3-bit patterns, and each pattern maps to either 0 or 1, giving 2^8 = 256 possible rules. Well-known examples are Rule 90 or Rule 110.

Here, you can select from a number of predefined rules or modify the rule patterns interactively. The interface shows a grid of 3-bit neighborhoods (from 111 down to 000) and their corresponding output bits. Clicking on a pattern’s output toggles that bit in the rule. The CA then resets and runs with the newly defined behavior.

### Bidirectional Navigation Through Time

The UI provides controls for:

- **Play/Pause:** Start evolving forward automatically or stop at the current state.
- **Step:** Move one step forward in time (or backward if reversed).
- **Reverse:** Switch the direction of time. If currently running forward, a click sets the automaton to run backward through the stored history.
- **Reset:** Return to a fresh random initial state.
- **Timeline Scrubbing:** A timeline bar at the bottom of the screen shows the stored history. You can click and drag to jump to any previously computed step. This direct random access to any time point in the automaton’s history is a powerful exploration tool.

### Visual Highlights of Reversibility

One unique feature is the ability to toggle "Reversibility Change Highlighting." When this is enabled, cells that differ between the actual reversible next state and a hypothetical "non-reversible" next state (computed by just the CA rule without the XOR reversibility trick) are visually highlighted with distinct colors. Forward differences are shown in green hues; backward differences in red. This provides a visual cue to understand how much "information" is being encoded or recovered at each step.

## Implementation Details

This project is implemented entirely within a single HTML file using JavaScript and WebGPU. Key implementation aspects include:

1. **WebGPU Acceleration:**  
   When supported, the application uses a GPU compute pipeline to apply the CA rule. This involves:
   - Uploading the current row, previous row, and next row buffers to GPU memory.
   - Dispatching a compute shader that applies the CA rule in parallel across all 32-bit words of the state.
   - The shader:
     - Extracts bit neighborhoods (left, center, right) from wrapped-around shifts.
     - Looks up the corresponding output bit from the rule.
     - Applies XOR with the previous or next row for reversibility.
   - Reads back the newly computed row from GPU memory.
   
   This parallel processing significantly speeds up the simulation on large grids.

2. **Graceful Fallback to CPU:**  
   If WebGPU is not available, the code falls back to a CPU-based computation. The CPU code:
   - Uses bitwise operations to shift and mask the current row to find the required neighborhoods.
   - Supports memoization (caching repeated pattern computations) to improve performance when the CA runs over large areas or for long time spans. This caching can significantly reduce repetitive computations, especially for stable or repetitive patterns.
   - Still runs at a reasonable speed given modern JavaScript engines, but may not handle the largest displays as fluidly as the GPU version.

3. **Memory and Data Structures:**
   - The state of the CA is stored in a bit-packed format, with 32 cells per 32-bit integer, arranged in an array of `Uint32Array`.
   - The width of the simulation is rounded up to a multiple of 64 for efficiency.
   - Each step, the code swaps buffers and updates references to avoid unnecessary data copying.
   - A history array stores each computed row, allowing arbitrary time navigation. Differences between reversible and non-reversible evolution are also stored for later highlighting.

4. **Dynamic Resizing:**
   - On window resize, the simulation can reset with a new grid size matching the updated viewport.
   - GPU and CPU resources are reallocated as needed.

5. **Pattern Editing and UI:**
   - The pattern overlay shows the 8 three-bit patterns and their outputs for the current rule.
   - Clicking an output toggles the bit, updates the rule, and reinitializes the simulation.
   - This immediate, interactive manipulation encourages experimenting with different rules.

6. **Timeline and Interaction:**
   - A canvas-based timeline shows the distribution of computed states.
   - Clicking and dragging on the timeline sets the current step index, loading the corresponding stored state from the history.
   - Tooltips provide step indices during interaction, aiding navigation through long runs.

7. **Performance Tricks and Optimizations:**
   - **Bitwise Operations for Efficiency:**  
     All cell computations are done using bitwise shifts, masks, and XOR to process 32 cells at a time. This minimizes overhead compared to handling each cell individually.
   
   - **Use of WebGPU Compute Shaders:**  
     Offloading computation to GPU workgroups of 256 threads each means thousands of cells can be updated in parallel, accelerating large simulations.
   
   - **Memoization (CPU Path):**  
     When running in CPU mode, certain state patterns (left, center, right, previous row) may recur. The code caches results keyed by these quadruples, speeding subsequent identical computations and reducing CPU load over time.
   
   - **Double Buffering Staging on GPU:**  
     The code uses two staging buffers to read data back from the GPU asynchronously. While one buffer is mapped for reading, the other is submitted for GPU writes, improving throughput and reducing stalls.
   
   - **Reduced Data Copies:**  
     The implementation reuses arrays, avoids unnecessary array copying, and uses typed arrays for fast memory operations.

8. **Highlighting Mechanism:**
   - For each computed row, the code also computes a "non-reversible" row (just applying the raw rule without XOR). The XOR of the actual reversible row and the non-reversible row highlights which bits differ.
   - During rendering, these differences are translated into colored pixels if the "Highlight Reversibility Changes" mode is on.
   - This color encoding provides immediate visual feedback regarding the complexity of reversible manipulation.

## Getting Started

1. Clone the repository:
   ```bash
   git clone https://github.com/Dicklesworthstone/interactive_reversible_cellular_automata.git
