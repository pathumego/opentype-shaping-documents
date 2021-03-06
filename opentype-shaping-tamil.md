# Tamil shaping in OpenType #

This document details the shaping procedure needed to display text
runs in the Tamil script.


**Table of Contents**

  - [General information](#general-information)
  - [Terminology](#terminology)
  - [Glyph classification](#glyph-classification)
      - [Shaping classes and subclasses](#shaping-classes-and-subclasses)
      - [Tamil character tables](#tamil-character-tables)
  - [The `<tml2>` shaping model](#the-tml2-shaping-model)
      - [1: Identifying syllables and other sequences](#1-identifying-syllables-and-other-sequences)
      - [2: Initial reordering](#2-initial-reordering)
      - [3: Applying the basic substitution features from GSUB](#3-applying-the-basic-substitution-features-from-gsub)
      - [4: Final reordering](#4-final-reordering)
      - [5: Applying all remaining substitution features from GSUB](#5-applying-all-remaining-substitution-features-from-gsub)
      - [6: Applying remaining positioning features from GPOS](#6-applying-remaining-positioning-features-from-gpos)
  - [The `<taml>` shaping model](#the-taml-shaping-model)
      - [Distinctions from `<tml2>`](#distinctions-from-tml2)
      - [Advice for handling fonts with `<taml>` features only](#advice-for-handling-fonts-with-taml-features-only)
      - [Advice for handling text runs composed in `<taml>` format](#advice-for-handling-text-runs-composed-in-taml-format)


## General information ##

The Tamil script belongs to the Indic family, and follows
the same general patterns as the other Indic scripts. More
specifically, it belongs to the South Indic subgroup.

The Tamil script is used to write multiple languages, most commonly
Tamil, Irula, and Saurashtra. In addition, Sanskrit may be written
in Tamil, so Tamil script runs may include glyphs from the Vedic
Extension block of Unicode. 

There are two extant Tamil script tags defined in OpenType, `<taml>`
and `<tml2>`. The older script tag, `<taml>`, was deprecated in 2005.
Therefore, new fonts should be engineered to work with the `<tml2>`
shaping model. However, if a font is encountered that supports only
`<taml>`, the shaping engine should deal with it gracefully.

## Terminology ##

OpenType shaping uses a standard set of terms for Indic scripts.  The
terms used colloquially in any particular language may vary, however,
potentially causing confusion.

**Matra** is the standard term for a dependent vowel sign. 

**Halant** and **Virama** are both standard terms for the above-base "vowel-killer"
sign. Unicode documents use the term "virama" most frequently, while
OpenType documents use the term "halant" most frequently. In the Tamil
language, this sign is known as _pulli_.

**Chandrabindu** (or simply **Bindu**) is the standard term for the diacritical mark
indicating that the preceding vowel should be nasalized. Tamil does
not include a "chandrabindu" character, but the term is still found in
multiple places in OpenType shaping documents.

The term **base consonant** is also critical to Indic shaping. The
base consonant of a syllable is the consonant that carries the
syllable's vowel sound, either the inherent vowel (for an unmarked
base consonant) or a dependent vowel (with the addition of a matra).

A syllable's base consonant is generally rendered in its full form
(although it may form ligatures), while other consonants in the
syllable frequently take on secondary forms. Different GSUB
substitutions may apply to a script's **pre-base** and **post-base**
consonants. Some of these substitutions create **above-base** or
**below-base** forms. The **Reph** form of the consonant "Ra" is an
example.

Where possible, using the standard terminology is preferred, as the
use of a language-specific term necessitates choosing one language
over all of the others that share a common script.

## Glyph classification ##

Shaping Tamil text depends on the shaping engine correctly
classifying each glyph in the run. As with most other scripts, the
classifications must distinguish between consonants, vowels
(independent and dependent), numerals, punctuation, and various types
of diacritical mark.

For most codepoints, the `General Category` property defined in the Unicode
standard is correct, but it is not sufficient to fully capture the
expected shaping behavior (such as glyph reordering). Therefore,
Tamil glyphs must additionally be classified by how they are treated
when shaping a run of text.

### Shaping classes and subclasses ###

The shaping classes listed in the tables that follow are defined so
that they capture the positioning rules used by Indic scripts. 

For most codepoints, the _Shaping class_ is synonymous with the `Indic
Syllabic Category` defined in Unicode. However, there are some
distinctions, where the defined category does not fully capture the
behavior of the character in the shaping process.

Several of the diacritic and syllable-modifying marks behave according
to their own rules and, thus, have a special class. These include
`BINDU`, `VISARGA`, `AVAGRAHA`, `NUKTA`, and `VIRAMA`. Some
less-common marks behave according to rules that are similar to these
common marks, and are therefore classified with the corresponding
common mark. The Vedic Extensions also include a `CANTILLATION`
class for tone marks.

Letters generally fall into the classes `CONSONANT`,
`VOWEL_INDEPENDENT`, and `VOWEL_DEPENDENT`. These classes help the
shaping engine parse and identify key positions in a syllable. For
example, Unicode categorizes dependent vowels as `Mark [Mn]`, but the
shaping engine must be able to distinguish between dependent vowels
and diacritical marks (which are categorized as `Mark [Mn]`).

Tamil includes one special class of letter, `MODIFYING_LETTER`, which
is used only for "Visarga" (`U+0B83`). This denotes the character's
usage in the Tamil language, which treats "Visarga" differently than
other Indic scripts. In older Tamil texts, "Visarga" may indicate the
presence of a silent letter; in recent Tamil texts, "Visarga" is used
to modify the following letter in order to denote a foreign phoneme,
such as "f". In shaping, "Visarga" should match tests for letters, but
it is neither a consonant nor a vowel.

Other characters, such as symbols and miscellaneous letters (for
example, letter-like symbols that only occur as standalone entities
and do not occur within syllables), need no special attention from the
shaping engine, so they are not assigned a shaping class.

Numbers are classified as `NUMBER`, even though they evoke no special
behavior from the Indic shaping rules, because there are OpenType features that
might affect how the respective glyphs are drawn, such as `tnum`,
which specifies the usage of tabular-width numerals, and `sups`, which
replaces the default glyphs with superscript variants.

Marks and dependent vowels are further labeled with a mark-placement
subclass, which indicates where the glyph will be placed with respect
to the base character to which it is attached. The actual position of
the glyphs is determined by the lookups found in the font's GPOS
table, however, the shaping rules for Indic scripts require that the
shaping engine be able to identify marks by their general
position. 

For example, left-side dependent vowels (matras), classified
with `LEFT_POSITION`, must frequently be reordered, with the final
position determined by whether or not other letters in the syllable
have formed ligatures or combined into conjunct forms. Therefore, the
`LEFT_POSITION` subclass of the character must be tracked throughout
the shaping process.

For most mark and dependent-vowel codepoints, the _Mark-placement
subclass_ is synonymous with the `Indic Positional Category` defined
in Unicode. However, there are some distinctions, where the defined
category does not fully capture the behavior of the character in the
shaping process. 

### Tamil character tables ###

Separate character tables are provided for the Tamil, Grantha, and Vedic
Extensions block as well as for other miscellaneous characters that
are used in `<tml2>` text runs:

  - [Tamil character table](character-tables/character-tables-tamil.md#tamil-character-table)
  - [Grantha marks character table](character-tables/character-tables-tamil.md#grantha-marks-character-table)
  - [Vedic Extensions character table](character-tables/character-tables-tamil.md#vedic-extensions-character-table)
  - [Miscellaneous character table](character-tables/character-tables-tamil.md#miscellaneous-character-table)

The tables list each codepoint along with its Unicode general
category, its shaping class, and its mark-placement subclass. The
codepoint's Unicode name and an example glyph are also provided.

For example:

| Codepoint | Unicode category | Shaping class     | Mark-placement subclass    | Glyph                        |
|:----------|:-----------------|:------------------|:---------------------------|:-----------------------------|
|`U+0B82`   | Mark [Mn]        | BINDU             | TOP_POSITION               | &#x0B82; Anusvara            |
| | | | |
|`U+0B95`   | Letter           | CONSONANT         | _null_                     | &#x0B95; Ka                  |



Codepoints with no assigned meaning are designated as _unassigned_ in
the _Unicode category_ column.

Assigned codepoints with a _null_ in the _Shaping class_
column evoke no special behavior from the shaping engine. 

The _Mark-placement subclass_ column indicates mark-placement
positioning for codepoints in the _Mark_ category. Assigned, non-mark
codepoints have a_null_ in this column and evoke no special
mark-placement behavior. Marks tagged with [Mn] in the _Unicode
category_ column are categorized as non-spacing; marks tagged with
[Mc] are categorized as spacing-combining.

Some codepoints in the tables use a _Shaping class_ that
differs from the codepoint's Unicode _General Category_. The _Shaping
class_ takes precedence during OpenType shaping, as it captures more
specific, script-aware behavior.

In addition to the marks in the Tamil Unicode block, Tamil text can
also include several diacritical marks from the Grantha Unicode block,
such as Grantha Candrabindu (`U+11301`), Grantha Visarga (`U+11303`),
and Grantha Nukta (`U+1133C`).

Other important characters that may be encountered when shaping runs
of Tamil text include the dotted-circle placeholder (`U+25CC`), the
zero-width joiner (`U+200D`) and zero-width non-joiner (`U+200C`), and
the no-break space (`U+00A0`).

The dotted-circle placeholder is frequently used when displaying a
dependent vowel (matra) or a combining mark in isolation. Real-world
text syllables may also use other characters, such as hyphens or dashes,
in a similar placeholder fashion; shaping engines should cope with
this situation gracefully.

The zero-width joiner is primarily used to prevent the formation of a conjunct
from a "_Consonant_,Halant,_Consonant_" sequence. The sequence
"_Consonant_,Halant,ZWJ,_Consonant_" blocks the formation of a
conjunct between the two consonants. 

Note, however, that the "_Consonant_,Halant" subsequence in the above
example may still trigger a half-forms feature. To prevent the
application of the half-forms feature in addition to preventing the
conjunct, the zero-width non-joiner must be used instead. The sequence
"_Consonant_,Halant,ZWNJ,_Consonant_" should produce the first
consonant in its standard form, followed by an explicit "Halant".

A secondary usage of the zero-width joiner is to prevent the formation of
"Reph". An initial "Ra,Halant,ZWJ" sequence should not produce a "Reph",
where an initial "Ra,Halant" sequence without the zero-width joiner
otherwise would.

The no-break space is primarily used to display those codepoints that
are defined as non-spacing (marks, dependent vowels (matras),
below-base consonant forms, and post-base consonant forms) in an
isolated context, as an alternative to displaying them superimposed on
the dotted-circle placeholder. These sequences will match
"NBSP,ZWJ,Halant,_Consonant_", "NBSP,_mark_", or "NBSP,_matra_".




## The `<tml2>` shaping model ##

Processing a run of `<tml2>` text involves six top-level stages:

1. Identifying syllables and other sequences
2. Initial reordering
3. Applying the basic substitution features from GSUB
4. Final reordering
5. Applying all remaining substitution features from GSUB
6. Applying all remaining positioning features from GPOS


As with other Indic scripts, the initial reordering stage and the
final reordering stage each involve applying a set of several
script-specific rules. The basic substitution features must be applied
to the run in a specific order. The remaining substitution features in
stage five, however, do not have a mandatory order.

Indic scripts follow many of the same shaping patterns, but they
differ in a few critical characteristics that the shaping engine must
track. These include:

  - The position of the base consonant in a syllable.
  
  - The final position of "Reph".
  
  - Whether "Reph" must be requested explicitly or if it is formed by
    a specific, implicit sequence.
	
  - Whether the below-base forms feature is applied only to consonants
    before the base consonant, only to consonants after the base
    consonant, or to both.
	
  - The ordering positions for dependent vowels
    (matras). Specifically, right-side, above-base, and below-base
    matras follow different rules in different scripts. 
	All Indic scripts position left-side matras in the same
    manner, in the ordering position `POS_PREBASE_MATRA`. 

With regard to these common variations, Tamil's specific shaping
characteristics include: 

  - `BASE_POS_LAST` = The base consonant of a syllable is the last
     consonant, not counting any special final-consonant forms.

  - `REPH_POS_AFTER_POST` = "Reph" is ordered after all post-base consonant forms.

  - `REPH_MODE_IMPLICIT` = "Reph" is formed by an initial "Ra,Halant" sequence.

  - `BLWF_MODE_PRE_AND_POST` = The below-forms feature is applied both to
     pre-base consonants and to post-base consonants.

  - `MATRA_POS_TOP` = `POS_AFTER_SUBJOINED` = Above-base matras are
     ordered after subjoined (i.e., below-base) consonant forms. 

  - `MATRA_POS_RIGHT` = `POS_AFTER_POST` = Right-side matras are
     ordered after all post-base consonant forms. 

  - `MATRA_POS_BOTTOM` = `POS_AFTER_POST` = Below-base matras are
     ordered after all post-base consonant forms.

These characteristics determine how the shaping engine must reorder
certain glyphs, how base consonants are determined, and how "Reph"
should be encoded within a run of text.

### 1: Identifying syllables and other sequences ###

A syllable in Tamil consists of a valid orthographic sequence
that may be followed by a "tail" of modifier signs. 

> Note: The Tamil Unicode block enumerates one modifier sign,
> "Anusvara" (`U+0B82`). Tamil text can also include several modifier
> signs from the Grantha Unicode block, such as Grantha Candrabindu
> (`U+11301`), Grantha Visarga (`U+11303`), and Grantha Nukta
> (`U+1133C`).In addition, Sanskrit text written in Tamil 
> may include additional signs from Vedic Extensions block. 
>
> Note: Unlike many other Indic scripts, the Tamil Unicode block
> categorizes "Visarga" (`U+0B83`) as a letter, not as a modifier sign.


Each syllable contains exactly one vowel sound. Valid syllables may
begin with either a consonant or an independent vowel. 

If the syllable begins with a consonant, then the consonant that
provides the vowel sound is referred to as the "base" consonant. If
the syllable begins with an independent vowel, that vowel is the
syllable's only vowel sound and, by definition, there is no "base"
consonant. 

> Note: A consonant that is not accompanied by a dependent vowel (matra) sign
> carries the script's inherent vowel sound. This vowel sound is changed
> by a dependent vowel (matra) sign following the consonant.

Generally speaking, the base consonant is the final consonant of the
syllable and its vowel sound designates the end of the syllable. This
rule is synonymous with the `BASE_POS_LAST` characteristic mentioned
earlier. 

Valid consonant-based syllables may include one or more additional 
consonants that precede the base consonant. Each of these
other, pre-base consonants will be followed by the "Halant" mark, which
indicates that they carry no vowel. They affect pronunciation by
combining with the base consonant (e.g., "_str_", "_pl_") but they
do not add a vowel sound.

Unlike many other Indic scripts, the consonant "Ra" does not receive special
treatment; "Ra,Halant" sequences are not replaced with "Reph".


In addition to valid syllables, stand-alone sequences may occur, such
as when an isolated codepoint is shown in example text.

> Note: Foreign loanwords, when written in the Tamil script, may
> not adhere to the syllable-formation rules described above. In
> particular, it is not uncommon to encounter foreign loanwords that
> contain a word-final suffix of consonants.
>
> Nevertheless, such word-final suffixes will be correctly matched by
> the regular expressions listed below. These loanwords are pronounced
> different, which raises issues for potential readers, but the
> character sequences do not affect the shaping process.



Syllables should be identified by examining the run and matching
glyphs, based on their categorization, using regular expressions. 

The following general-purpose Indic-shaping regular expressions can be
used to match Tamil syllables. The regular expressions utilize the
shaping classifications from the tables above.

> Note: The Tamil block does not include the Anudatta (`U+0952`)
> sign. However, Tamil text may include the Anudatta sign from
> Devanagari as part of the Unicode Script Extensions. 


	C	  Consonant
	V	  Independent vowel
	N	  Nukta
	H	  Halant/Virama
	ZWNJ	  Zero-width non-joiner
	ZWJ	  Zero-width joiner
	M	  Matra (up to one of each type: pre-base, above-base, below-base, or post-base)
	SM	  Syllable modifier signs
	VD	  Vedic signs
	A	  Anudatta
	NBSP	  No-break space
	      
	{ }	  zero or more occurences of the enclosed expression
	[ ]	  optional occurence of the enclosed expression
	<|>	  one of the options separated by the vertical bar
	( )	  one or two occurences of the enclosed expression

A consonant-based syllable will match the expression:
```
{C+[N]+<H+[<ZWNJ|ZWJ>]|<ZWNJ|ZWJ>+H>} + C+[N]+[A] + [< H+[<ZWNJ|ZWJ>] | {M}+[N]+[H]>]+[SM]+[(VD)]
```

A vowel-based syllable will match the expression:
```
[Ra+H]+V+[N]+[<[<ZWJ|ZWNJ>]+H+C|ZWJ+C>]+[{M}+[N]+[H]]+[SM]+[(VD)]
```

A stand-alone sequence (which can only occur at the start of a word) will match the expression:
```
[Ra+H]+NBSP+[N]+[<[<ZWJ|ZWNJ>]+H+C>]+[{M}+[N]+[H]]+[SM]+[(VD)]
```

A sequence that does not match any of these expressions should be
regarded as broken. The shaping engine may make a best-effort attempt
to shape the broken sequence, but making guarantees about the
correctness or appearance of the final result is out of scope for this
document.

After the syllables have been identified, each of the subsequent 
shaping stages occurs on a per-syllable basis.

### 2: Initial reordering ###

The initial reordering stage is used to relocate glyphs from the
phonetic order in which they occur in a run of text to the
orthographic order in which they are presented visually.

> Note: Primarily, this means moving dependent-vowel (matra) glyphs.
>
> These reordering moves are mandatory. The final-reordering stage
> may make additional moves, depending on the text and on the features
> implemented in the active font.

The syllable should be processed by tagging each glyph with its
intended position based on its ordering category. After all glyphs
have been tagged, the entire syllable should be sorted in stable order,
so that glyphs of the same ordering category remain in the same
relative position with respect to each other.

The final sort order of the ordering categories should be:


	POS_RA_TO_BECOME_REPH
	POS_PREBASE_MATRA
	POS_PREBASE_CONSONANT

	POS_BASE_CONSONANT
	POS_AFTER_MAIN

	POS_ABOVEBASE_CONSONANT

	POS_BEFORE_SUBJOINED
	POS_BELOWBASE_CONSONANT
	POS_AFTER_SUBJOINED

	POS_BEFORE_POST
	POS_POSTBASE_CONSONANT
	POS_AFTER_POST

	POS_FINAL_CONSONANT
	POS_SMVD


This sort order enumerates all of the possible final positions to
which a codepoint might be reordered, across all of the Indic
scripts. It includes some ordering categories not utilized in
Tamil. 

The basic positions (left to right) are "Reph" (`POS_RA_TO_BECOME_REPH`), dependent
vowels (matras) and consonants positioned before the base
consonant (`POS_PREBASE_MATRA` and `POS_PREBASE_CONSONANT`), the base
consonant (`POS_BASE_CONSONANT`), above-base consonants
(`POS_ABOVEBASE_CONSONANT`), below-base consonants
(`POS_BELOWBASE_CONSONANT`), consonants positioned after the base consonant
(`POS_POSTBASE_CONSONANT`), syllable-final consonants (`POS_FINAL_CONSONANT`),
and syllable-modifying or Vedic signs (`POS_SMVD`).

In addition, several secondary positions are defined to handle various
reordering rules that deal with relative, rather than absolute,
positioning. `POS_AFTER_MAIN` means that a character must be
positioned immediately after the base consonant. `POS_BEFORE_SUBJOINED`
and `POS_AFTER_SUBJOINED` mean that a character must be positioned
before or after any below-base consonants, respectively. Similarly,
`POS_BEFORE_POST` and `POS_AFTER_POST` mean that a character must be
positioned before or after any post-base consonants, respectively. 

For shaping-engine implementers, the names used for the ordering
categories matter only in that they are unambiguous. 

For a definition of the "base" consonant, refer to step 2.1, which follows.

#### 2.1: Base consonant ####

The first step is to determine the base consonant of the syllable, if
there is one, and tag it as `POS_BASE_CONSONANT`.

The base consonant is defined as the consonant in a consonant-based
syllable that carries the syllable's vowel sound. That vowel sound
will either be provided by the script's inherent vowel (in which case
it is not written with a separate character) or the sound will be designated
by the addition of a dependent-vowel (matra) sign.

Vowel-based syllables, standalone-sequences, and broken text runs will
not have base consonants.

The algorithm for determining the base consonant is

  - If the syllable starts with "Ra,Halant" and the syllable contains
    more than one consonant, exclude the starting "Ra" from the list of
    consonants to be considered. 
  - Starting from the end of the syllable, move backwards until a consonant is found.
      * If the consonant has a below-base or post-base form or is a
        pre-base-reordering "Ra", move to the previous consonant. If
        neither condition is true, stop. 
      * If the consonant is the first consonant, stop.
  - The consonant stopped at will be the base consonant.

Shaping engines may choose any method to identify consonants that have
below-base or post-base forms while executing the above algorithm. For
example, one implementation may choose to maintain a static table of
below-base and post-base consonants to compare again the text
run. Another implementation might examine the active font to see if it
includes a relevant `blwf` or `pstf` lookup in the GSUB table.

> Note: The algorithm is designed to work for all Indic
> scripts. However, Tamil does not utilize pre-base-reordering "Ra".


#### 2.2: Matra decomposition ####

Second, any multi-part dependent vowels (matras) must be decomposed
into their components. Tamil has three multi-part dependent vowels,
"O" (`U+0BCA`), "Oo" (`U+0BCB`), and "Au" (`U+0BCC`).

> "O" (`U+0BCA`) decomposes to "`U+0BC6`,`U+0BBE`"
>
> "Oo" (`U+0BCB`) decomposes to "`U+0BC7`,`U+0BBE`"
> 
> "Au" (`U+0BCC`) decomposes to "`U+0BC6`,`U+0BD7`"


Because this decomposition is a character-level operation, the shaping
engine may choose to perform it earlier, such as during an initial
Unicode-normalization stage. However, all such decompositions must be
completed before the shaping engine begins step three, below.

![Two-part matra decomposition](/images/tamil/tamil-matra-decompose.png)

#### 2.3: Tag matras ####

Third, all left-side dependent-vowel (matra) signs must be tagged to be
moved to the beginning of the syllable, with `POS_PREBASE_MATRA`.

Above-base dependent-vowel (matra) signs must be tagged with `POS_AFTER_SUBJOINED`.

Right-side dependent-vowel (matra) signs must be tagged with `POS_AFTER_POST`.

Below-base dependent-vowel (matra) signs must be tagged with `POS_AFTER_POST`.

#### 2.4: Adjacent marks ####

Fourth, any subsequences of adjacent marks ("Halant"s, "Nukta"s,
syllable modifiers, and Vedic signs) must be reordered so that they
appear in canonical order. 

For `<tml2>` text, the canonical ordering means that any "Nukta"s must
be placed before all other marks. No other marks in the subsequence
should be reordered.

> Note: The Tamil Unicode block does not include a "Nukta"
> codepoint. However, Tamil text may include "Grantha Nukta" (`U+1133C`)
> and other modifier signs from the Grantha Unicode block.
>
> In addition, `<tml2>` text runs in minority languages that
> use the Tamil script may incorporate nukta characters from other
> blocks. Therefore shaping engines must apply the appropriate
> mark-reordering move if a character matching the NUKTA shaping class
> is encountered.


#### 2.5: Pre-base consonants ####

Fifth, consonants that occur before the base consonant must be tagged
with `POS_PREBASE_CONSONANT`.

#### 2.6: Reph ####

Sixth, initial "Ra,Halant" sequences that will become "Reph"s must be tagged with
`POS_RA_TO_BECOME_REPH`.

Tamil does not use "Reph", so this step will
involve no work when shaping `<tml2>` text. It is included here in order
to maintain compatibility with the other Indic scripts.

#### 2.7: Post-base consonants ####

Seventh, any non-base consonants that occur after a dependent vowel
(matra) sign must be tagged with `POS_POSTBASE_CONSONANT`. 

In Tamil, no consonants appear in post-base position, so this step
will not involve any work. It is included here in order
to maintain compatibility with the other Indic scripts.

#### 2.8: Mark tagging ####

Eighth, all marks must be tagged with the same positioning tag as the
closest non-mark character the mark has affinity with, so that they move together
during the sorting step.

For all marks preceding the base consonant, the mark must be tagged
with the same positioning tag as the closest preceding non-mark
consonant.

For all marks occurring after the base consonant, the mark must be
tagged with the same positioning tag as the closest subsequent consonant.

> Note: In this step, joiner and non-joiner characters must also be
> tagged according to the same rules given for marks, even though
> these characters are not categorized as marks in Unicode.


With these steps completed, the syllable can be sorted into the final sort order.

### 3: Applying the basic substitution features from GSUB ###

The basic-substitution stage applies mandatory substitution features
using the rules in the font's GSUB table. In preparation for this
stage, glyph sequences should be tagged for possible application 
of GSUB features.

The order in which these substitutions must be performed is fixed for
all Indic scripts:

	locl
	nukt 
	akhn
	rphf (not used in Tamil) 
	rkrf (not used in Tamil)
	pref 
	blwf 
	abvf 
	half
	pstf 
	vatu (not used in Tamil)
	cjct 
	cfar (not used in Tamil)

#### 3.1 locl ####

The `locl` feature replaces default glyphs with any language-specific
variants, based on examining the language setting of the text run.

> Note: Strictly speaking, the use of localized-form substitutions is
> not part of the shaping process, but of the localization process,
> and could take place at an earlier point while handling the text
> run. However, shaping engines are expected to complete the
> application of the `locl` feature before applying the subsequent
> GSUB substitutions in the following steps.

#### 3.2: nukt ####


The `nukt` feature replaces "_Consonant_,Nukta" sequences with a
precomposed nukta-variant of the consonant glyph. 

> Note: The Tamil Unicode block does not include a "Nukta"
> codepoint. However, Tamil text may include "Grantha Nukta" (`U+1133C`)
> from the Grantha Unicode block.
>
> In addition, `<tml2>` text runs in minority languages that
> use the Tamil script may incorporate nukta characters from other
> blocks. Therefore shaping engines must apply the `nukt` feature if
> it is used in the active font.


#### 3.3: akhn ####

The `akhn` feature replaces one specific sequence with a required ligature. 

  - "Ka,Halant,Ssa" is substituted with the "KSsa" ligature. 
  
These sequences can occur anywhere in a syllable. The "KSsa" 
character has orthographic status equivalent to full
consonants in some languages, and fonts may have `cjct` substitution
rules designed to match them in subsequences. Therefore, this
feature must be applied before all other many-to-one substitutions.

![akhn KSsa formation](/images/tamil/tamil-akhn-kssa.png)


#### 3.4: rphf ####

> This feature is not used in Tamil.

<!--- The `rphf` feature replaces initial "Ra,Halant" sequences with the
"Reph" glyph.

  - An initial "Ra,Halant,ZWJ" sequence, however, must not be tagged for
    the `rphf` substitution.
	

![Reph formation](/images/tamil/tamil-rphf.png) --->
	
#### 3.5 rkrf ####

> This feature is not used in Tamil.

<!--- The `rkrf` feature replaces "_Consonant_,Halant,Ra" sequences with the
"Rakaar"-ligature form of the consonant glyph.

![Rakaar ligation](/images/tamil/tamil-rkrf.png) --->

#### 3.6 pref ####

The `pref` feature replaces pre-base-consonant glyphs with 
any special forms.

> Note: Tamil does not usually incorporate pre-base-consonant forms, but it is
> possible for a font to implement them in order to provide for
> desired typographic variation.


#### 3.7: blwf ####

The `blwf` feature replaces below-base-consonant glyphs with any
special forms. 

> Note: Tamil does not usually incorporate below-base-consonant forms, but it is
> possible for a font to implement them in order to provide for
> desired typographic variation.


Because Tamil incorporates the `BLWF_MODE_PRE_AND_POST` shaping
characteristic, any pre-base consonants and any post-base consonants
may potentially match a `blwf` substitution; therefore, both cases must
be tagged for comparison. Note that this is not necessarily the case in other
Indic scripts that use a different `BLWF_MODE_` shaping
characteristic. 


#### 3.8: abvf ####

The `abvf` feature replaces above-base-consonant glyphs with any
special forms. 

> Note: Tamil does not usually incorporate above-base-consonant forms, but it is
> possible for a font to implement them in order to provide for
> desired typographic variation.


#### 3.9: half ####

The `half` feature replaces "_Consonant_,Halant" sequences before the
base consonant with "half forms" of the consonant glyphs. There are
four exceptions to the default behavior, for which the shaping engine
must test:

  - Initial "Ra,Halant" sequences, which should have been tagged for
    the `rphf` feature earlier, must not be tagged for potential
    `half` substitutions.

  - Non-initial "Ra,Halant" sequences, which should have been tagged
    for the `rkrf` or `blwf` features earlier, must not be tagged for
    potential `half` substitutions.
  
  - A sequence matching "_Consonant_,Halant,ZWJ,_Consonant_" must be
    tagged for potential `half` substitutions, even though the presence of the
    zero-width joiner suppresses the `cjct` feature in a later step.

  - A sequence matching "_Consonant_,Halant,ZWNJ,_Consonant_" must not be
    tagged for potential `half` substitutions.


> Note: Tamil does not usually incorporate half forms, but it is
> possible for a font to implement them in order to provide for
> desired typographic variation.

![half-form feature application](/images/tamil/tamil-half.png)

#### 3.10: pstf ####

The `pstf` feature replaces post-base-consonant glyphs with any
special forms. 

> Note: Tamil does not usually incorporate post-base-consonant forms, but it is
> possible for a font to implement them in order to provide for
> desired typographic variation.


#### 3.11: vatu ####

> This feature is not used in Tamil.

#### 3.12: cjct ####

The `cjct` feature replaces sequences of adjacent consonants with
conjunct ligatures. These sequences must match "_Consonant_,Halant,_Consonant_".

A sequence matching "_Consonant_,Halant,ZWJ,_Consonant_" or
"_Consonant_,Halant,ZWNJ,_Consonant_" must not be tagged to form a conjunct.

The font's GSUB rules might be implemented so that `cjct`
substitutions apply to half-form consonants; therefore, this feature
must be applied after the `half` feature. 


> Note: Tamil does not usually incorporate conjunct forms, but it is
> possible for a font to implement them in order to provide for
> desired typographic variation.


#### 3.13: cfar ####

> This feature is not used in Tamil.


### 4: Final reordering ###

The final reordering stage repositions marks, dependent-vowel (matra)
signs, and "Reph" glyphs to the appropriate location with respect to
the base consonant. Because multiple substitutions may have occurred
during the application of the basic-shaping features in the preceding
stage, these repositioning moves could not be performed during the
initial reordering stage.

Like the initial reordering stage, the steps involved in this stage
occur on a per-syllable basis.

<!--- Check that classifications have not been mangled. If the -->
<!--character is a Halant AND a ligature was formed AND a multiple
substitution was performed, restore the classification to VIRAMA
because it was almost certainly lost in the preceding GSUB stage.
--->

#### 4.1: Base consonant ####

The final reordering stage, like the initial reordering stage, begins
with determining the base consonant of each syllable, following the
same algorithm used in stage 2, step 1.

The codepoint of the underlying base consonant will not change between
the search performed in stage 2, step 1, and the search repeated
here. However, the application of GSUB shaping features in stage 3
means that several ligation and many-to-one substitutions may have
taken place. The final glyph produced by that process may, therefore,
be a conjunct or ligature form — in most cases, such a glyph will not
have an assigned Unicode codepoint.
   
#### 4.2: Pre-base matras ####

Pre-base dependent vowels (matras) that were reordered during the
initial reordering stage must be moved to their final position. This
position is defined as:
   
   - after the last standalone "Halant" glyph that comes after the
     matra's starting position and also comes before the main
     consonant.
   - If a zero-width joiner or a zero-width non-joiner follows this
     last standalone "Halant", the final matra position is moved to
     after the joiner or non-joiner.

This means that the matra will move to the right of all explicit
"consonant,Halant" subsequences, but will stop to the left of the base
consonant, all conjuncts or ligatures that contains the base
consonant, and all half forms.

![Pre-base matra positioning](/images/tamil/tamil-matra-position.png)

#### 4.3: Reph ####

"Reph" must be moved from the beginning of the syllable to its final
position. Because Tamil incorporates the `REPH_POS_BEFORE_POST`
shaping characteristic, this final position is immediately before any
independent post-base consonant forms (meaning the first post-base
consonant that has not formed a ligature with the base consonant).

  - If the syllable does not have any post-base consonants, then the
    final "Reph" position is immediately before the first post-base
    matra, syllable modifier, or Vedic sign.

Tamil does not use "Reph", so this step will involve no work when
processing `<tml2>` text. It is included here in order to maintain
compatibility with the other Indic scripts. 


#### 4.4: Pre-base-reordering consonants ####

Any pre-base-reordering consonants must be moved to immediately before
the base consonant.
  
Tamil does not use pre-base-reordering consonants, so this step will
involve no work when processing `<tml2>` text. It is included here in order
to maintain compatibility with the other Indic scripts.
  
#### 4.5: Initial matras ####

Any left-side dependent vowels (matras) that are at the start of a
word must be tagged for potential substitution by the `init` feature
of GSUB.

Tamil does not use the `init` feature, so this step will involve no
work when processing `<tml2>` text. It is included here in order to
maintain compatibility with the other Indic scripts.
   
### 5: Applying all remaining substitution features from GSUB ###

In this stage, the remaining substitution features from the GSUB table
are applied. The order in which these features are applied is not
canonical; they should be applied in the order in which they appear in
the GSUB table in the font. 

	init (not used in Tamil)
	pres
	abvs
	blws
	psts
	haln

The `init` feature is not used in Tamil.

The `pres` feature replaces pre-base-consonant glyphs with special
presentations forms. This can include consonant conjuncts, half-form
consonants, and stylistic variants of left-side dependent vowels
(matras). 

![pres feature application](/images/tamil/tamil-pres.png)

The `abvs` feature replaces above-base-consonant glyphs with special
presentation forms. This usually includes contextual variants of
above-base marks or contextually appropriate mark-and-base ligatures.

![abvs feature application](/images/tamil/tamil-abvs.png)

The `blws` feature replaces below-base-consonant glyphs with special
presentation forms. This usually includes replacing base consonants that
are adjacent to the below-base marks with contextually appropriate
ligatures.


The `psts` feature replaces post-base-consonant glyphs with special
presentation forms. This usually includes replacing right-side
dependent vowels (matras) with stylistic variants or replacing
post-base-consonant/matra pairs with contextual ligatures. 

![psts feature application](/images/tamil/tamil-psts.png)

The `haln` feature replaces syllable-final "_Consonant_,Halant" pairs with
special presentation forms. This can include stylistic variants of the
consonant where placing the "Halant" mark on its own is
typographically problematic. 

![haln feature application](/images/tamil/tamil-haln.png)

> Note: The `calt` feature, which allows for generalized application
> of contextual alternate substitutions, is usually applied at this
> point. However, `calt` is not mandatory for correct Tamil shaping
> and may be disabled in the application by user preference.

### 6: Applying remaining positioning features from GPOS ###

In this stage, mark positioning, kerning, and other GPOS features are
applied. As with the preceding stage, the order in which these
features are applied is not canonical; they should be applied in the
order in which they appear in the GPOS table in the font.

        dist
        abvm
        blwm

> Note: The `kern` feature is usually applied at this stage, if it is
> present in the font. However, `kern` (like `calt`, above) is not
> mandatory for shaping Tamil text and may be disabled by user preference.

The `dist` feature adjusts the horizontal positioning of
glyphs. Unlike `kern`, adjustments made with `dist` do not require the
application or the user to enable any software _kerning_ features, if
such features are optional. 

The `abvm` feature positions above-base marks for attachment to base
characters. In Tamil, this includes above-base dependent vowels
(matras), diacritical marks, and Vedic signs.

![abvm feature application](/images/tamil/tamil-abvm.png)

The `blwm` feature positions below-base marks for attachment to base
characters. In Tamil, this includes below-base diacritical marks.



## The `<taml>` shaping model ##

The older Tamil script tag, `<taml>`, has been deprecated. However,
shaping engines may still encounter fonts that were built to work with
`<taml>` and some users may still have documents that were written to
take advantage of `<taml>` shaping.

### Distinctions from `<tml2>` ###

The most significant distinction between the shaping models is that the
sequence of "Halant" and consonant glyphs used to trigger shaping
features) was altered when migrating from `<taml>` to
`<tml2>`. 

Specifically, shaping engines were expected to reorder post-base
"Halant,_Consonant_" sequences to "_Consonant_,Halant".

As a result, a font's GSUB substitutions would be written to match
"_Consonant_,Halant" sequences in all pre-base and post-base positions.


The `<taml>` syllable

	Pre-baseC Halant BaseC Halant Post-baseC

would be reordered to

	Pre-baseC Halant BaseC Post-baseC Halant

before features are applied.

In `<tml2>` text, as described above in this document, there is no
such reordering. The correct sequence to match for GSUB substitutions is
"_Consonant_,Halant" for pre-base consonants, but "Halant,_Consonant_"
for post-base consonants.


### Advice for handling fonts with `<taml>` features only ###

Shaping engines may choose to match post-base "_Consonant_,Halant"
sequences in order to apply GSUB substitutions when it is known that
the font in use supports only the `<taml>` shaping model.

### Advice for handling text runs composed in `<taml>` format ###

Shaping engines may choose to match post-base "_Consonant_,Halant"
sequences for GSUB substitutions or to reorder them to
"Halant,_Consonant_" when processing text runs that are tagged with
the `<taml>` script tag and it is known that the font in use supports
only the `<tml2>` shaping model.

Shaping engines may also choose to position left-side matras according
to the `<taml>` ordering scheme; however, doing so might interfere
with matching GSUB or GPOS features.

