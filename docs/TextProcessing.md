# Text processing

(AKA preprocessing in TTS parlance)

**Overall goal:**
- use our own text tokenisation and analysis tools to reduce ambiguity and thus increase precision in the produced speech

Basic architecture:
- rule based and lexicon based (fst and vislcg3)
- fall back on general rules only for unknown items

Processing steps:
- tokenisation and morphological analysis
- whitespace markup
- add valency info (helps in later disambiguation)
- disambiguation of MWE's - one or several tokens?
- morphological disambiguation, syntactic & dependency parsing
- [normalisation](normalisation.md)
- [conversion to IPA](Rewrite_to_IPA.md)
- exctraction of IPA strings from full analysis output ⇒ synthesis

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
./configure --enable-tokenisers --enable-phonetic --enable-tts --enable-custom-fsts
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

To extract the phonetic transciption only, extend the command above as follows:

```sh
... \
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

To mostly restore the text as it was (but now normalised/transkribed), instead do like this:

```sh
... \
| egrep '(^:|phon)' | rev | cut -d'"' -f2 | rev | uniq |\
 sed -e 's/^://' | tr -d '\n' | sed -e 's/\\n/\n/'
```

The output then becomes:

```
skɔvlɔːn heætːuji juokːaː ɑktaː siɛrːaː skɔvlːɔːpiktaːsijht ɑtnɛht.
```