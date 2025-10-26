---
layout: post
title:  JSON parsing and good vs bad programming
date:   2025-10-25
tags: [programming, intelligence]
---

## Background

While I was prepping for interviews, I came across an interesting question which was write a JSON parser.  My attempt at solving this question and subsequent review with GPT-5 got me thinking about some meta points about good vs bad programming.

When I first read the question, I thought about it a bit and it was not obvious how to do it at first glance.
It's clearly a problem that involves recursion - JSON string can be a dict whose values are arbitrary JSON objects or an array whose elements are arbitrary JSON objects.

The first challenge is handling delimiters.  For example, if we come across a `{`, we're parsing a dict and need to find the `}` that corresponds to the end of the dict.  We can't just use the next `}` we come across because that could be for an inner dict like if we had `{ "a": { "b" : "c" } }`.

A second challenge is handling whitespace to make it robust to formatting.  For example, `{ "a": { "b": "c" } }` and

```json
{
  "a": {
    "b": "c"
  }
}
```

both represent the same object.

A last challenge is to determine when the encoding is invalid and throw an error.  For example, the following encoding would be invalid `{ "a", "b" }`.

## My bad solution

With all of these in mind, I came up with a general strategy of having a top level `parse` method and then various specific handlers like `parse_string`, `parse_dict`, `parse_array` which would be called in a switch statement depending on current character (ie. `{` would signal to call `parse_dict`, `[` would signal to call `parse_array` and `"` would signal to call `parse_string`, etc).

In order for this approach to work, we need to keep track of our position in the encoding string and each of these methods need to return either a new offset or the number of bytes consumed.  I ended up doing a functional approach and having method signatures like `parse_*(encoding, pos) -> tuple(obj, new_pos)`.  I could have easily returned the number of bytes consumed instead of new position, or made this into a class and had `pos` be a member of that class.  I consider those stylistic differences rather than fundamental differences in approach.

The following illustrations are roughly what I visualized in my mind when I was thinking of this

The first character is `{` so we make call to `parse_dict()` (red)

![json parse 1]({{"/assets/2025-10-21-json-parsing/json-parse-1.png" | relative_url }})

We process a few more characters until we encounter the inner `{`. Pause our position here

![json parse 2]({{"/assets/2025-10-21-json-parsing/json-parse-2.png" | relative_url }})

We recursively call `parse_dict()` (yellow) and which processes its part of encoding and finishes at the next '}' in this case.

![json parse 3]({{"/assets/2025-10-21-json-parsing/json-parse-3.png" | relative_url }})

After has finished `parse_dict()` finishes, we need to update position of the outer `parse_dict()` (red) beyond the span of the inner `parse_dict()` (yellow).

![json parse 4]({{"/assets/2025-10-21-json-parsing/json-parse-4.png" | relative_url }})

*Note that there are probably better ways to visualize this but I wanted to maintain fidelity of what I had in my mind 

Here is what my `parse_dict` function looked like

```python

def _parse_dict(encoded: str, pos: int) -> tuple[dict, int]:
    ret = {}
    pos += 1 # initial '{'
    state = 0 # 0 - key empty, 1 - val empty, 2 - kv added
    key = None
    while pos < len(encoded):
        c = encoded[pos]
        if c in WHITESPACE:
            pos += 1
        elif c == ':':
            if state != 0 or not key:
                raise ValueError("Invalid JSON")
            state = 1
            pos += 1
        elif c == ',':
            if state != 2:
                raise ValueError("Invalid JSON")
            state = 0
            pos += 1
        elif c == '}':
            if state not in (0,2):
                raise ValueError("Invalid JSON")
            return ret, pos + 1
        else:
            if state == 0:
                key, pos = _parse_str(encoded, pos)
            elif state == 1:
                val, pos = _parse(encoded, pos)
                ret[key] = val
                key = None
                state = 2 
            else:
                raise ValueError("Invalid JSON")
```

This code handles character by character and tracks state "externally" outside of the while loop.  The problem here is that it's difficult to do state transitions and validity checks this way.  For example, when handling the `,` we can only have come from state 2 (just added a kv) and we must then reset the state to 0 (key empty).

## Hypothetical path to good solution

I'd like to say that, after taking a proper look at this abomination, I immediately saw the errors in my ways and fixed them.  Unfortunately, I did not.  What ended up happening is that I wrote a few test cases, banged things out until they passed, and called it a day.

I consulted GPT-5 about this problem and it gave me the proper solution, which I will show later.  I'm going to try to work backwards from there to try to construct a sequence of thoughts that could have led to the better design.

Going back to `parse_dict`, something definitely feels off about this code.  The state management feels particularly "wrong".  The key jump I would have needed to make is something like this chain of thought

===================

"Hmm, this state management feels particularly bad"

-> "It's because I'm processing a single character in the body of the while loop and updating external state"
  
-> "Oh, it would be better if I processed a single key-value pair in the body of the while loop instead"

-> "Does this actually work? Yes, because each key-value pair has a predictable structure"

===================

So let's do that. Rewriting it this way we get something like this

```python
def _parse_dict(encoded: str, pos: int) -> tuple[dict, int]:
    ret = {}
    pos += 1 # we're already at opening '{'
    while pos < len(encoded):
        # consume any initial whitespace
        pos = _consume_whitespace(encoded, pos)

        # parse key
        key, pos = _parse_str(encoded, pos)
        pos = _consume_whitespace(encoded, pos)
        
        # check for ':' delimiter
        if encoded[pos] != ':':
            raise ValueError("Invalid JSON")
        pos += 1
        
        # parse value
        value, pos = _parse(encoded, pos)
        pos = _consume_whitespace(encoded, pos)

        # check for closing '}'
        if encoded[pos] == '}':
            return ret, pos + 1
        
        # check for ',' delimiter
        if encoded[pos] != ',':
            raise ValueError("Invalid JSON")
    
    # never found closing '}' so invalid
    raise ValueError("Invalid JSON")
```

This looks much better as we've gotten rid of our external state variables `key` and `state`.  We've had to change how we handle the whitespace though, which is now done with the `_consume_whitespace` method.  

Ideally, another alarm bell should have gone off here.  All of these `_consume_whitespace` calls feel unnecessary, and if hypothetical me had investigated that further, he might have gotten to a better solution.  Such a chain of thought might look like this

===================

"Hmm, this looks better than before but something still feels wrong"

-> "I feel forced to repeat myself with these `_consume_whitespace()` calls plus it feels very brittle.  If I miss one or put it in the wrong spot, it breaks"

-> "What if I just got rid of the whitespace beforehand with preprocessing?"

-> "I can't do that because I could have whitespace inside a string quotation"

-> "What if I also parsed out the strings as part of the preprocessing?  That's the only place where I'd need to preserve whitespace I think"

-> "Since I'm parsing strings, I might as well parse out numbers too since they can be decimal notation"

-> "Maybe I need something more general like preprocessing into tokens..."

===================

This leads us to the well-known two-phased approach from compilers, which is lexical analysis (lexing) then parsing.  The fundamental problem here is that we're still operating on a stream of characters and that character processing is coupled together with our logical parsing.  A cleaner approach would be to preprocess the characters into a stream of logical tokens, and have the actual logic of our parser work on those. 

##  Good solution

At this point, we can introduce the code I got from GPT-5

```python
from dataclasses import dataclass
import re

@dataclass
class Tok:
    kind: str          # "{", "}", "[", "]", ",", ":", "STRING", "NUMBER", "TRUE","FALSE","NULL","EOF"
    val: str | None
    line: int
    col: int

NUM_RE = re.compile(r'-?(0|[1-9]\d*)(?:\.\d+)?(?:[eE][+-]?\d+)?')

def lex(s: str):
    i = 0; line = 1; col = 1
    def adv(n=1):
        nonlocal i, line, col
        for _ in range(n):
            if i < len(s) and s[i] == '\n': line += 1; col = 1
            else: col += 1
            i += 1
    def peek(): return s[i] if i < len(s) else ''

    while i < len(s):
        c = peek()
        if c in " \t\r\n": adv(); continue
        start_line, start_col = line, col
        if c in "{}[],:": adv(); yield Tok(c, None, start_line, start_col); continue
        if c == '"':                      # minimal string with escapes \" \\ (add more if time)
            adv()                         # skip "
            out = []
            while True:
                if i >= len(s): raise SyntaxError(f"Unterminated string at {start_line}:{start_col}")
                c = peek()
                if c == '"': adv(); yield Tok("STRING", "".join(out), start_line, start_col); break
                if c == '\\':
                    adv()
                    e = peek()
                    if e in ['"', '\\', '/']: out.append(e); adv()
                    elif e == 'n': out.append('\n'); adv()
                    elif e == 't': out.append('\t'); adv()
                    elif e == 'r': out.append('\r'); adv()
                    else: raise SyntaxError(f"Invalid escape at {line}:{col}")
                else:
                    if ord(c) < 0x20: raise SyntaxError(f"Control char in string at {line}:{col}")
                    out.append(c); adv()
            continue
        # literals with boundary
        for word, kind in (("true","TRUE"), ("false","FALSE"), ("null","NULL")):
            if s.startswith(word, i):
                j = i + len(word)
                if j == len(s) or not s[j].isalnum():
                    adv(len(word)); yield Tok(kind, word, start_line, start_col); break
        else:
            # number
            m = NUM_RE.match(s, i)
            if m:
                text = m.group(0); adv(len(text)); yield Tok("NUMBER", text, start_line, start_col); continue
            raise SyntaxError(f"Unexpected character {repr(c)} at {start_line}:{start_col}")
    yield Tok("EOF", None, line, col)

class Parser:
    def __init__(self, s: str):
        self._it = iter(lex(s))
        self.tok = next(self._it)

    def _eat(self, kind: str):
        if self.tok.kind != kind:
            raise SyntaxError(f"Expected {kind}, got {self.tok.kind} at {self.tok.line}:{self.tok.col}")
        self.tok = next(self._it)

    def parse(self):
        v = self.value()
        if self.tok.kind != "EOF":
            raise SyntaxError(f"Extra content at {self.tok.line}:{self.tok.col}")
        return v

    def value(self):
        k = self.tok.kind
        if k == "NULL":  self._eat("NULL");  return None
        if k == "TRUE":  self._eat("TRUE");  return True
        if k == "FALSE": self._eat("FALSE"); return False
        if k == "STRING": v = self.tok.val; self._eat("STRING"); return v
        if k == "NUMBER":
            t = self.tok.val; self._eat("NUMBER")
            return int(t) if t.lstrip('-').isdigit() else float(t)
        if k == "{": return self.obj()
        if k == "[": return self.arr()
        raise SyntaxError(f"Unexpected token {k} at {self.tok.line}:{self.tok.col}")

    def obj(self):
        d = {}; self._eat("{")
        if self.tok.kind == "}": self._eat("}"); return d
        while True:
            if self.tok.kind != "STRING":
                raise SyntaxError(f"Object keys must be strings at {self.tok.line}:{self.tok.col}")
            k = self.tok.val; self._eat("STRING")
            self._eat(":")
            d[k] = self.value()
            if self.tok.kind == "}": self._eat("}"); return d
            self._eat(",")

    def arr(self):
        a = []; self._eat("[")
        if self.tok.kind == "]": self._eat("]"); return a
        while True:
            a.append(self.value())
            if self.tok.kind == "]": self._eat("]"); return a
            self._eat(",")

```
The core idea here was to divide the task into two phases.

First phase is lexer, which converts the encoded string into a stream of tokens, and throws out the useless whitespace.  The second phase is the actual parsing, which does the recursion through the `.value()` method.  The `.obj()` method corresponds to my earlier `parse_dict` and also processes a single key-value pair in the body of the while loop.

Overall, I would say this code is pretty well designed with a `._eat()` method which clearly shows the intent of consuming tokens from the stream.  It feels like a very "textbook" solution, which as of now the LLMs greatly excel at.

## Reflections

This post isn't meant to be about JSON parsing or compilers.  There are far better treatments of either of those topics than what I'm capable of writing.  

I'm more interested in these meta questions

1. What goes through the mind of a good programmer vs a bad programmer?
2. Are there tactics that can be used to get from bad design to good design?
3. Are there generalizable lessons that could be valuable to bad programmers or programmers in general? (I don't have anything to teach good programmers after all)

Regarding question 1, I have a hard time believing that a good programmer would ever had ended up with the terrible state transitions in my initial version.  I think they would either 1. immediately recognize the two-phased approach, especially if they were familiar with compilers or 2. started with the  character-by-character processing and eventually iterated to the two-phased approach through awareness/persistence/intelligence.  Option 2 would be more excusable if they hadn't taken a compilers course before.

Regarding question 2 and 3, I would like to believe that some skills can be cultivated through experience and deliberate practice that could move a bad programmer towards being a good one, namely

1. Pattern matching, which can be thought of as expanding the "toolbox".
2. Recognizing when something is off, which can be thought of as cultivating your sense of "smell".
3. Perseverance, which is not stopping until you have completely solved the problem.

This post was mostly an exercise in 2 and 3, where we (ideally) would notice that something was off, start thinking and understanding the problem better, and not give up until we came up with the solution that fixed it.

What are some actionables that bad programmers could take away from this?  This is all I could think of for now

1. Try to extract some general patterns through analysis and reflection for honing your sense of smell.  For this problem, working with the wrong abstraction (characters vs tokens) resulted in us being forced to couple the boring whitespace processing and the "real work" of parsing.

2. Periodically look at old code you wrote and see if anything "smells" bad.  Try rewriting the portions that do.

3. Try to replace suboptimal thought patterns.  For example, if your pattern is "something seems off here" -> "I don't immediately see it, so I'll move on" to "something seems off here" -> "let me try digging deeper into what is going on here"

I'm not sure how useful these tips would be in practice.  They could turn out to be things that "sound reasonable" and end up being useless.  In fact, it's entirely possible that this whole post has been an overanalysis of something completely trivial.

Finally, I should note that these questions stem from one that affects me on a personal level, which is - how could I have written that horrible solution in the first place?  I took a compilers course in college (albeit that was many years ago) but I did some exercises in Nand2Tetris which involved compilers within the last two years, so I've seen the two-phase approach more than once.  I plan to explore that more and other observations about good vs bad programmers in another post.
