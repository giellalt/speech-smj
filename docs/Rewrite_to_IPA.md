# Conversion to IPA

We now have a working setup. Below are first instructions for getting started, followed by the original specifications, and a list of open issues.

## Working setup

**Requirements:**

- nightly hfst package from Tino (at least from June 10 or thereabout, June 21 is known to work), see [this](/infra/compiling_HFST3.html#the-simple-installation-you-download-ready-made-programs) for details
- [newest version of](https://github.com/giellalt):
    - giella-core
    - giella-shared
    - lang-smj

**Configuration & building:**

Run the following commands:

```sh
cd lang-smj
./autogen.sh
./configure --enable-tokenisers --enable-phonetic --enable-tts
cd tools/tts/
make dev # creates all modes/ files
cd ../../
make -j6 # takes a while, ~12 minutes on a fast MacBook Pro
```

**Converting text:**

```sh
echo 'Skåvlån hæhttuji juohkka akta sierra skåvllåbiktasijt adnet. \
Eskilin li tjáhppis båvså, tjáhppis jali bieddjis skirtto ja alek slippsa. \
Næjtsojn li sæmmi skåvllåbiktasa, valla sij máhtti aj vuolppuj tjágŋat.' \
| tools/tts/modes/smj-tts-txt2ipa.mode
```

This will output text of the form:

```
"<Skåvlån>"
	"skåvllå" N Sem/Edu_Org Sg Ine "skåvllå>Q1n"MIDTAPE <W:0.0> @ADVL> #1->7 "skɔvlɔːn"phon
: 
"<hæhttuji>"
	"hæhttuji" ? @X #2->0 "heætːuji"phon
: 
"<juohkka>"
	"juohkka" Pron Indef Attr "juohkka>"MIDTAPE <W:0.0> @>Num #3->4 "juokːaː"phon
: 
"<akta>"
	"akta" Num Sg Nom "akta>"MIDTAPE <W:0.0> @SUBJ> #4->7 "ɑktaː"phon
: 
```

To exract the phonetic transciption, extend the command above as follows:

```sh
echo 'Skåvlån hæhttuji juohkka akta sierra skåvllåbiktasijt adnet. \
Eskilin li tjáhppis båvså, tjáhppis jali bieddjis skirtto ja alek slippsa. \
Næjtsojn li sæmmi skåvllåbiktasa, valla sij máhtti aj vuolppuj tjágŋat.' \
| tools/tts/modes/smj-tts-txt2ipa.mode \
| grep 'phon' | rev | cut -d'"' -f2 | rev | uniq
```

The output is then:

```
skɔvlɔːn
heætːuji
juokːaː
ɑktaː
siɛrːaː
skɔvlːɔːpiktaːsijht
ɑtnɛht
.
```

The idea is that the output should be ready to be fed to the TTS engine, to generate the actual
voice signal.

# Specification

Input is a VISLCG3 cohort usually containing a `Phon` element (string with `Phon` attached at the end):

```sh
"<Dr.>"
	"dåktår" Area/NO N Sem/Hum Sg Acc "dåktår>av"Phon <W:0.0>
		"dr" Area/NO N Sem/Hum ABBR Gram/TAbbr Sg Acc <W:0.0>
```

The conversion is done using one or more rule transducers. That is, the output should be something like this (with huge reservations for the actual IPA!):

```sh
"<Dr.>"
	"dåktår" Area/NO N Sem/Hum Sg Acc "dɔkʰtɔrav"Phon <W:0.0>
		"dr" Area/NO N Sem/Hum ABBR Gram/TAbbr Sg Acc <W:0.0>
```

## Compounds

```
"<álggosijddo>"
	"sijddo" N Sem/Obj-surfc Sg Nom "álggo#sijddo"Phon <W:0.0> @SUBJ> #7->16
		"álggo" N Sem/Time Cmp/SgNom Cmp "álggo#sijddo"Phon <W:0.0> #7->16
```

## Fake three tape fst — abstract phonemic representation

There is now support for building TTS specific analysers that do the analysis in two steps:

1. step: 
```sh
echo åludallabehtit | hfst-lookup -q src/analyser-tts-gt-input.hfstol 
åludallabehtit	oallo»X7dalla>behti>t	0,000000
```
2. step:
```sh
echo 'oallo»X7dalla>behti>t' | hfst-lookup -q src/analyser-tts-gt-output.hfstol 
oallo»X7dalla>behti>t	oallot+V+TV+Der/dalla+V+Ind+Prs+Pl2+Err/Orth	0,000000
```

And `hfst-pmatch` and `hfst-tokenise` have been extended such that given the above input, the output should be:

```sh
echo åludallabehtit | hfst-tokenise -g tools/tokenisers/tokeniser-tts-cggt-desc.pmhfst 
"<åludallabehtit>"
	"oallot" V TV Der/dalla V Ind Prs Pl2 Err/Orth "oallo»X7dalla>behti>t"MIDTAPE <W:0.0>
:\n
```

Using this, we have access to underlying abstractions over phonemes, and more detailed consonant gradation information, which should help us produce more accurate and better speech synthesis.

To use this setup, please configure as follows:

```sh
./configure --enable-tokenisers --enable-phonetic --enable-tts --enable-custom-fsts
```

**MISSING!** MIDTAPE is always printed, also in cases where the MIDTAPE is identical to the surface string. It should not, since that will interfere with `phon` data from other steps.

## Input data priority and IPA conversion 

The processing goes as follows:
- `hfst-tokenise` gives a MIDTAPE representation if different from surface form
- normalisation turns abbreviations into words, and store them in `phon` strings
- in IPA conversion, MIDTAPE takes presedence over `phon` strings
- in IPA conversion, if neither MIDTAPE nor `phon` string is present, use word form as starting point (also handles cases of no analysis)
- use different conversion fst's for MIDTAPE and `phon` strings, and possibly also for different OLang tags

Given that we retain the VISLCG3 cohort stream even after the IPA conversion, it is possible to do further disambiguation afterwords, if needed or wanted.

# Outputting IPA

We need a very simple tool that will take a VISLCG3 stream, and for each cohort only output the `Phon` strings  (possibly with the plain-text wordform as comment, for debugging).

Given this input:

```sh
"<Dr.>"
	"dåktår" Area/NO N Sem/Hum Sg Acc "dɔkʰtɔrav"Phon <W:0.0>
		"dr" Area/NO N Sem/Hum ABBR Gram/TAbbr Sg Acc <W:0.0>
```

return this (or something similar, must be compatible with the synthesiser engines we want to use):

```sh
dɔkʰtɔrav # Dr.

```

This string is then fed to the synthesiser.

## Ambiguity handling

In cases where there is still ambiguity left in the CG stream, proceed as follows:

- if the resulting IPA strings are identical, give a warning to `stderr`, and proceed
- if the resulting IPA strings are different AND in debug/verbose mode, print the amboiguous strings to `stderr`, and stop
- if the resulting IPA strings are different AND NOT in debug/verbose mode AND they have different weights: pick the one with the lowest weight, print a warning on `stderr`, and proceed
- if the resulting IPA strings are different AND NOT in debug/verbose mode AND they have identical weights: pick the first one, print a warning on `stderr`, and proceed

# Open issues

- **punctuation conversion:** they are presently left as is, but should be converted to symbols for various breaks, intonation modulation etc
- **exceptional pronunciation:** we have not yet decided, but it could either be specified in `src/phonetics/txt2ipa.xfscript` or in the lexc files.
      Using the `txt2ipa` file will work right out of the box, lexc requires some hard thinking
- make use of syntactic parsing to govern prosody
