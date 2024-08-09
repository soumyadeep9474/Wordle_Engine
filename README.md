# Wordle Sovlver (Major League Hacking : Global Hack Week : Task)
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
## Algorithm Breakdown
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
  Explanation:
  * The term p * p.log2() represents the contribution to the entropy from a word with probability p. Higher probabilities contribute more to the entropy.
  * Summing over all remaining words gives the total entropy, which quantifies the uncertainty about the remaining words.

### 2. Estimation of Remaining Steps
The function est_steps_left estimates the number of guesses needed based on the current entropy. This estimation helps guide the guessing strategy by predicting how many more guesses might be required to solve the puzzle.
* Estimation Function:
  ```
  fn est_steps_left(entropy: f64) -> f64 {
    (entropy * 3.870 + 3.679).ln()
  }
  ```
  Explanation:
  * This function uses a regression model to estimate the number of steps left based on entropy. The formula (entropy * 3.870 + 3.679).ln() provides an
    estimate of remaining guesses.
  Purpose:
  * To inform the strategy on how many additional guesses are needed. A higher estimated number of steps suggests that the remaining entropy is high,
    indicating a more uncertain state.

### 3. Guess Ranking Strategies
Different strategies are employed to rank guesses based on their potential usefulness. The choice of strategy impacts how the solver selects the next guess:
* Rank by Expected Score:
  ```
  -(p_word * (score + 1.0)
    + (1.0 - p_word) * (score + est_steps_left(remaining_entropy - e_info)))
  ```
  Explanation:

  * p_word is the probability of the word being the solution.
  * score is the number of guesses made so far.
  * e_info is the expected information gain if the word is guessed.
  * This strategy evaluates guesses based on their expected score, combining probability and the estimated number of remaining steps.

* Rank by Weighted Information:
  ```
  p_word * e_info
  ```
  Explanation:
  * This strategy ranks guesses by the product of their probability and the expected information gain. Higher values indicate more valuable guesses.

* Rank by Info Plus Probability:
  ```
  p_word + e_info
  ```
  Explanation:
  * This strategy combines the probability of the word with the expected information gain. It gives a higher score to words that are either more probable
    or provide more information.
    
* Rank by Expected Information:
  ```
  e_info
  ```
  Explanation:
  * This strategy ranks guesses based solely on the expected information gain. It focuses purely on how much new information the guess will provide.


### 4. Sigmoid Smoothing
Sigmoid smoothing adjusts the raw word counts to prevent extreme values from dominating the probability distribution. This helps to normalize the probabilities and make the ranking more stable.
* Sigmoid Function:
  ```
  fn sigmoid(p: f64) -> f64 {
    L / (1.0 + (-K * (p - X0)).exp())
  }
  ```
  Explanation:
  * L, K, and X0 are constants that control the steepness and cutoff of the sigmoid function.
  * This function transforms raw probabilities into a smoothed form, making the distribution less sensitive to outliers.
  Purpose:
  * To create a more balanced probability distribution that helps in making fairer and more reliable guesses.

### 5. Caching and Efficiency
Caching is used to store precomputed correctness results to speed up the computation of guesses. This is essential for handling the large number of possible word pairs efficiently.
* Cache Initialization:
  ```
  COMPUTES.with(|c| {
      c.get_or_init(|| {
          let mem = unsafe {
              std::alloc::alloc_zeroed(
                  std::alloc::Layout::from_size_align(
                      std::mem::size_of::<Cache>(),
                      std::mem::align_of::<Cache>(),
                  )
                  .unwrap(),
              )
          };
          unsafe { Box::from_raw(mem as *mut _) }
      })
  });
  ```
  Explanation:
  * This code allocates a large array on the heap to store cached results. It uses unsafe operations to handle the allocation and initialization
    efficiently.
  Purpose:
  * To avoid recalculating correctness results for the same word pairs, thus improving the solver’s performance.

### 6. Decision Making
* Choosing the Best Guess:
  ```
  let mut best: Option<Candidate> = None;
  ```
  * The solver iterates through possible guesses, computes their goodness, and selects the best one based on the chosen ranking strategy.
    
* Final Guess Selection:
  ```
  let best = best.unwrap();
  self.last_guess_idx = Some(best.idx);
  best.word.to_string()
  ```
  Explanation:
  * The best guess is determined by comparing the goodness scores of all possible guesses and selecting the one with the highest score.
  Purpose:
  * To select the optimal guess that maximizes the expected information gain or other criteria based on the chosen strategy.
