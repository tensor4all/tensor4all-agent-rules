# What a table may compare

## One table, one declared piece of work

Every cell in a table answers the same question: the same contraction order, the same shape grid,
the same thread budget, the same dtype. Two orders of one library in one table with a ratio column
between them is not a comparison between implementations, and it reads as one.

## A row is an implementation

Rows name independent implementations - the measured library, a vendor, another language's
library - not internal modes of one. When only one implementation exists, the table has one column,
and that is the honest shape; add the ratio column when a second arm arrives.

## Internal comparisons are a different table

A speedup between two modes of one library (one lane against a pool, one contraction order against
another) is worth measuring, and it is a scaling table: a speedup column, no verdict marks, and the
word "same library" in the text. Never let it borrow the visual language of a competition.

## Name the subject

A verdict names the row it is about (`tlinalg-pool 8.7x faster than lapack-mkl`), and rows print
under the measured library's name, not the dependency it wraps. A reader should not need the
campaign descriptor to learn which of two rows a sentence refers to.
