# smarts-evolution

[![CI](https://github.com/LucaCappelletti94/smarts-evolution/actions/workflows/ci.yml/badge.svg)](https://github.com/LucaCappelletti94/smarts-evolution/actions/workflows/ci.yml)
[![codecov](https://codecov.io/gh/LucaCappelletti94/smarts-evolution/graph/badge.svg)](https://codecov.io/gh/LucaCappelletti94/smarts-evolution)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

Evolving SMARTS patterns against a binary classification task.

Native terminal runs can use `task.evolve_with_tui(&config, &seed_corpus)` with the `tui` feature enabled.

## Quick Start

```rust
use smiles_parser::Smiles;
use smarts_evolution::{
    EvolutionConfig, EvolutionTask, FoldData, FoldSample, SeedCorpus,
};
use smarts_rs::PreparedTarget;

fn prepared(smiles: &str) -> PreparedTarget {
    PreparedTarget::new(Smiles::from_str(smiles).unwrap())
}

let task = EvolutionTask::new(
    "amide-vs-rest",
    vec![FoldData::new(vec![
        FoldSample::positive(prepared("CC(=O)N")),
        FoldSample::positive(prepared("NC(=O)C")),
        FoldSample::negative(prepared("CCO")),
        FoldSample::negative(prepared("c1ccccc1")),
    ])],
);

let config = EvolutionConfig::builder()
    .population_size(8)
    .generation_limit(2)
    .stagnation_limit(2)
    .build()
    .unwrap();

let seed_corpus = SeedCorpus::try_from([
    "[#6](=[#8])[#7]",
    "[#6]~[#7]",
])
.unwrap();

let result = task.evolve(&config, &seed_corpus).unwrap();
assert!(!result.best_smarts().is_empty());
assert!(result.best_mcc().is_finite());
```

## TUI Clipboard

The native TUI (`tui` feature) has `[copy]` buttons for the best SMARTS and for change-point SMARTS. Copying tries the OS clipboard first on a local desktop session, and falls back to a terminal OSC 52 escape sequence when no native clipboard is reachable. For SSH and tmux sessions it skips the native clipboard (which would land on the wrong machine) and forwards to the controlling terminal via OSC 52 directly, so the SMARTS reaches the clipboard on the machine you are actually viewing.

Two things must hold for the OSC 52 path to land in your local clipboard:

- The outer terminal (the one you read on the local end of the SSH/tmux session) must support OSC 52 clipboard writes. Most modern terminals do (kitty, Alacritty, WezTerm, foot, iTerm2, recent xterm); some, such as GNOME Terminal and other VTE-based terminals, do not.
- Inside tmux, OSC 52 is wrapped in tmux's passthrough sequence so tmux forwards it to the outer terminal. This requires `allow-passthrough` to be enabled (tmux 3.3+ defaults it off). Add this to your `tmux.conf`:

  ```tmux
  set -g allow-passthrough on
  ```

  Alternatively, configure tmux's own clipboard forwarding instead of passthrough (`set -g set-clipboard on` plus `set -as terminal-features ',*:clipboard'`), which also lets tmux pass the bare OSC 52 to a capable outer terminal.

When a copy happens inside tmux, the TUI queries `tmux show-options` for the effective `allow-passthrough` value of the current pane. If it is off, the status line reports that the copy will not reach the clipboard and shows the exact `tmux set -g allow-passthrough on` command to fix it, instead of falsely claiming success. Note that `set -g allow-passthrough on` is a tmux command: run it as `tmux set -g allow-passthrough on` from a shell, add the bare `set -g allow-passthrough on` line to `~/.tmux.conf`, or type it after `Ctrl-b :` inside tmux. It is not a shell builtin.

## Terminal Progress Bars

Enable the `indicatif` feature and call `task.evolve_with_indicatif(&config, &seed_corpus)` to get generation, per-generation SMARTS evaluation, and offspring candidate-generation progress bars. The evaluation bar reports completion, current SMARTS and MCC, generation-best SMARTS and MCC, and incumbent-best SMARTS and MCC; the offspring bar reports mutation and candidate selection progress while the next generation is prepared. For large prepared datasets, use `task.evolve_owned_with_indicatif_progress(...)` to move the task folds into the session; use `IndicatifEvolutionProgress::attach_to(&multi)` or `from_bars_with_offspring(...)` to embed all bars in an existing `MultiProgress`. Non-terminal callers can implement `EvolutionProgressObserver` and pass it to `task.evolve_with_observer(...)`.

## Pathological SMARTS Evaluation

The GA logs each SMARTS evaluation at `debug` level, applies `EvolutionConfig::match_time_limit` as a cooperative per-SMARTS evaluation safety fuse, and emits a `warn` log when matching exceeds that limit. Limit-exceeded matcher results are treated as unknown, so the affected genome receives invalid fitness instead of counting the sample as a non-match. SMARTS length is used only for deterministic tie-breaking and optional `max_evaluation_smarts_len` filtering.

For a standalone `.log` file, initialize the built-in file logger before starting evolution:

```rust,no_run
smarts_evolution::FileLogConfig::new("smarts-evolution.log")
    .level(smarts_evolution::LevelFilter::Debug)
    .init()
    .expect("initialize smarts-evolution file logger");
```

The file logger is file-only by default so indicatif bars keep control of stderr. Use `.mirror_to_stderr(true)` only when terminal log lines are useful.
