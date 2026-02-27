# using the imagination engine skill

the kidblocks-engine skill teaches an AI to generate complete, self-contained HTML5 apps from natural language. it's built for kids ages 5 to 10 on a touchscreen, but the output works in any browser.

## quick start

### option 1: OpenClaw

```bash
mkdir -p ~/.openclaw/workspace/skills/kidblocks-engine
cp skill/kidblocks-engine/SKILL.md ~/.openclaw/workspace/skills/kidblocks-engine/

# then tell your agent:
# "read the kidblocks-engine skill and make a cat maze game for age 7"
```

### option 2: any LLM API

```python
import anthropic, json

with open("skill/kidblocks-engine/SKILL.md") as f:
    skill = f.read()

client = anthropic.Anthropic()
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    system=skill,
    messages=[{
        "role": "user",
        "content": json.dumps({
            "description": "cat maze game",
            "studio": "games",
            "age": 7,
            "safety": "standard"
        })
    }],
    max_tokens=8000
)

result = json.loads(response.content[0].text)
# result["html"] is the complete app
# result["logic"] is the visual programming data
```

### option 3: paste it

1. copy the contents of `SKILL.md`
2. paste as the system prompt or first message in any chat UI
3. ask: "make a piano app for a 6 year old"
4. copy the html from the JSON output
5. save as .html, open in a browser

## input format

send a JSON object as the user message:

```json
{
  "description": "what the child asked for",
  "studio": "games",
  "age": 7,
  "safety": "standard"
}
```

| field | required | notes |
|-------|----------|-------|
| description | yes | natural language, what to build |
| studio | no | games, stories, music, art, science, or tinker. defaults to games |
| age | no | 5 through 10. defaults to 7 |
| safety | no | strict, standard, or relaxed. defaults to standard |

## output format

the skill tells the AI to return JSON only. no markdown, no explanation.

```json
{
  "html": "<!DOCTYPE html><html>...full app...</html>",
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

**html**: complete self-contained page. all CSS and JS inline. no external dependencies. no images. visuals use emoji, Canvas, or CSS. sound uses Web Audio API.

**logic**: visual representation of app behavior for the KidBlocks programming layer. if you don't need it, ignore it.

## studios

### games (Game Forge)
platformer, catcher, maze, whack-a-mole, pong, runner, memory, target practice

### stories (Story Weaver)
branching narrative, mad libs, comic strip, adventure

### music (Beat Lab)
piano, beat maker, sequencer, sound board, theremin, music box

### art (Canvas Magic)
freehand drawing, stamps, color mixer, pixel art, kaleidoscope, fireworks, pattern maker

### science (Lab Explorer)
solar system, ecosystem, weather sim, body explorer, chemistry mixer, dinosaur timeline, gravity sim

### tinker (Gadget Shop)
calculator, clock/timer, flashcards, fortune teller, dice roller, color palette, animations

## content safety

the skill has explicit rules about what never gets generated:

- violence, weapons, death
- sexual content
- drugs, alcohol
- bullying, hate speech
- horror, jump scares
- gambling mechanics

instead of refusing, the AI reinterprets. "gun game" becomes "water balloon launcher". "kill enemies" becomes "help friends". this keeps the creative flow going while keeping the output safe.

## performance notes

the skill targets constrained hardware:

- keep DOM elements under 200 for games
- use requestAnimationFrame, not setInterval
- canvas games should clear and redraw each frame
- throttle touch events to 16ms minimum
- use CSS transform and opacity for animations (GPU accelerated)
- target 30fps

these constraints mean the generated apps are lightweight and run well everywhere, not just on Pi hardware.

## testing changes

if you modify the skill:

1. use it as a system prompt with any model
2. test across all six studios:
   - "make a penguin platformer" (games)
   - "tell me a story about a brave fox" (stories)
   - "I want a drum machine" (music)
   - "drawing app with rainbow brush" (art)
   - "show me the solar system" (science)
   - "build a calculator" (tinker)
3. check that output is valid JSON
4. open the HTML at 1024x600 in a browser
5. test touch (Chrome DevTools device mode)
6. check Network tab for external requests (there should be none)
7. try edge cases that should be caught by safety rules
