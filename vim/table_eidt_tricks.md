# Tricks When Editing Tables In Vim

- [date]: Sun Feb 5 2023
- [about] : vim, tables, column mode

To easily create and maintain tables in vim is not easy. But I think it's a shame to use plugin for this job, so I conclude some tricks I found useful when editing tables.

---

## Tip 1: Virtual Edit
Use `set virtualedit=block` to enable virtual edit feature, default setting is `set virtualedit=`, for me, `vim.o.virtualedit="block"` is my default. `set virtualedit=all` is pretty natural when you focusing on editing table, though not quite suitable for most of the editing experience.

**Why this is important?** Here is an explanation from `:h virtualedit`
- Virtual editing means that the cursor can be positioned where there is
no actual character.  This can be halfway into a tab or beyond the end
of the line.  Useful for selecting a rectangle in Visual mode and
editing a table.

To summarize, `virtualedit` let you put cursor on anywhere.

I use this to add a column of `|` as right bound. Suppose you have this unfinished table:
```
1 | 🌸            | 🌸       | 
2 | 🌸🌸          | 🌸🌸     | 
3 | 🌸🌸🌸        | 🌸🌸🌸   | 
4 | 🌸🌸🌸🌸  =>  | 🌸🌸🌸🌸 | 
5 | 🌸🌸🌸        | 🌸🌸🌸   | 
6 | 🌸🌸          | 🌸🌸     | 
7 | 🌸            | 🌸       | 
```

Then to add the right margin: `1GA space space space space <c-v> 6j r|`

## Tip 2: Replace And Virtual Replace
So what is `Replace mode`?

In Replace mode, one character in the line is deleted for every character you
type.  If there is no character to delete (at the end of the line), the
typed character is appended (as in Insert mode).  Thus the number of
characters in a line stays the same until you get to the end of the line.

Virtual Replace mode is similar to Replace mode, but instead of replacing
actual characters in the file, you are replacing **screen real estate**, so that
characters further on in the file never appear to move.


If a <NL> is typed, a line break is inserted and no character is deleted.
To edit inside cell in **insert mode**, it's a pain to keep the cell the same size as before.So the trick is to use **Replace mode** or **Virtual Replace mode**. Enter `R` to enter replace mode, and press `gR` to enter virtual replace mode which virtual additional feature is quite useful to handle with `tab`

```
+-------+        +-------+
| one   |        | five? |
+-------+        +-------+
| two   |        | four? |
+-------+        +-------+
| three |   =>   | three?|
+-------+        +-------+
| four  |        | two?  |
+-------+        +-------+
| five  |        | one?  |
+-------+        +-------+
```

## Tip 3: Yank And Paste In Block Mode
Yank in visual block mode is quite simple, you visual select the range and yank it. Paste would work in block mode.

- To extend the space in one cell
Keep pasting the second column counted from right to left.
```
+-------+           ------          +-------------------+
| one   |                           | one two three ?   |
+-------+           ------          +-------------------+
| two   |                           | two three four ?  | 
+-------+           ------          +-------------------+
| three |  paste             =>     | three four five   |
+-------+           ------          +-------------------+
| four  |                           | four five six     |
+-------+           ------          +-------------------+
| five  |                           | five six seven    |
+-------+           ------          +-------------------+
```
- To paste a buddy table
I draw all the tables above using this trick.

- To add more columns
Still select the suitable range of the block and paste.
```
+-------+            -------+           +-------+-------+-------+-------+-------+-------+
| one   |             one   |           | one   | one   | one   | one   | one   | one   |
+-------+            -------+           +-------+-------+-------+-------+-------+-------+
| two   |             two   |           | two   | two   | two   | two   | two   | two   |
+-------+  select    -------+  paste    +-------+-------+-------+-------+-------+-------+
| three |  ======>    three | ========> | three | three | three | three | three | three |
+-------+            -------+           +-------+-------+-------+-------+-------+-------+
| four  |             four  |           | four  | four  | four  | four  | four  | four  |
+-------+            -------+           +-------+-------+-------+-------+-------+-------+
| five  |             five  |           | five  | five  | five  | five  | five  | five  |
+-------+            -------+           +-------+-------+-------+-------+-------+-------+
```

## Tip 4: Replace Command
I sometimes use command to modify every line. For example to cleanup ill format tables looks like below, I need to delete the last character from every line:
```'
| hello |
| I |
| am |
| ? |
```
The command is `[auto range]s/.$//`

## Tip 4: Shift Blocks
Sometimes I plan the arrangement badly, and there is no space for more graphs:
```
       I want to
       add between
       here
          |
          ▼
+-------+    +-------+
| one   |    | one   |
+-------+    +-------+
| two   |    | two   |
+-------+    +-------+
| three |    | three |
+-------+    +-------+
| four  |    | four  |
+-------+    +-------+
| five  |    | five  |
+-------+    +-------+

```

The tricks here is to shift a block to the right: `<C-V>` and select the whole block and `>>` to do the shifting.
```

               I want to
               add between
               here
                  |
                  ▼
+-------+                    +-------+
| one   |                    | one   |
+-------+                    +-------+
| two   |                    | two   |
+-------+                    +-------+
| three | ================== | three |
+-------+                    +-------+
| four  |                    | four  |
+-------+                    +-------+
| five  |                    | five  |
+-------+                    +-------+

```
