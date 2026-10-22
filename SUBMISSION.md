# HW4 — submission

**AI assistance:** <!-- what you used, and for what. "None" is an answer too. -->

---

## Part A — Plan on paper (Easy)

> Commit this whole part before any `.py` file exists.


### A1 — The steps

| # | Step | What goes in (types) | What comes out (types) | When it goes wrong | Tool, loop, or model? |
|---|---|---|---|---|---|
| 1 | Understand the question | question (str) | district names (list), fields (list), operation (str) | The question is unclear | Model |
| 2 | Find the facts | district (str), field (str) | value (float), unit (str), source (str) | No match, similar names, or missing data | lookup tool |
| 3 | Check the data | facts (dict), sources (list) | checked facts (dict) | Old profile, correction, or wrong year | lookup tool |
| 4 | Prepare numbers | values (list), units (list) | numbers (list) | Invalid number or unknown unit | Model and loop |
| 5 | Do the calculation | expression (str) | result (float) | Division by zero or invalid expression | calculate tool |
| 6 | Compare the results | results (list) | district name (str), difference (float) | Missing result or unclear comparison | Model |
| 7 | Give the final answer | name (str), number (float) | answer (str) | Wrong format or rounding | Model and loop |

**How the steps connect:**

Question → Understand → Lookup → Check data → Prepare numbers → Calculate → Compare → Final answer.

If a fact is missing, the model can request another lookup.

**What I am assuming:**

- I use the 2025 data unless the question asks for another year.
- Archived profiles are not used when newer data exists.
- Correction notices have priority over original profiles.
- I use the correct units before calculation.
- For water calculations, 1 megalitre = 1000 cubic metres.
- I keep full precision during calculations and round only the final answer.
- I use exact district names to avoid confusion.



### A2 — The two tools

#### `lookup`

```json
{
  "type": "function",
  "function": {
    "name": "lookup",
    "description": "Find a fact about a district or the region in the Nine Rivers documents. Use this tool before doing calculations. Give an exact subject and field name. It uses current profiles, ignores archived data, applies corrections, and returns the source document ID.",
    "parameters": {
      "type": "object",
      "properties": {
        "subject": {
          "type": "string",
          "description": "Exact district name, for example Brightwater, or Nine Rivers Region for regional data."
        },
        "field": {
          "type": "string",
          "description": "Fact needed, for example population_2025, population_2020, area_km2, schools, pupils, water_use_m3, budget_million_tenge, paved_road_km, founded_year, or water_tariff_2025."
        }
      },
      "required": ["subject", "field"],
      "additionalProperties": false
    }
  }
}
```

| Case | What the tool returns |
|---|---|
| success | JSON with subject, field, value, unit and source |
| district not found | Error with possible district names |
| field not found | Error explaining that the field is unavailable |
| similar names | Uses exact matching and never guesses |
| archived profile | Uses the current 2025 profile instead |
| corrected value | Uses the correction notice and includes its source |
| invalid input | Returns an error without crashing |

**Worst thing it could do with strange or dangerous input:**

The model could request a wrong district or field and get incorrect data if the tool guesses. My tool will use exact names, return errors for unknown inputs, and never open arbitrary files or execute user input.

#### `calculate`

```json
{
  "type": "function",
  "function": {
    "name": "calculate",
    "description": "Calculate a mathematical expression using numbers and basic arithmetic. Use this tool for divisions, percentages, differences and unit conversions. Only +, -, *, /, parentheses and numeric values are allowed. Do not send Python code.",
    "parameters": {
      "type": "object",
      "properties": {
        "expression": {
          "type": "string",
          "description": "A mathematical expression such as 44675 / 921.4 or (100 - 80) / 80 * 100."
        }
      },
      "required": ["expression"],
      "additionalProperties": false
    }
  }
}
```

| Case | What the tool returns |
|---|---|
| success | JSON with expression and numeric result |
| division by zero | Error: division by zero |
| invalid expression | Error: invalid mathematical expression |
| dangerous Python code | Error: operation not allowed |
| very large number | Error: number exceeds the allowed limit |
| empty expression | Error: expression is empty |

**Worst thing it could do with strange or dangerous input:**

A dangerous expression could run unwanted Python code or use too much memory. I will use Python's `ast` module instead of `eval` or `exec`. Only basic arithmetic will be allowed. I will also limit expression length and number size.




### A3 — Q-09 by hand

This is my expected conversation between the model and my two tools.

```json
[
  {
    "role": "user",
    "content": "Which district has the higher population density in 2025, Brightwater or Millbrook, and by how many residents per square kilometre?"
  },
  {
    "role": "assistant",
    "tool_calls": [
      {
        "id": "call_1",
        "type": "function",
        "function": {
          "name": "lookup",
          "arguments": "{\"subject\":\"Brightwater\",\"field\":\"population_2025\"}"
        }
      },
      {
        "id": "call_2",
        "type": "function",
        "function": {
          "name": "lookup",
          "arguments": "{\"subject\":\"Brightwater\",\"field\":\"area_km2\"}"
        }
      },
      {
        "id": "call_3",
        "type": "function",
        "function": {
          "name": "lookup",
          "arguments": "{\"subject\":\"Millbrook\",\"field\":\"population_2025\"}"
        }
      },
      {
        "id": "call_4",
        "type": "function",
        "function": {
          "name": "lookup",
          "arguments": "{\"subject\":\"Millbrook\",\"field\":\"area_km2\"}"
        }
      }
    ]
  },
  {
    "role": "tool",
    "tool_call_id": "call_1",
    "content": "{\"value\":44675,\"unit\":\"residents\",\"source\":\"NR-05\"}"
  },
  {
    "role": "tool",
    "tool_call_id": "call_2",
    "content": "{\"value\":921.4,\"unit\":\"km2\",\"source\":[\"NR-05\",\"NR-16\"]}"
  },
  {
    "role": "tool",
    "tool_call_id": "call_3",
    "content": "{\"value\":37203,\"unit\":\"residents\",\"source\":\"NR-11\"}"
  },
  {
    "role": "tool",
    "tool_call_id": "call_4",
    "content": "{\"value\":758.6,\"unit\":\"km2\",\"source\":\"NR-11\"}"
  },
  {
    "role": "assistant",
    "tool_calls": [
      {
        "id": "call_5",
        "type": "function",
        "function": {
          "name": "calculate",
          "arguments": "{\"expression\":\"44675 / 921.4\"}"
        }
      },
      {
        "id": "call_6",
        "type": "function",
        "function": {
          "name": "calculate",
          "arguments": "{\"expression\":\"37203 / 758.6\"}"
        }
      }
    ]
  },
  {
    "role": "tool",
    "tool_call_id": "call_5",
    "content": "{\"result\":48.48599956587801}"
  },
  {
    "role": "tool",
    "tool_call_id": "call_6",
    "content": "{\"result\":49.041655681518584}"
  },
  {
    "role": "assistant",
    "tool_calls": [
      {
        "id": "call_7",
        "type": "function",
        "function": {
          "name": "calculate",
          "arguments": "{\"expression\":\"49.041655681518584 - 48.48599956587801\"}"
        }
      }
    ]
  },
  {
    "role": "tool",
    "tool_call_id": "call_7",
    "content": "{\"result\":0.5556561156405735}"
  },
  {
    "role": "assistant",
    "content": "Millbrook has a higher population density by 0.6 residents per square kilometre. ANSWER: Millbrook | 0.6"
  }
]
```

**My arithmetic:**

Population density = Population / Area

Brightwater:
44675 / 921.4 = 48.486 residents/km2

Millbrook:
37203 / 758.6 = 49.042 residents/km2

Difference:
49.041656 - 48.486000 = 0.555656

Rounded to one decimal place:
0.6 residents/km2

The higher density is in Millbrook.

I used NR-16 because it corrects Brightwater's area from 912.4 to 921.4 km2. The corrected value must be used.

Final answer: Millbrook | 0.6


### A4 — Why I split it this way

<!-- one other way to split the task, and why you did not choose it -->

---

## Part B — Build it and run it (Medium)

### What I changed from Part A while coding

<!-- what changed in your plan, and why. "None" is an answer too. -->

### B2 — Tests

```text
<!-- pytest output -->
```

### B3 — The loop

- Round limit, and why:
- Where the logs are:
- Real Q-09 conversation compared with my A3 version, what is different:

### B4 — Three strategies

**What I kept the same:**

<!-- model, temperature, k, prompts, answer format, anything else -->

**My rule for a correct answer** (written before the first full run):

| Run | Strategy | Correct /20 | Unanswered | Input tokens | Output tokens | Median seconds per question | Total seconds | Cost ($) |
|---|---|---:|---:|---:|---:|---:|---:|---:|
| 1 | S1 direct | | | | | | | |
| 1 | S2 think and vote (k = ) | | | | | | | |
| 1 | S3 tools | | | | | | | |
| 2 | S1 direct | | | | | | | |
| 2 | S2 think and vote (k = ) | | | | | | | |
| 2 | S3 tools | | | | | | | |

**Per question (run 1):**

| Question | S1 | S2 | S3 |
|---|:---:|:---:|:---:|
| Q-01 | | | |
| Q-02 | | | |
| Q-03 | | | |
| Q-04 | | | |
| Q-05 | | | |
| Q-06 | | | |
| Q-07 | | | |
| Q-08 | | | |
| Q-09 | | | |
| Q-10 | | | |
| Q-11 | | | |
| Q-12 | | | |
| Q-13 | | | |
| Q-14 | | | |
| Q-15 | | | |
| Q-16 | | | |
| Q-17 | | | |
| Q-18 | | | |
| Q-19 | | | |
| Q-20 | | | |

---

## Part C — Explain and choose (Hard)

### C1 — Does the tool description matter?

| Question | Calls (your description) | Calls (vague) | Answer (yours) | Answer (vague) |
|---|---:|---:|---|---|
| | | | | |

### C2 — Three mistakes explained

### C3 — Which strategy I would use, and what would change it

### C4 — How sure I can be

### Final questions

1.
2.
3.
4.
5.
6.
7.
8.
