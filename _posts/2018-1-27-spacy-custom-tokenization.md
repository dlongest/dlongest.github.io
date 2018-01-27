---
layout: post
title: Adding Custom Tokenization Rules to spaCy
tags: machine learning, spacy, nlp
description: This post discusses how to add tokenization rules to spaCy while preserving the existing default rules.
---
The part of machine learning that has always fascinated me is natural language processing (NLP).  I'm somehow drawn to problems whose input is text from some source though I'm no expert in NLP at all.  I've been playing with spaCy, an incredibly powerful Python library for common NLP tasks.  I admire spaCy's focus: instead of offering a variety of options and algorithms (like NLTK), spaCy aims to implement ones that are deemed the most effective by the maintainer.  

While playing with spaCy, I came across a situation that I felt should have been easy but I just couldn't work out the right way to extend spaCy to deal with it.  In the text I was dealing with, the author had organized it as basically an ordered list.  So the text read "1.Liaising with blahblahblah. 2.Developing the blahblahalbha" and so on.  The challenge was that out of the box spaCy was not splitting "1.Liaising" into "1", ".", "Liaising" like I needed/expect.  Okay, simple enough: spaCy's docs discuss tokenization so I immediately realized I needed to add a prefix search:

```python

def create_custom_tokenizer(nlp):

	prefix_re = re.compile(r'[0-9]\.')
	
	return Tokenizer(nlp.vocab, prefix_search = prefix_re.search)

nlp = spacy.load('en')
nlp.tokenizer = custom_tokenizer(nlp)
	
```


This worked great as far as my custom text was concerned but now other things spaCy was previously splitting properly were no longer being split correctly e.g. "(for" was formerly split into "(", "for" but was now retained as the single token "(for".  What I wanted to do was simply add my tokenization rule to spaCy's set.  It also appeared I'd lost the other tokenization rules as well (the suffix and infix ones).  Losing those rules isn't wholly surprising (since I didn't pass them to the `Tokenizer` I instantiated) but it wasn't clear to me how to preserve those rules. 

Searching the web was fruitless for a few hours but finally I came across <a href="https://github.com/explosion/spaCy/issues/1251">this issue</a> that unlocked the secret to my problem.  The problem as described was one I was also also wrestling with: incorrect spacing around a parentheses i.e. "portfolio(Stocks" should have been split into "portfolio", "(", "Stocks" but it was being retained as one token.  Shown in the issue was a few keys.

First, you can access spaCy's bulit-in prefix, suffix, and infix entries as:

```python

default_prefixes = nlp.Defaults.prefixes
default_infixes = nlp.Defaults.prefixes
default_suffixes = nlp.Defaults.suffixes

```

Each of these types is a tuple of Regex patterns.  If you look at spaCy's <a href="https://github.com/explosion/spaCy/blob/master/spacy/util.py">util.py</a>, locate the methods `compile_prefix_regex`, `compile_suffix_regex`, and `compile_infix_regex`.  You'll see that these methods combine each of the patterns in the tuple into a single Regex pattern that is then compiled:

```python

def compile_prefix_regex(entries):
    if '(' in entries:
        # Handle deprecated data
        expression = '|'.join(['^' + re.escape(piece)
                               for piece in entries if piece.strip()])
        return re.compile(expression)
    else:
        expression = '|'.join(['^' + piece
                               for piece in entries if piece.strip()])
        return re.compile(expression)


def compile_suffix_regex(entries):
    expression = '|'.join([piece + '$' for piece in entries if piece.strip()])
    return re.compile(expression)


def compile_infix_regex(entries):
    expression = '|'.join([piece for piece in entries if piece.strip()])
    return re.compile(expression)

```

Once we learn this fact, it becomes more obvious that what we really want to do to define our custom tokenizer is add our Regex pattern to spaCy's default list *and* we need to give `Tokenizer` all 3 types of searches (even if we're not modifying them).  My custom tokenizer factory function thus becomes:

```python

def create_custom_tokenizer(nlp):
    
    my_prefix = r'[0-9]\.'
    
    all_prefixes_re = spacy.util.compile_prefix_regex(tuple(list(nlp.Defaults.prefixes) + [my_prefix]))
    
    # Handle ( that doesn't have proper spacing around it
    custom_infixes = ['\.\.\.+', '(?<=[0-9])-(?=[0-9])', '[!&:,()]']
    infix_re = spacy.util.compile_infix_regex(tuple(list(nlp.Defaults.infixes) + custom_infixes))
    
    suffix_re = spacy.util.compile_suffix_regex(nlp.Defaults.suffixes)   
    
    return Tokenizer(nlp.vocab, nlp.Defaults.tokenizer_exceptions,
                     prefix_search = all_prefixes_re.search, 
                     infix_finditer = infix_re.finditer, suffix_search = suffix_re.search,
                     token_match=None)
```

I hope this saves people time in the future.  spaCy is a great library and its documentation is quite good.  If I'm able to figure out their website generation, I hope to offer this example to their documentation since at least to me this was a non-obvious situation.
