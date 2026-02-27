# Using the Imagination Engine Skill

> How to use the kidblocks-engine skill with OpenClaw or any LLM.

## What the Skill Does

The kidblocks-engine skill is a detailed instruction set that teaches an AI to generate complete, self-contained HTML5 apps from natural language. It's designed for children ages 5-10 using a touchscreen, but the generated apps work in any browser.

## Quick Start

### Option 1: OpenClaw

```bash
# Install the skill
mkdir -p ~/.openclaw/workspace/skills/kidblocks-engine
cp skill/kidblocks-engine/SKILL.md ~/.openclaw/workspace/skills/kidblocks-engine/

# Use it
# Tell your agent: "Read the kidblocks-engine skill and make a cat maze game for age 7"
```

### Option 2: Any LLM API

```python
import anthropic

# Read the skill
with open("skill/kidblocks-engine/SKILL.md") as f:
    skill = f.read()

client = anthropic.Anthropic()
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    system=skill,
    messages=[{
        "role": "user",
        "content": '{"description": "cat maze game", "studio": "games", "age": 7, "safety": "standard"}'
    }],
    max_tokens=8000
)

# Parse the JSON response
import json
result = json.loads(response.content[0].text)
# result["html"] — complete HTML app
# result["logic"] — visual programming data
```

### Option 3: Paste into ChatGPT/Claude

1. Copy the contents of `SKILL.md`
2. Paste as the system prompt (or first message)
3. Ask: "Make a piano app for a 6 year old"
4. Copy the JSON output
5. Save the `html` field as an `.html` file and open in a browser

## Input Format

The skill expects a JSON object as the user message:

```json
{
  "description": "what the child asked for",
  "studio": "games|stories|music|art|science|tinker",
  "age": 7,
  "safety": "strict|standard|relaxed"
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `description` | Yes | Natural language description of what to build |
| `studio` | No | Which studio context (defaults to games) |
| `age` | No | Child's age, 5-10 (defaults to 7) |
| `safety` | No | Content safety level (defaults to standard) |

## Output Format

The skill instructs the AI to return a JSON object:

```json
{
  "html": "<!DOCTYPE html><html>...complete self-contained app...</html>",
  "logic": {
    "things": [
      {
        "name": "Player",
        "icon": "🐧",
        "props": [
          { "label": "Speed", "type": "slider", "min": 1, "max": 10, "value": 5 }
        ]
      }
    ],
    "rules": [
      { "when": "Player touches a star", "then": "Score goes up by 1" },
      { "when": "All stars collected", "then": "You win!" }
    ]
  }
}
```

### The `html` Field

A complete, self-contained HTML page. All CSS and JS are inline. No external dependencies. No images — all visuals use emoji, Canvas API, or CSS. Sound uses Web Audio API.

### The `logic` Field

A visual programming representation of the app's behavior:
- **things**: Objects/characters with adjustable properties (sliders, toggles, colors)
- **rules**: When/then pairs in kid-friendly language

KidBlocksOS displays this in a drawer panel so kids can see "how it works." If you're not using KidBlocksOS, you can ignore this field or use it for educational display.

## Studios

The skill defines 6 studios, each with specific patterns:

### Games (Game Forge)
Patterns: platformer, catcher, maze, whack-a-mole, pong, runner, memory, target practice

### Stories (Story Weaver)
Patterns: branching narrative, mad libs, comic strip, adventure

### Music (Beat Lab)
Patterns: piano, beat maker, sequencer, sound board, theremin, music box

### Art (Canvas Magic)
Patterns: freehand drawing, stamp tool, color mixer, pixel art, kaleidoscope, fireworks, pattern maker

### Science (Lab Explorer)
Patterns: solar system, ecosystem, weather sim, body explorer, chemistry mixer, dinosaur timeline, gravity sim

### Tinker (Gadget Shop)
Patterns: calculator, clock/timer, flashcards, fortune teller, dice roller, color palette, simple animations

## Content Safety

The skill includes explicit safety rules:

**Never generates:**
- Violence, weapons, death
- Sexual content
- Drugs, alcohol
- Bullying, hate speech
- Horror, jump scares
- Gambling mechanics

**Safe reinterpretation:**
| Child says | AI generates |
|-----------|-------------|
| "kill enemies" | "help friends" |
| "gun game" | "water balloon launcher" |
| "scary monster" | "friendly monster who needs help" |
| "war game" | "pillow fort battle" |
| "fight" | "dance-off" |

## Performance Notes

The skill targets Raspberry Pi 5 hardware:
- DOM elements < 200 for games
- `requestAnimationFrame` (not `setInterval`)
- Canvas clear-and-redraw pattern
- Touch event throttling (16ms minimum)
- CSS transforms/opacity only (GPU accelerated)
- Target 30fps

These constraints produce lightweight apps that run well everywhere.

## Testing

To test changes to the skill:

1. Edit `SKILL.md`
2. Use the skill as a system prompt
3. Test with a variety of prompts:
   - "make a penguin platformer" (games)
   - "tell me a story about a brave fox" (stories)
   - "I want a drum machine" (music)
   - "drawing app with rainbow brush" (art)
   - "show me the solar system" (science)
   - "build a calculator" (tinker)
4. Verify output is valid JSON
5. Open the HTML in a browser at 1024x600
6. Test touch interactions (use Chrome DevTools device mode)
7. Verify no external resource requests (Network tab should be empty)

## Contributing

See the main [README](../README.md) for contribution guidelines.

Key areas:
- Add new patterns to existing studios
- Improve safety reinterpretation rules
- Better age adaptation
- Performance optimization hints
- Localization support
