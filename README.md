# RubricJudge

A reusable GenLayer Intelligent Contract for rubric-based scoring. A round owner defines weighted criteria, anyone submits a public URL, and validators independently score each entry with an LLM. Consensus is reached with a custom **tolerance rule** instead of exact matching, which suits subjective scoring where identical numbers are unrealistic.

Use it for hackathon judging, grant triage, bounty submissions, content review, or any "score this against a rubric" workflow. Other contracts can read results with `get_winner(round_id)` or `get_score(entry_id)`.

## How it works

1. **`create_round`**: the owner sets a title and a rubric of 1 to 5 criteria, each with a name, description and weight (1 to 10).
2. **`submit_entry`**: anyone submits a name and a public URL. One entry per address per round, up to 20 entries.
3. **`score_entry`**: anyone can trigger scoring of an entry. Every validator fetches the page and an LLM scores each criterion from 0 to 10. Validators agree if their scores are close enough to the leader's.
4. **`finalize_round`**: the owner closes the round once every entry is scored. The highest weighted total wins.
5. **`get_leaderboard`** / **`get_winner`**: read the ranking and the winner.

## Consensus design

This contract uses `gl.vm.run_nondet_unsafe` with a custom leader and validator function, not an exact-match principle.

| Aspect | Choice |
|---|---|
| Non-deterministic block | Fetch the entry page, then one LLM scoring call |
| Leader output | `{"scores": [int, ...], "note": str}`, one 0-10 integer per criterion |
| Validator rule | Re-fetch and re-score independently; accept if **every criterion is within 2 points** of the leader's score |
| Structural checks | Leader result must be a dict, have exactly one integer 0-10 per criterion, otherwise the validator rejects it |
| Why tolerance | LLM scores for the same page vary by a point or two; exact match would cause constant disagreement, while a loose rule would accept anything |
| Fallback | A page that fails to load scores zero on every criterion, with a note |
| Total score | Computed deterministically from the stored scores: `sum(score x weight) x 100 / (10 x sum(weights))`, giving 0 to 100 |
| Tie-break | The earliest submitted entry wins a tie |
| Prompt injection | Page content is marked as untrusted, and text that tries to dictate its own score is ignored |

## Contract interface

**Write methods**

| Method | Parameters | Notes |
|---|---|---|
| `create_round` | `title`, `criteria_json` | Returns the round id |
| `submit_entry` | `round_id`, `name`, `url` | One entry per address per round |
| `score_entry` | `entry_id` | Anyone can trigger; scores once |
| `finalize_round` | `round_id` | Owner only; needs all entries scored |

**View methods**

| Method | Returns |
|---|---|
| `get_round(round_id)` | Round details, rubric and status as JSON |
| `get_entry(entry_id)` | Entry, per-criterion scores, total and note as JSON |
| `get_score(entry_id)` | Weighted total, 0 to 100 |
| `get_leaderboard(round_id)` | Ranked list of scored entries as JSON |
| `get_winner(round_id)` | Winning entry as JSON (finalized rounds only) |
| `total_rounds()` / `total_entries()` | Counters |

Example rubric:

```json
[
  {"name": "Clarity", "description": "Is it explained clearly?", "weight": 3},
  {"name": "Originality", "description": "Is the idea novel?", "weight": 2}
]
```

## Using it from another contract

```python
judge = gl.get_contract_at(Address("0x...RubricJudge address..."))
winner = json.loads(judge.view().get_winner(round_id))
# pay out, mint, or grant access to winner["submitter"]
```

## Testing in GenLayer Studio

1. Upload `rubric_judge.py` and deploy it.
2. Call `create_round` with a title and the example rubric above.
3. Submit one entry from each of two accounts.
4. Call `score_entry` for each entry and wait for ACCEPTED.
5. As the round owner, call `finalize_round`, then `get_leaderboard` and `get_winner`.

## Test results

Run in GenLayer Studio under Normal (Full Consensus) mode, with a two-criterion rubric (Clarity weight 3, Originality weight 2):

| Entry | URL | Scores | Total |
|---|---|---|---|
| GenLayer | genlayer.com | 8, 7 | 76 |
| Example | example.com | 2, 0 | 12 |

- `create_round`, both `submit_entry` calls, both `score_entry` calls and `finalize_round` all finalized
- Leaderboard ranked GenLayer first; `get_winner(0)` returned entry 0 with a total of 76
- The stored notes explain each score, for example that the example.com page is a standard notice with minimal explanation of any project

Screenshots of the run are in the `screenshots/` folder.

## Edge cases handled

- Rubrics must be valid JSON with 1 to 5 named criteria and integer weights from 1 to 10
- One entry per address per round, and a 20-entry cap per round
- An entry can be scored only once, and only while the round is open
- `finalize_round` is owner only and refuses to run while any entry is unscored
- An unreachable page scores zero instead of blocking the round
- LLM output is clamped to integers from 0 to 10; malformed output falls back to zeros

## Limitations and ideas

- Entries must be public web pages
- The tolerance is a fixed constant; making it configurable per round is a natural extension
- Scores are computed by one LLM call per validator; multi-model or multi-sample scoring would tighten agreement
- Prize payout can be built on top by reading `get_winner`

## License

MIT
