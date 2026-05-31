# urc
This is the specification for `urc`, a user-readable configuration format.
> File extension: `.urc`

## Table of contents
* [The goal](#the-goal)
* [Format features](#format-features)
* [Quick look](#quick-look)
* [Format rules](#format-rules)
* [Basic implementation idea](#basic-implementation-idea-python)
* [Why is urc not strict?](#why-is-urc-not-strict)
* [Some cool things you can do](#some-cool-things-you-can-do)

## The goal
To create a configuration format that even non-technical users can directly interact with while still being easy to work with by developers.

## Format features
- ~~Blazingly fast 🚀~~
- **Readable and understandable even for non-technical users:** If regular users can edit configuration files like how technical/power users can, you can save time by not implementing a dedicated GUI for all of your application settings. You can have it so that, in your app settings, the "Advanced Settings" section would be just using the `urc` format.
- **Self-documented:** Gone are the days when you are staring at a configuration file and you absolutely have no idea what any of those mean or do. Or the config file has tons of comments for explanation. In `urc`, even the users AND applications can inject their own text to the config file.
- **Robust, isolated, and foolproof:** It's very hard for the user to accidentally break the configuration unless they intentionally want to. `urc` is separated by "blocks" for isolation. The parser processes the config block by block. If the user broke one block, it won't affect the rest of the configuration. Your application can then decide what to do with the broken block. Unlike JSON or YAML where one mistake could completely ruin everything.
- **Parsing flexibility:** Since the nature of `urc` is sequential and isolated, it's very easy to make cool things with the parser:
  - Block by block checking for extreme user input validation.
  - You can do optimizations like lazy evaluation.
  - If the current block is broken, you are free to do whatever you want with it. You can throw an error to the user, then proceed to the next blocks, or you can also just stop going through the rest of the blocks entirely.
  - You can even put custom rules like "ignore the next block", "only evaluate this block if the block there has this value". Which allows for the user to do some very rudimentary programming without knowing programming. They just read the config like English and follow the instructions. Of course you can also do this on other formats like JSON or YAML but that's harder for the users.
  
  Note that `urc` doesn't have to be sequential if you are implementing your own parser. Like you can make an optimized parser that processes the entire config in one go. Useful if you have a really big config and you don't really care about validating block by block.
- **Syntax highlighting:** `urc` also supports syntax highlighting like in Markdown code blocks intended for more technical users. Because of this, you can write like a text-based Jupyter Notebook where the blocks are the "cells".
- **Highly extensible:** The `urc` format is very simple and minimal. It's very easy to extend and change the syntax and rules. You can implement your own parser that fits exactly with your application needs.
- **User-friendly:** Lastly and most importantly, the configuration syntax will not scare your users away 🤩

## Quick look
```urc
Enable dark mode? (yes or no) |>>> yes

|======================================

What should be the font size? (From 1 to 100)
|>>> 13

|======================================

What should be the application theme?
Choices are: default, classic, modern

Theme in light mode |>>> classic
Theme in dark mode |>>> modern
```

## Format rules
Note that this is the original but you are free to add or change any rules for your own implementation.
- **Rule no. 1:** Every constructs must be prefixed by a vertical line/pipe character (`|`). That character is chosen because the user is less likely to type that while still making the configuration look aesthetic.
- **Block separator:** Blocks are separated using `|=====`.
  - It must have at least 5 equals.
  - Any excess equals like `|=====================` is allowed. For readability purposes, put as much equals as to make it aesthetic and easier for the user to read.
  - You can put any text before or after:
    - `|===== Text after`
    - `|=====No whitespace is okay`
    - `Text before |=====New Block`
    - This is not recommended but this is only for resilience. Your config should still work even if the user accidentally typed extra characters.
- **Input:** To accept an input, the syntax is `|>`.
  - It must have at least one `>`.
  - Any excess `>` like `|>>>>>` is allowed. The convention is three: `|>>>`.
  - You can put any text before.
  - Every text after is the actual value up until the line break.
  - You can add any number of inputs in a block.
- **Multiline input:**
  ```urc
  What's your bio? |>>>
  ...
  ...
  Note that indentation is not part of the syntax. Indentation is not very user-friendly.
  <<<|
  ```
  The vertical line at the end instead of start breaks rule no. 1. This is an exception because aesthetic is more important 😉.
- **Input syntax highlighting:** If your user need to input some code: `|py>>> np.array([1, 2, 3, 4, 5])` or:
  ```urc
  |py>>>
  import numpy as np

  a = np.array([10, 20, 30, 40])
  b = np.array([1, 2, 3, 4])

  print(a + b)
  <<<|
  ```

## Basic implementation idea (Python)
```py
import urc

config = {
  "is_dark_mode": True,
  "font_size": 13,
}

urc_string = """
Enable dark mode? (yes or no) |>>> yes

|======================================

What should be the font size? (From 1 to 100)
|>>> 20
"""

# Instantiate from string
parser = urc.Parser.from_string(urc_string)

# Instantiate from file
with open("config.urc") as file:
  parser = urc.Parser.from_file(file)

# You can also pass in an "expectation"
# This is just an idea but maybe this should be done using regex as well
expectation: urc.Expectation = [
  "Enable dark mode? (yes or no)",
  "What should be the font size? (From 1 to 100)",
]
parser = urc.Parser.from_string(urc_string, expect=expectation)

# Loop through the block generator
for index, block in parser.stream_blocks():
  # We assume there's only one input per block
  inp = block.inputs[0]

  if index == 0:
    config["is_dark_mode"] = urc.eval_bool(inp.value)
    
  elif index == 1:
    config["font_size"] = urc.eval_int(inp.value)
```
The code above is only for basic demonstration purposes.
This method can become unmaintainable really fast as you add more config.
We can implement a more scalable and maintainable solution.
Overall, I am not really concerned on how it is implemented. As long as it does the job.
I may or may not create a Python library for this. But I will definitely be making a library for this for TypeScript and Zig.

## Why is `urc` not strict?
As you might have already noticed, I keep mentioning about changing the rules and syntax and making your own implementation. At the end of the day, the goal of `urc` is to make a configuration format that works for both users and developers. If your users understand your implementation, then it's doing its job. Just treat `urc` as a foundation you build upon.

## Some cool things you can do
- You can add a version to your config using the existing functionality:
  ```urc
  version |> 1
  
  |==========================
  
  Rest of the config...
  ```
- Your app can inject metadata to the blocks:
  ```urc
  Dark theme? |>>> yes

  [meta:last_modified:1780217208046]
  [meta:is_ignored:false]
  ```
  The metadata is not a syntax to `urc`. Your application needs to parse it manually.
  Obviously, by doing this, you are making your config look more scary to the user, but it is a choice.
  Alternatively, your app UI for editing the config can automatically hide metadata.
  Or even separate the blocks into different textareas so it's guaranteed that the user won't mess up the blocks.
  Or to go even further, you can make it so that the users can only edit the actual values.
- The user can add their own notes:
  ```urc
  Enable VSync? |>>> yes

  Note to self: vsync is the setting to fix my screen tearing
  ```
- You can treat it like a text-based Jupyter Notebook:
  ```urc
  Import packages:
  |py>
  import numpy as np
  import pandas as pd
  <|
  
  |===============================
  
  Create data:
  |py>
  data = {'Item': ['Apples', 'Bananas', 'Oranges'], 'Price': [1.20, 0.50, 0.80]}
  df = pd.DataFrame(data)
  <|

  |===============================
  
  Calculate:
  |py>
  average = df['Price'].mean()
  print(f"The average price is ${average:.2f}")
  <|
  ```
  Obviously it's not runnable. But it's a quick way to share your code with explanation.
- You can (and should) put everything the user needs to know in the config itself:
  ```urc
  READ ME FIRST!!
  
  How to use this?
  Read the instructions and only change the text after the ">>>".

  Example:
  "Do you want to enable dark mode? (yes or no) |>>> yes"

  You can turn into:
  "Do you want to enable dark mode? (yes or no) |>>> no"

  |========================================================================

  Do you want to enable dark mode? (yes or no) |>>> yes
  ```
  Notice that we used two inputs for the examples. You just have to take that into account when parsing the config and choose to ignore them.

----------------------------------------------------------------------------

Feel free to leave any suggestions and feedback!
I'm looking forward for devs to adopt this in their projects. I personally made this for my own apps to make my life easier but still friendly for my (non-existent) users.
