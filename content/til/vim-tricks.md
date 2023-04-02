---
title: "Vim tricks"
date: 2023-04-02T15:00:29-07:00
draft: false
---

I've been using (neo)vim as my main IDE for the last several years. If you're interested in my config, you can find it [here](https://github.com/hmcguinn/dot-files/tree/master/nvchad). These are some tricks that I've picked up from using vim.


### Wrap lines

Sometimes, I'll write long-form content in vim (like this blog). It gets a lot easier to edit if you use:

```
:set wrap
```

To undo it:

```
:set nowrap
```

### Visual block mode 


#### Deletion
You can use ``Ctrl-v`` to enter visual block mode. This is useful when you want to delete several part of consecutive lines. For example, given the text:

```
1. asd visual block example
2. asd a second line 
3. asd more text here
```

Place your cursor on the "a" in the first "asd". Enter visual block mode with ``Ctrl-v``. Select the entireword with ``e``. Select the next two lines and the space in between with ``jjl``. Then you can delete across multiple lines with ``x``!

We're left with:
```
1. visual block example
2. a second line 
3. more text here
```

#### Insertion 
You can also insert onto several consecutive lines. Again, use ``Ctrl-v`` to enter visual block mode. Then ``I`` (shift-i) to enter insert mode. Once you've added your text, press ``Esc`` or ``Ctrl-[`` to propogate it to all selected blocks. Note, that ``i`` won't get you into insert-mode here, it needs to be ``I``.

### Move split to tab

To move a split to its own tab you can use ``Ctrl-w T``.

