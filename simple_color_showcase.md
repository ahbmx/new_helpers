To display different **ASCII color combination codes** (technically ANSI escape sequences used in terminals), you can write a Python function that prints foreground and background color combinations along with their corresponding escape codes.

Here's a function that:

* Loops over **foreground** and **background** color codes
* Constructs the appropriate ANSI escape sequences
* Displays the text colored accordingly
* Prints the escape sequence used

### ✅ Python Function: Show ASCII (ANSI) Color Combinations

```python
def show_ansi_color_combinations():
    reset = "\033[0m"

    # ANSI standard color codes (30–37 for foreground, 40–47 for background)
    colors = {
        30: 'Black',
        31: 'Red',
        32: 'Green',
        33: 'Yellow',
        34: 'Blue',
        35: 'Magenta',
        36: 'Cyan',
        37: 'White',
    }

    print("ANSI Color Combinations (Foreground on Background):\n")
    
    for fg_code in range(30, 38):
        for bg_code in range(40, 48):
            sequence = f"\033[{fg_code};{bg_code}m"
            fg_color = colors[fg_code]
            bg_color = colors[bg_code - 10]
            print(f"{sequence} {fg_color} on {bg_color} \033[0m -> {repr(sequence)}")

    print("\nNote: Some terminals may not support all color combinations.")

# Example usage
show_ansi_color_combinations()
```

---

### 🔍 Explanation:

* `\033[` is the escape character that introduces ANSI codes.
* `[<fg>;<bg>m` sets the foreground and background color.
* `30–37` → standard foreground colors
* `40–47` → standard background colors
* `\033[0m` resets the color to default after each line.

---

### 🖥️ Sample Output (visual):

In a compatible terminal, it would display lines like:

```
 Red on Black  -> '\x1b[31;40m'
 Green on Blue -> '\x1b[32;44m'
 ...
```

Let me know if you also want to include **bright** or **256-color** modes.
