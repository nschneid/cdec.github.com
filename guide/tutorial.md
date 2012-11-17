---
layout: default
title: cdec system building tutorial
---
<section> This tutorial will guide you through the process of creating a Spanish-English statistical machine translation system with `cdec`. To complete this tutorial successfully, you will need:

 - About 1 hour of time
 - A Linux or MacOS X system with at least 2GB RAM
 - At least least 2GB disk space
 - Python 2.7 or newer

</section>
<section>
### 0. Prerequisites

 - [Compile `cdec`](compiling.html) and run `tests/run-system-tests.pl`
 - Build the `pycdec` Python language extensions to `cdec`
 - Download and compile [SRILM](http://www.speech.sri.com/projects/srilm/download.html) (for estimating the language model)
 - Download and untar [`cdec-spanish-demo.tar.gz`](http://data.cdec-decoder.org/cdec-spanish-demo.tar.gz) (16.7 MB)

</section>
<section>
### 1. Tokenize and lowercase the training, dev, and devtest data
Estimated time: **~2 minutes**
    ~/cdec/corpus/tokenize-anything.sh < training/news-commentary-v7.es-en |
       ~/cdec/corpus/lowercase.pl > nc.lc-tok.es-en

    ~/cdec/corpus/tokenize-anything.sh < dev/2010.es-en |
       ~/cdec/corpus/lowercase.pl > dev.lc-tok.es-en

    ~/cdec/corpus/tokenize-anything.sh < devtest/2011.es-en |
       ~/cdec/corpus/lowercase.pl > devtest.lc-tok.es-en

Read more about the [data format](/documentation/corpus-format.html) used for parallel corpora.

**Exercises:**

 - Tokenization (making punctuation into white-space delimited tokens) and lowercasing are techniques for reducing data sparsity. Try running the tutorial without these.

</section>
<section>
### 2. Filter training corpus sentence lengths
Estimated time: **20 seconds**.

    ~/cdec/corpus/filter-length.pl nc.lc-tok.es-en > training.es-en

This step filters out sentence pairs that are either very long or have a very unusual length ratio (relative to the corpus average). This tends to remove sentence pairs that are either misaligned or will be hard to model.

**Exercises:**

 - Compare `nc.lc-tok.es-en` and `training.es-en` to see what sentence pairs were removed. Were these good translations or not?

</section>
<section>
### 3. Run word bidirectional word alignments
Estimated time: **~10 minutes**
    ~/cdec/training/fast_align -i training.es-en -d -v > training.es-en.fwd_align
    ~/cdec/training/fast_align -i training.es-en -d -v -r > training.es-en.rev_align

You can read more about [word alignment](/concepts/alignment.html) and the [`fast_align` alignment tool](fast_align.html).

**Exercises:**

 - Visualize the outputs of the word aligner with the `~/cdec/utils/atools -c display -i FILE.ALIGNS`. How do the forward and reverse alignments differ?
 - The `-d` option causes the aligner to favor "diagonal" alignments. How do the alignments change without this flag?
 - Most unsupervised aligners work by maximizing the likelihood of the training data. The `-v` option instructs the aliger to favor model parameters that make a small number of very confident decisions, rather than purely trying to maximizing the likelihood. How do the alignments differ with and without this flag?
 - Download another alignment toolkit (a list can be found [here](fast_align.html)) and compare the alignments it produces to the ones produced by `fast_align`.

</section>
<section>
### 4. Symmetrize word alignments
Estimated time: **5 seconds**
    ~/cdec/utils/atools -i training.es-en.fwd_align -j training.es-en.rev_align \
        -c grow-diag-final-and > training.gdfa

**Exercises:**

 - How do the symmetrized alignments (in `training.gdfa`) differ from the forward and reverse alignments?
 - Try out other symmetrization heuristics (e.g., `grow-diag-final`, `union`, and `intersect`) to see how the resulting alignments differ.

</section>
<section>
### 5. Compile the training data
Estimated time: **~1 minute**

*This step assumes your shell is `bash`*.

    export PYTHONPATH=`echo ~/cdec/python/build/lib.*`
    python -m cdec.sa.compile -b training.es-en -a training.gdfa -c extract.ini -o training.sa

This step compiles the parallel training data (in `training.es-en`) into a data structure called a [suffix array](http://en.wikipedia.org/wiki/Suffix_array) that enables very fast lookup of string matches. By representing the training data as a suffix array, it is possible to do a targeted extraction of rules for any input sentence, rather than extracting all rules licensed by the training data.

**Exercises:**

 - Try creating a parallel corpus consisting of two copies (concatenated one after the other) of the training data, but with two different kinds of alignments (e.g, produced by different symmetrization heuristics or by different alignment toolkits). Observe how this changes the downstream grammars and system performance.

</section>
<section>
### 6. Extract grammars for the dev and devtest sets
Estimated time: **15 minutes**

    python -m cdec.sa.extract -c extract.ini -g dev.grammars -j 2 < dev.lc-tok.es-en > dev.lc-tok.es-en.sgm
    python -m cdec.sa.extract -c extract.ini -g devtest.grammars -j 2 <
        devtest.lc-tok.es-en > devtest.lc-tok.es-en.sgm
The `-j 2` option tells the extractor to use 2 processors. This can be adjusted based on your hardward capabilities. The extraction process can be slow, so using more processors if they are available is recommended.

You can read more about the [synchronous context-free grammars](/concepts/scfgs.html) that are being extracted in this step.

The grammars extracted in this section are written to the `dev.grammars` and `devtest.grammars` directory; there is one grammar for each sentence in the dev and devtest sets, containing all rules that could potentially apply to that sentence. The following shows 5 rules extracted for the first sentence of the dev set (*la casa blanca ya ha anunciado que , al recibir el premio , hablará de la guerra afgana .*).

    tail -5 dev.grammars/grammar.0 

    [X] ||| de la guerra [X,1] . ||| the war [X,1] . ||| EgivenFCoherent=1.69896996021 SampleCountF=2.39967370033
    CountEF=0.778151273727 MaxLexFgivenE=1.23959636688 MaxLexEgivenF=0.382706820965 IsSingletonF=0.0 IsSingletonFE
    =0.0 ||| 0-0 1-0 2-1 4-3
    [X] ||| de la guerra [X,1] . ||| from the [X,1] war . ||| EgivenFCoherent=2.39793992043 SampleCountF=2.3996737
    0033 CountEF=0.301030009985 MaxLexFgivenE=0.895682394505 MaxLexEgivenF=1.93757390976 IsSingletonF=0.0 IsSingle
    tonFE=1.0 ||| 0-0 1-1 2-3 4-4
    [X] ||| de la guerra [X,1] . ||| of [X,1] war . ||| EgivenFCoherent=2.39793992043 SampleCountF=2.39967370033 C
    ountEF=0.301030009985 MaxLexFgivenE=1.22095942497 MaxLexEgivenF=0.517585158348 IsSingletonF=0.0 IsSingletonFE=
    1.0 ||| 0-0 2-2 4-3
    [X] ||| de la [X,1] afgana . ||| of afghan [X,1] . ||| EgivenFCoherent=0.301030009985 SampleCountF=0.477121263
    742 CountEF=0.301030009985 MaxLexFgivenE=1.92075574398 MaxLexEgivenF=0.473047554493 IsSingletonF=0.0 IsSinglet
    onFE=1.0 ||| 0-0 3-1 4-3
    [X] ||| de la [X,1] afgana . ||| of the afghan [X,1] . ||| EgivenFCoherent=0.301030009985 SampleCountF=0.47712
    1263742 CountEF=0.301030009985 MaxLexFgivenE=1.31197583675 MaxLexEgivenF=0.796108067036 IsSingletonF=0.0 IsSin
    gletonFE=1.0 ||| 0-0 1-1 3-2 4-4

You can read more about [the cdec grammar format](/documentation/rule-format.html).

**Exercises:**

 - If you use denser alignments (e.g., those produced by the `union` symmetrization heuristic) or sparser alignments (e.g., those produced by `intersect`), how do the number and quality of rules change?
 - Come up with a feature or features whose value can be computed using information just found in the grammar rules (i.e., the source and target RHS yields). Write code to implement it and add it to the grammars. You will need to add a weight for each feature you add to your weights file (discussed below) and then rerun MERT.

</section>
<section>
### 7. Build the target language model
Estimated time: **1 minute**

    ~/cdec/corpus/cut-corpus.pl 2 training.es-en |
       ~/srilm/bin/i686/ngram-count -unk -text - -interpolate -kndiscount -order 3 -lm nc.lm

You can read more about [language models](/concepts/language-models.html).

**Exercises:**

 - Try varying the order of the language model. How does the size of the model file change? How does the translation quality change?

</section>
<section>
### 8. Compile the target language model
Estimated time: **5 seconds**

    ~/cdec/klm/lm/build_binary nc.lm nc.klm

</section>
<section>
### 9. Create a `cdec.ini` configuration file

Create a `cdec.ini` file with the following contents, making sure to substitute the full path to your language model for `$DEMO_ROOT`.
    formalism=scfg
    add_pass_through_rules=true
    density_prune=80
    feature_function=WordPenalty
    feature_function=KLanguageModel $DEMO_ROOT/nc.klm

Create a `weights.init` file (MERT requires that you list the features you want to optimize):
    CountEF 0.1
    EgivenFCoherent -0.1
    Glue 0.01
    IsSingletonF -0.01
    IsSingletonFE -0.01
    LanguageModel 0.1
    LanguageModel_OOV -1
    MaxLexFgivenE -0.1
    MaxLexEgivenF -0.1
    PassThrough -0.1
    SampleCountF -0.1

You can read more about [feature weights](/concepts/weights.html).

**Exercises:**

  - Run the decoder directly from the command line with the command `~/cdec/decoder/cdec -c cdec.ini -w weights.init < dev.lc-tok.es-en.sgm` and observe the output.
  - Run the decoder with the `-k N` option to produce *k*-best lists. Do you see duplicates? Add the `-r` option to filter duplicate entries.

</section>
<section>
### 10. Tune system using development data with MERT
Estimated time: **20-40 minutes**

    ~/cdec/dpmert/dpmert.pl -w weights.init -d dev.lc-tok.es-en.sgm -c cdec.ini -j 2

The `-j 2` option tells MERT to use 2 processors for decoding and optimization. This can be adjusted based on your hardware capabilities.

You can read more about [linear models](/concepts/linear-models.html), [discriminative training](/concepts/training.html), and the [minimum error rate training algorithm](/documentation/mert.html).

**Troubleshooting:**

 - check the `dpmert/logs.1/decoder.sentserver.log.1` file for errors
 - check the `dpmert/logs.1/cdec.1.ER` file for errors

**Exercises:**

 - Repeat the exercises from the previous segment using the `dpmert/weights.final` weights file produced by MERT. How have the translations changed?
 - Use another parameter estimation technique to learn your weights. How do the learned weights vary? How do the translation quality of the dev set and the devtest set vary?

</section>
<section>
### 11. Evaluate test set using trained weights
Estimated time: **5 minutes**

    ~/cdec/dpmert/decode-and-evaluate.pl -c cdec.ini -w dpmert/weights.final -i devtest.lc-tok.es-en.sgm -j 2


This will produce a score report that will look something like the following:
    ### SCORE REPORT #############################################################
      SCRIPT INPUT=devtest.lc-tok.es-en.sgm
            OUTPUT=eval.devtest.lc-tok.es-en.20121014-194914/test.trans
     DECODER INPUT=eval.devtest.lc-tok.es-en.20121014-194914/test.input
        REFERENCES=eval.devtest.lc-tok.es-en.20121014-194914/test.refs
    ------------------------------------------------------------------------------
              BLEU=0.273918
               TER=0.547938
    ##############################################################################
