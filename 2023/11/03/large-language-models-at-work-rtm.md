# Large Language Models at Work RTM

Keeping [with](https://vladris.com/blog/2019/10/16/programming-with-types-rtm.html)
[tradition](https://vladris.com/blog/2021/06/21/data-engineering-on-azure-rtm.html),
I'm writing the RTM post for Large Language Models at Work. The book is done.
Now available [on Kindle](https://www.amazon.com/dp/B0CLSSM8RL).

## Self-publishing

I decided not to contact a publisher this time around, for a couple of reasons:
First, I didn't want the pressure of a contract and timelines (though looking
back, I did finish this book faster than the previous two); Second, I had no
idea if I will be able to write something that is still valuable by the time
the book is done, considering the speed of innovation. More on this later.

I authored the book in the open, at <https://vladris.com/llm-book/> and
self-published [on Kindle](https://www.amazon.com/dp/B0CLSSM8RL). Maybe I will
look into making it a print book at some point, for now I'm keeping it digital.

Amazon offers a nice set of tools to import and format ebooks, but they have
some big limitations - for example, no support for formatting tables, footnotes
etc. I also couldn't convince the tool the code samples should be monospace on
import so I had to manually re-set the font on each. The book has a few
formatting glitches because of these limitations, which make me reluctant to
look into a print book as I expect I will need to do a lot more manual tweaking
for the text to look good in print.

## Speed of innovation

I mused about this in chapter 10: Closing Thoughts. I'll repeat it here as it
perfectly highlight why it is impossible to pin down this strange new world of
AI.

I started writing the book in April 2023. When I picked up the project, GPT-4
was in private preview, with GPT-3.5 being the most powerful globally available
model offered by OpenAI. Since then, GPT-4 opened to the public.

In June, OpenAI announced Functions - fortunately, this happened just before I
started working on chapter 6, Interacting with External Systems. Before
Functions, the way to get a large language model to connect with native code was
through few-shot learning in the prompt, covered in the Non-native functions
section. Originally, I was planning to focus exclusively on this implementation.
Of course, built-in support makes it easier to specify available functions and
the model interaction is likely to work better - since the model has been
specifically trained to "understand" function definitions and output correct
function calls.

In August, OpenAI announced fine-tuning support for `gpt-3.5-turbo`. When I was
writing the first draft of chapter 4, Learning and Tuning, the only models that
used to support fine-tuning were the older GPT-3 generation models: Ada,
Babbage, Currie, and Davinci. This was particularly annoying, as the quality of
output produced by these models is way below `gpt-3.5-turbo` levels. Now, with
the newer models having fine-tuning support, I had to rewrite the Fine-tuning
section.

`text-davinci-003` launched in November of 2022, while `gpt-3.5-turbo` launched
on March 1st 2023. When I started writing the book, `text-davinci-003` was
backing most large language model-based solutions across the industry, and
migrations to the newer `gpt-3.5-turbo` were underway. `text-davinci-003` is
deprecated to be removed by January 4, 2024 (to be replaced by
`gpt-3.5-turbo-instruct`), and the industry is moving to adopt GPT-4. I had to
update several code samples from `text-davinci-003` to `gpt-3.5-turbo-instruct`.

No idea how long the code samples will keep working or when OpenAI will decide
to deprecate `gpt-3.5-turbo` or introduce an even more powerful model with
capabilities not covered in the book.

## Time(lessness)

While some of the code examples will not age well as new models and APIs get
release, the underlying principles of working with large language models that I
walked through in this book - prompt engineering, memory, interacting with
external systems, planning, and so on - will be relevant for a while.
Understanding these fundamentals should help anyone ramp up in the space.

This is an exciting new field, that is going to see a lot more innovation in
the near future. But I expect some of these fundamentals to carry on, in one
shape or another. I hope the topics discussed in this book to remain
interesting for long after the specific models used in the examples become
obsolete.

## Excertps

Like with my previous books, I've been publishing excerpts as shorter,
stand-alone reads. This might sound a bit strange in this case, as the book
is already all online. But I figured it will hopefully help reach more
people, and I did some work on each excerpt to remove references to other
parts of the book so they can, indeed, be read wihtout context. I published
all of these on Medium:

* [N-shot Learning](https://medium.com/@vladris/n-shot-learning-f9bc0d670a41)
* [Embeddings and Vector Databases](https://medium.com/@vladris/embeddings-and-vector-databases-732f9927b377)
* [Interacting with External Systems](https://medium.com/@vladris/interacting-with-external-systems-1951e820b2e7)
* [Planning](https://medium.com/@vladris/planning-d00cc124868f)
* [Adversarial LLM Attacks](https://medium.com/@vladris/adversarial-llm-attacks-17ba03621e61)

I hope you enjoy the book! Check it out here: [Large Language Models at Work](https://www.amazon.com/dp/B0CLSSM8RL).
