---
layout: default
img: mayhem
img_link: http://en.wikipedia.org/wiki/Text_segmentation
caption: Segmentation is harder than it seems.
title: Homework 1 | Segmentation
active_tab: homework
---

Word Segmentation <span class="text-muted">Homework 1</span>
=============================================================

<p class="text-muted">Due on Tuesday, September 23, 2014</p>

Word segmentation is the task of restoring missing word
boundaries. For example, in some cases word boundaries
are lost as in web page URLs like _choosespain.com_ which
could be either _chooses pain_ or _choose spain_ and you 
might visit such an URL looking for one or the other.

This homework is on Chinese word segmentation, a language
in which word boundaries are not usually provided. For
instance here is an example Chinese sentence without word
boundaries:

    北京大学生比赛

This can be segmented a few different ways and one segmentation
leads to a particular meaning (indicated by the English translation below):

    北京 大学生 比赛
    Beijing student competition

A different segmentation leads to a different meaning (and translation):

    北京大学 生 比赛
    Peking University Health Competition

We will be using _training data_ collected from Chinese
sentences that have been segmented by human experts.
We will run the word segmentation program that you
will write for this homework on _test data_ that will
be automatically evaluated against a reference
segmentation.

Getting Started
---------------

You must have git and python (2.7) on your system to run the assignments.
Once you've confirmed this, run this command:

    git clone https://github.com/anoopsarkar/nlp-class-hw.git

In the `segmenter` directory you will find a python program called
`default.py`, which contains a complete but very simple segmentation algorithm.
It simply inserts word boundaries between each Chinese character in the
input. It is a terrible segmenter but it does read the input and produce
a valid output that can be scored.

You can see how well `default.py` does by running the following:

    python default.py | python score-segments.py

Alternatively, you can run:

    python default.py > output
    python score-segments.py -t output

The score reported is [F-measure](http://en.wikipedia.org/wiki/F1_score) which combines 
[precision and recall](http://en.wikipedia.org/wiki/Precision_and_recall) into a single score.

The Challenge
-------------

Your task is to _improve the F-measure as much as possible_. To help you do
this the `data` directory in `segmenter` contains two files:

    count_1w.txt : unigram counts of Chinese words
    count_2w.txt : bigram counts of Chinese word pairs

### The Baseline

A simple baseline uses a unigram language model over Chinese words.
The input is a sequence of Chinese characters (without word
boundaries): $$c_0, \ldots, c_n$$.

Let us define a word as a sequence of characters: $$w_i^j$$ is
a word that spans from character $$i$$ to character $$j$$. So
one possible word sequence is $$w_0^3 w_4^{10} w_{11}^n$$. We
can score this sequence using unigram probabilities.

<p>$$\arg\max_{w_0^i, w_{i+1}^j, \ldots, w_{n-k}^n} P_w(w_0^i) \times P_w(w_{i+1}^j) \times \ldots \times P_w(w_{n-k}^n)$$</p>

The unigram probability $$P_w$$ can be constructed using the data
in `count_1w.txt`. The model is simple, an unigram model, but the
search is over all possible ways to form word sequences for the
input sequence of characters. The argmax over all such sequences
will give you the baseline system. The $$\arg\max$$ above can be computed
using the following recursive search over $$segment(c_0, \ldots, c_n)$$:

<p>$$\begin{eqnarray}
segment(c_i, \ldots, c_j) &=& \arg\max_{\forall k <= L} P_w(w_i^k) \times segment(c_{k+1}, \ldots, c_j) \\
segment(\emptyset) &=& 1.0
\end{eqnarray}$$</p>

where $$L = min(maxlen, j)$$ in order to avoid considering segmentations
of very long words which are going to be very unlikely.
$$segment(\emptyset)$$ is the base case of the recursion: an input
of length zero, which results in a segmentation of length zero with
probability $$1.0$$.

One can [memoize](http://en.wikipedia.org/wiki/Memoization) $$segment$$ in order
to avoid the slow exploration of the exponentially many segmentations.
An alternative is to do this iteratively. The following pseudo-code illustrates
how to find the argmax iteratively.

#### Algorithm: Iterative segmenter

---
**## Data Structures ##**

`input`
: the input sequence of characters

`chart`
: the dynamic programming table to store the argmax for every prefix of `input`
: indexed by character position in `input`

`Entry`
: each entry in the `chart` has four components: Entry(`word`, `start-position`, `log-probability`, `back-pointer`)
: the `back-pointer` in each `entry` links it to a previous entry that it extends

`heap`
: a list or priority queue containing the entries to be expanded, sorted on `start-position` or `log-probability`
{: .dl-horizontal}

---
**## Initialize the `heap` ##**

* for each `word` that matches `input` at position 0    
    * insert Entry(`word`, 0, $$\log P_w$$(`word`), $$\emptyset$$) into `heap`
{: .list-unstyled}

**## Iteratively fill in `chart[i]` for all `i` ##**

* while `heap` is nonempty:
    * `entry` = top entry in the `heap`
    * get the `endindex` based on the length of the word in `entry`
    * if `chart`[`endindex`] has a previous entry, `preventry`
        * if `entry` has a higher probability than `preventry`:
            * `chart`[`endindex`] = `entry`
        * if `entry` has a lower or equal probability than `preventry`:
            * continue  **## we have already found a good segmentation until `endindex` ##**
    * else 
        * `chart`[`endindex`] = `entry`
    * for each `newword` that matches `input` starting at position `endindex`+1
        * `newentry` = Entry(`newword`, `endindex`+1, `entry`.`log-probability` + $$\log P_w$$(`newword`), `entry`)
        * if `newentry` does not exist in `heap`:
            * insert `newentry` into `heap`
{: .list-unstyled}

**## Get the best segmentation ##**

* `finalindex` is the length of `input`
* `finalentry` = `chart`[`finalindex`] 
* The best segmentation starts from `finalentry` and follows the `back-pointer` recursively until the first word
{: .list-unstyled}
---

It might help to examine [an example
run](https://gist.github.com/anoopsarkar/da67c6566a7268bb53b7) of
the above pseudo-code on a particular input. To keep the example
short, the segmenter in the example assumes that unknown words can
only be on length one. You will get a better F-score if you allow
unknown words of arbitrary length (with the appropriate smoothed
probability score).

### Your Task

Developing a segmenter using the above pseudo-code that uses unigram probabilities is
good enough to get close to the baseline system. But getting closer to the oracle
score will be a more interesting challenge. To get full credit you
**must** experiment with at least one additional model of your
choice and document your work. Here are some ideas:

* Use the bigram model to score word segmentation candidates.
* Do better _smoothing_ of the unigram and bigram probability models.

But the sky's the limit! You are welcome to design your own model, as long 
as you follow the ground rules:

Ground Rules
------------

* Each group should submit using one person as the designated uploader.
* You must turn in three things:
  1. A segmentation of the entire dataset which is in `segmenter/data/input` uploaded to the [leaderboard submission site](http://sfu-nlp-class.appspot.com) according to [the Homework 0 instructions](hw0.html). You can upload new output as often
     as you like, up until the assignment deadline. Your score on the leaderboard is the score on the development data set which shown to you immediately after you upload your output file. The **Submit** button in unavailable until after the homework deadline has passed, and when pressed it will show you the test data set score.
The output will be evaluated using a secret metric, but the `score-segments.py` program will give you a good
     idea of how well you're doing.
  1. Your code. Each group should assign one member to upload the source code to [Coursys](https://courses.cs.sfu.ca) as the submission for Homework 1. The code should be self-contained, self-documenting, and easy to use. It should use the same input and output assumptions of `default.py`.
  1. A clear, mathematical description of your algorithm and its motivation
     written in scientific style. This needn't be long, but it should be
     clear enough that one of your fellow students could re-implement it 
     exactly. Include the file for this writeup as part of the tarball or zip file you
     will upload to [Coursys](https://courses.cs.sfu.ca).
* You cannot use data or code resources outside of what is provided
to you. You can use NLTK but not the NLTK tokenizer class. 
* For the written description of your algorithm, you can use plain ASCII but
for math equations it is better to use either
[latex](http://www.latex-project.org/) or
[kramdown](https://github.com/gettalong/kramdown).  Do __not__ use
any proprietary or binary file formats such as Microsoft Word.

If you have any questions or you're confused about anything, just ask.

