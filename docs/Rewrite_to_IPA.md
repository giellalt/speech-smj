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
| tools/tts/modes/smj-analyser.mode
```

This will output text of the form:

```
"<Skåvlån>"
	"skåvllå" N Sem/Edu_Org Sg Ine <W:0.0> @ADVL> #1->7 "skɔwlɔn"phon
: 
"<hæhttuji>"
	"hæhttut" V IV Ind Prs Pl3 <W:0.0> @FAUX #2->0 "hæhtːʉji"phon
	"hæhttut" V IV Ind Prt Sg2 <W:0.0> @FAUX #2->0 "hæhtːʉji"phon
: 
"<juohkka>"
	"juohkka" Pron Indef Attr <W:0.0> @>Num #3->4 "jʉuhkːɑ"phon
: 
"<akta>"
	"akta" Num Sg Nom <W:0.0> @SUBJ> #4->7 "ɑktɑ"phon
: 
```

To exract the phonetic transciption, extend the command above as follows:

```sh
echo 'Skåvlån hæhttuji juohkka akta sierra skåvllåbiktasijt adnet. \
Eskilin li tjáhppis båvså, tjáhppis jali bieddjis skirtto ja alek slippsa. \
Næjtsojn li sæmmi skåvllåbiktasa, valla sij máhtti aj vuolppuj tjágŋat.' \
| tools/tts/modes/smj-analyser.mode \
| grep 'phon' | rev | cut -d'"' -f2 | rev | uniq
```

The output is then:

```
skɔwlɔn
hæhtːʉji
jʉuhkːɑ
ɑktɑ
sierːɑ
skɔvlːɔbiktɑsijt
ɑdnət
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

The conversion is done using a rule transducer. Essentially, the algorithm is simply:

```sh
for each main reading; do
    remove lemma
    remove weight (starts with `<W`)
    remove dependency relations (starts with `#`)
    send the rest to the fst, including tags that can be used to guide the IPA conversion
    replace the `Phon` string with the returned IPA string, leaving all tags in place
done
```

That is, the output should be something like this (with huge reservations for the actual IPA!):

```sh
"<Dr.>"
	"dåktår" Area/NO N Sem/Hum Sg Acc "dɔkʰtɔrav"Phon <W:0.0>
		"dr" Area/NO N Sem/Hum ABBR Gram/TAbbr Sg Acc <W:0.0>
```

## Compounds

Example input:

```
"<álggosijddo>"
	"sijddo" N Sem/Obj-surfc Sg Nom "sijddo"Phon <W:0.0> @SUBJ> #7->16
		"álggo" N Sem/Time Cmp/SgNom Cmp "álggo"Phon <W:0.0> #7->16
```

or:

```
"<álggosijddo>"
	"sijddo" N Sem/Obj-surfc Sg Nom "álggo#sijddo"Phon <W:0.0> @SUBJ> #7->16
		"álggo" N Sem/Time Cmp/SgNom Cmp <W:0.0> #7->16
```

**To consider:** We don't know yet what the `Phon` string of a compound will look like: will it be only on the main reading, or split among the sub-readings, as the lemma? It will probably be easiest if we can keep the `Phon` string in one place, but that depends on the tool(s) creating the `Phon` strings.

The most robust approach is to allow for both variants.

## Other comments

If there is no `Phon` string in the input, use the wordform in the cohort as a substitute, and create the `Phon` string from the returned IPA. This should also handle cases of unknown input

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

# Open issues

- punctuation conversion
    - they are presently left as is, but should be converted to symbols for short breaks, intonation modulation etc
- exceptional pronunciation
    - we have not yet decided, but it could either be specified in `src/phonetics/txt2ipa.xfscript` or in the lexc files.
      Using the `txt2ipa` file will work right out of the box, lexc requires some hard thinking
- make use of syntactic parsing
