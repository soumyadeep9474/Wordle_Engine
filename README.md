# Wordle Sovlver
## Building and Running the Application
Building the Application
To build the application in release mode, use the following command:
```
cargo build --release
```
Running the Application
To run the application, you can use the compiled binary or run it directly using cargo. Here are the commands for both methods:
```
cargo run
```
## Block Diagram
```
+-----------------------------------------------------+
|                Command Line Interface (CLI)        |
|                  (argument parsing)                 |
|                                                     |
| - Argument Parsing:                                 |
|   - `no_sigmoid`                                    |
|   - `rank_by`                                       |
|   - `no_cache`                                      |
|   - `no_cutoff`                                     |
|   - `easy`                                          |
|   - `games`                                         |
|   - `interactive`                                  |
+-----------------------------------------------------+
                             |
                             |
                             V
+-----------------------------------------------------+
|                  Main Program (`main.rs`)           |
|   (configuration, solver setup, interactive/batch)  |
|                                                     |
| - Argument Handling:                               |
|   - `Args::parse()`                                |
| - Solver Configuration:                            |
|   - Set flags (no_cache, no_cutoff, etc.)           |
|   - Set ranking method (`rank_by`)                  |
| - Mode Selection:                                  |
|   - Interactive Mode                                |
|     - Call `play_interactive()`                     |
|   - Batch Mode                                      |
|     - Call `play()`                                |
+-----------------------------------------------------+
                             |
       +---------------------+---------------------+
       |                                           |
       |                                           |
       V                                           V
+-----------------------------+       +-----------------------------+
|       Interactive Mode       |       |          Batch Mode          |
|   (interactive gameplay)      |       |     (statistical analysis)   |
|                              |       |                             |
| - User Interaction:           |       | - Game Execution:            |
|   - Receive guesses from user |       |   - Run multiple games        |
|   - Ask for feedback          |       |   - Track results and scores  |
|   - Provide next guess        |       |   - Display statistics        |
|                              |       |                             |
| - Functions:                 |       | - Function: `play()`         |
|   - `play_interactive()`     |       |   - Loop through all games    |
|   - `ask_for_correctness()`  |       |   - Record results            |
+-----------------------------+       +-----------------------------+
                             |
                             |
                             V
+-----------------------------------------------------+
|                  Wordle Game Logic (`lib.rs`)        |
|               (gameplay and correctness logic)      |
|                                                     |
| - Core Structures:                                 |
|   - `Wordle`                                        |
|     - Manages dictionary and game state            |
|     - `play()`                                     |
|       - Runs the game with a guesser               |
|   - `Correctness`                                  |
|     - Enum for correctness states (Correct,        |
|       Misplaced, Wrong)                             |
|     - `compute()`                                  |
|       - Computes correctness of guesses            |
|   - `PackedCorrectness`                            |
|     - Efficiently packs correctness into a byte     |
|   - `Guess`                                        |
|     - Stores word and its correctness               |
|     - `matches()`                                 |
|       - Checks if a guess matches expected result  |
| - Guesser Trait                                    |
|   - Defines interface for guessing strategy        |
|   - Methods:                                       |
|     - `guess()`                                    |
|     - `finish()`                                   |
+-----------------------------------------------------+
                             |
                             |
                             V
+-----------------------------------------------------+
|                    Test Suite                       |
|                 (unit tests for verification)       |
|                                                     |
| - Test Modules:                                    |
|   - `guess_matcher`                                |
|     - Tests for correctness computation and        |
|       matching                                     |
|   - `game`                                         |
|     - Tests for different guessing strategies      |
|   - `compute`                                      |
|     - Tests for correctness calculation in various  |
|       scenarios                                    |
+-----------------------------------------------------+
```
## Data Flow Representation
```
[Dictionary File (`dictionary.txt`)] 
       |
       v
   [build.rs]
       |
       v
[Parse Dictionary Data] --> [Sort Dictionary Data] --> [Generate Output File (`dictionary.rs`)]
       |
       v
[Notify Cargo to Rerun]

[CLI Arguments] 
       |
       v
   [main.rs]
       |
       v
[Parse CLI Arguments] --> [Initialize Application] --> [Choose Game Mode]
       |
       v
[Run Selected Mode]
       |
       |----> [Interactive Mode]
       |         |
       |         v
       |   [Display Game Interface]
       |   [Receive User Input]
       |   [Process Feedback]
       |   [Provide Next Guess]
       |   [Repeat or End Game]
       |
       v
[Batch Mode]
       |
       v
   [Run Simulations] --> [Track Results] --> [Analyze Data]

[Initialize Wordle Game] --> [Play Game] --> [Guess Processing]
       |
       v
[Correctness Computation] --> [Pack Correctness] --> [Guess Matching]

[Solver (`solver.rs`)]
       |
       v
[Rank Calculation] --> [Guesser Interface] --> [Guesser Implementation]
```
## Detailed Breakdown
### 1. Entropy Calculation
Entropy is a crucial concept in information theory, representing the average amount of uncertainty or information content associated with a set of probabilities. In the context of this solver, entropy measures how unpredictable or uncertain the remaining possible solutions are.

* Formula:
  The entropy is calculated as:
  ```
  let remaining_entropy = -self
    .remaining
    .iter()
    .map(|&(_, p, _)| {
        let p = p / remaining_p;
        p * p.log2()
    })
    .sum::<f64>();

  ```
where
* p is the probability of a word.
* remaining_p is the total probability of all remaining words.
* p.log2() computes the log base 2 of p, which is used to measure the information content of each word.
  
