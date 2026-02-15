# Structured-Interactive-Narrative-SIN-
SIN turns AID into a pure CYOA.  It is designed to output choices (default 3 choices), like in a CYOA novel, every few responses (default 5).  It works best with DeepSeek, though I have tested it on most of the models.
COMMANDS:
/choice : forces a choice to appear in the next response
/set freq <#> : sets how many narrative continues the AI goes through before forcing a choice (default 4, so the AI returns choices on the 5th turn).
/set choices <#> : sets how many choices the AI returns.  I recommend leaving it at 3.  The AI sometimes struggles filling out lists longer than that, especially the smaller models.

Installation Instructions:
Go to the 'scripts' section of your AID scenario, and replace Library, Input, Context, and Output with the following:

INPUT:
```
const modifier = (text) => {
  const trimmed = text.trim();
  const lc = trimmed.toLowerCase();
  // Handle numeric selections (1 through the number of available options)
  const numeric = /^\d+$/.test(trimmed);
  if (numeric && Array.isArray(state.options) && state.options.length > 0) {
    const idx = parseInt(trimmed, 10) - 1;
    if (idx >= 0 && idx < state.options.length) {
      const choiceText = state.options[idx];
      if (choiceText) {
        return { text: 'You choose to ' + choiceText };
      }
    }
  }
  // Handle the /choice command
  if (lc === '/choice') {
    state.forceChoice = true;
    return { text: ' ' };
  }
  // Handle configuration commands
  const freqMatch = lc.match(/^\/set\s+freq\s+(\d+)/);
  if (freqMatch) {
    const value = parseInt(freqMatch[1], 10);
    if (!isNaN(value) && value > 0) {
      state.choiceFrequency = value;
      state.responseCounter = 0;
      state.showChoicesNext = false;
      state.forceChoice = false;
      state.message = 'Choice frequency updated to ' + value + ' narrative response' + (value === 1 ? '' : 's') + ' between choices.';
    } else {
      state.message = 'Invalid frequency value. Please use a positive number.';
    }
    return { text: ' ' };
  }
  const choicesMatch = lc.match(/^\/set\s+choices\s+(\d+)/);
  if (choicesMatch) {
    const value = parseInt(choicesMatch[1], 10);
    if (!isNaN(value) && value > 0) {
      state.numOptions = value;
      // Also reset stored options to avoid stale indices when reducing the count
      state.options = [];
      state.message = 'Number of options to display updated to ' + value + (value === 1 ? ' choice.' : ' choices.');
    } else {
      state.message = 'Invalid choices value. Please use a positive number.';
    }
    return { text: ' ' };
  }
  return { text };
};

modifier(text);
```

CONTEXT:
```
const modifier = (text) => {
  let instruction = '';
  if (state.forceChoice || state.showChoicesNext) {
    // Request only choices: no narrative
    instruction =
      '\n\n[System Note: In your next response, do not continue the story. Provide exactly three complete choices the player could take next, each fully described on a single line with no omissions. If you cannot think of three obvious choices, invent plausible options until you have three. After listing the three options, stop.]\n';
  } else {
    // Request narrative and explicitly suppress choices
    instruction =
      '\n\n[System Note: Continue narrating the story. Do not list any choices or options in this response. The script will tell you when to list choices.]\n';
  }
  return { text: text + instruction };
};

modifier(text);
```

OUTPUT:
```
const modifier = (text) => {
  // Prepend any pending system messages once and clear them
  let prefix = '';
  if (typeof state.message === 'string' && state.message.trim() !== '') {
    prefix = state.message.trim() + '\n';
    state.message = '';
  }
  // Determine whether we need to display choices on this turn
  const force = state.forceChoice === true;
  const showNow = force || state.showChoicesNext === true;
  // Parse any options present
  const { options, positions } = parseOptions(text);
  if (!showNow) {
    // Narrative turn: remove any option lines to avoid partial lists
    let lines = (typeof text === 'string' ? text.trimEnd() : String(text || '')).split('\n');
    if (Array.isArray(options) && options.length > 0) {
      lines = lines.filter((_, idx) => !positions.includes(idx));
    }
    let cleanText = lines.join('\n').trimEnd();
    // If removing option lines results in an empty string, fall back to the
    // original text.  This prevents blank outputs which cause errors on
    // some models.
    if (!cleanText) {
      cleanText = (typeof text === 'string' ? text.trimEnd() : String(text || '')).trim();
    }
    // Increment the narrative counter
    let count = typeof state.responseCounter === 'number' ? state.responseCounter : 0;
    count += 1;
    if (count >= state.choiceFrequency) {
      // Schedule choices for next turn and cap the counter at choiceFrequency until options are shown
      state.showChoicesNext = true;
      state.responseCounter = state.choiceFrequency;
    } else {
      state.responseCounter = count;
    }
    // If a force flag was still set, clear it so it doesn't bleed into future turns
    if (state.forceChoice) {
      state.forceChoice = false;
    }
    return { text: prefix + cleanText };
  }
  // Choice turn: Build the numbered list of options
  let opts = Array.isArray(options) ? options.slice() : [];
  // Remove undefined or empty entries
  opts = opts.filter(opt => typeof opt === 'string' && opt.trim() !== '');
  // Only take the number of options requested
  const desired = (typeof state.numOptions === 'number' && state.numOptions > 0) ? state.numOptions : 3;
  if (opts.length > desired) {
    opts = opts.slice(0, desired);
  }
  // Pad with generic placeholders if we have too few options
  const fallbackList = [
    'Continue along the current path.',
    'Try something unexpected or creative.',
    'Ask for more information or help.'
  ];
  let i = 0;
  while (opts.length < desired) {
    opts.push(fallbackList[i % fallbackList.length]);
    i++;
  }
  // Build the numbered list
  const numbered = opts.map((opt, idx) => `[${idx + 1}] ` + opt);
  // Persist the options for input selection (un-numbered)
  state.options = opts;
  // Reset counters and flags for the next cycle
  state.responseCounter = 0;
  state.showChoicesNext = false;
  state.forceChoice = false;
  // Return only the numbered choices (no narrative), with any prefix message
  return { text: prefix + numbered.join('\n') };
};

modifier(text);
```

LIBRARY:
```
function parseOptions(outputText) {
  // Guard against undefined or non-string inputs by converting to a string
  const safeText = (typeof outputText === 'string' ? outputText : String(outputText || ''));
  const lines = safeText.trimEnd().split('\n');
  let options = [];
  let positions = [];
  // First pass: detect explicit enumerated lines (e.g. [1] option or 1. option)
  for (let i = 0; i < lines.length; i++) {
    const line = lines[i];
    // Match either bracketed number or a number followed by punctuation
    const match = line.match(/^\s*(?:\[(\d)\]|(\d+)[\.\)]?)\s*(.*)/);
    if (match) {
      const num = match[1] || match[2];
      const idx = parseInt(num, 10) - 1;
      // Accept enumerations indexed 0 through 8 (numbers 1 through 9)
      if (idx >= 0 && idx <= 8) {
        // Capture the third group safely; it may be undefined if the pattern did not capture
        const matchGroup = (typeof match[3] === 'string' ? match[3] : '');
        let text = matchGroup.trim();
        positions.push(i);
        // If the option text is blank, look to subsequent lines until a blank or new enumeration
        if (!text) {
          let nextIdx = i + 1;
          while (nextIdx < lines.length) {
            // Safely trim the next line; handle undefined entries
            const nextLineRaw = lines[nextIdx];
            const nextLine = (nextLineRaw || '').trim();
            // Stop if we hit a blank line
            if (nextLine === '') {
              break;
            }
            // Stop if the next line is another enumeration
            const enumerationMatch = nextLine.match(/^\s*(?:\[(\d)\]|(\d+)[\.\)]?)/);
            if (enumerationMatch) {
              break;
            }
            // Otherwise, append the next line to the current option
            text += (text ? ' ' : '') + nextLine;
            positions.push(nextIdx);
            nextIdx++;
          }
          text = text.trim();
        }
        options[idx] = text;
      }
    }
  }
  // If fewer than three options detected, fall back to last three non‑empty lines
  const desiredFallback = (typeof state === 'object' && typeof state.numOptions === 'number' && state.numOptions > 0 ? state.numOptions : 3);
  if (!options || options.filter(x => x !== undefined).length < desiredFallback) {
    options = [];
    positions = [];
    // Determine the index after the last blank line
    let start = lines.length - desiredFallback;
    for (let i = lines.length - 1; i >= 0; i--) {
      const current = lines[i] || '';
      if (current.trim() === '') {
        start = i + 1;
        break;
      }
    }
    const candidateIndices = [];
    for (let i = start; i < lines.length; i++) {
      const current = lines[i] || '';
      if (current.trim() !== '') {
        candidateIndices.push(i);
      }
    }
    if (candidateIndices.length >= 1) {
      const lastN = candidateIndices.slice(-desiredFallback);
      lastN.forEach((idx) => {
        const lineRaw = lines[idx] || '';
        // Strip leading numbers and punctuation (e.g. "2." or "3:")
        const stripped = lineRaw.replace(/^\s*(?:\d+[\)\.:]\s*)?/, '').trim();
        options.push(stripped);
        positions.push(idx);
      });
    }
  }
  // Do NOT persist options here; options will be stored in the Output Modifier
  return { options, positions };
}

// Initialise persistent state variables if they don’t exist.  New settings
// include `choiceFrequency` (how many narrative responses between choices)
// and `numOptions` (how many options to show).  Players can adjust these
// using `/set freq <#>` and `/set choices <#>` in the Input Modifier.
if (!Array.isArray(state.options)) {
  state.options = [];
}
if (typeof state.forceChoice !== 'boolean') {
  state.forceChoice = false;
}
// Track how many narrative responses have occurred since the last set of options
if (typeof state.responseCounter !== 'number') {
  state.responseCounter = 0;
}
// Whether the next response should consist only of choices (scheduled by the frequency counter)
if (typeof state.showChoicesNext !== 'boolean') {
  state.showChoicesNext = false;
}

// How many narrative responses should occur before choices are shown.  The
// default is 4, meaning the AI narrates four responses and then presents
// options on the following turn.  Players can adjust this using the
// `/set freq <number>` command.  Clamp to at least 1.
if (typeof state.choiceFrequency !== 'number' || isNaN(state.choiceFrequency) || state.choiceFrequency < 1) {
  state.choiceFrequency = 4;
}

// How many choices to display on a choice turn.  Default is 3.  Players can
// adjust this using the `/set choices <number>` command.  Clamp to at least 1.
if (typeof state.numOptions !== 'number' || isNaN(state.numOptions) || state.numOptions < 1) {
  state.numOptions = 3;
}
```
