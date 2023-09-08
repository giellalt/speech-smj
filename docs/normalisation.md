# New fst tool to expand abbr etc in TTS pipeline

We are going to use the existing grammar checker pipeline infra for text-to-speech (TTS) processing. One of the processing steps is expansion or normalisation of abbreviations, numeric expressions, dates, titles, etc. Presently there is no such tool, but most pieces are available or are easy to build.

The basic idea is a tool with the following interface (inspired by `divvung-cgspell`):

```
divvun-normaliser \
  --normaliser=path/to/normaliserfst \
  --generator=path/to/generator/fst \
  --surface-analyser=path/to/surface/analyser/fst \
  --deep-analyser=path/to/deep/analyser/fst \
  --taggs="list of tags to target"
```

The option `--deep-analyser` is optional, if left out it is assumed that there is no deep layer, and the surface analyser will be used to arrive at the complete analysis, while the orthographic/surface form will be used as the basis for IPA conversion (see more further down).

The list of target tags should be a list of tags found in the analyses of input that needs normalisation, f.ex.:
- `ABBR`
- `Arab`
- `Symbol`

Typically these tags are used as sub-pos tags, such as `N ABBR` (an abbreviation of a noun), or `Num Arab` (arabic numeral, as opposed to a numeral expression written as text).

The normaliser fst is a simple pair-string fst, created from a source file along the following lines:

```
dr.:dåktår # ;
```

The tool takes input in the following form:

```sh
echo "Dr. Mikkelsen"  | hfst-tokenise -g tokeniser-gramcheck-gt-desc.pmhfst | divvun-blanktag analyser-gt-whitespace.hfst | vislcg3 -g valency.cg3 | vislcg3 -g mwe-dis.cg3 | cg-mwesplit 
"<Dr.>"
	"dr" Area/NO N Sem/Hum ABBR Gram/TAbbr Attr <W:0.0>
	"dr" Area/NO N Sem/Hum ABBR Gram/TAbbr Sg Acc <W:0.0>
	"dr" Area/NO N Sem/Hum ABBR Gram/TAbbr Sg Gen <W:0.0>
	"dr" Area/SE N Sem/Hum ABBR Gram/TAbbr Attr <W:0.0>
	"dr" Area/SE N Sem/Hum ABBR Gram/TAbbr Sg Acc <W:0.0>
	"dr" Area/SE N Sem/Hum ABBR Gram/TAbbr Sg Gen <W:0.0>
: 
"<Mikkelsen>"
	"Mikkelsen" Area/NO N Prop Sem/Sur Attr <W:0.0>
	"Mikkelsen" Area/NO N Prop Sem/Sur Sg Nom <W:0.0>
	"Mikkelsen" Area/SE N Prop Sem/Sur Attr <W:0.0>
	"Mikkelsen" Area/SE N Prop Sem/Sur Sg Nom <W:0.0>
:\n
```

The intended output should be something like:

```sh
echo "Dr. Mikkelsen"  | hfst-tokenise -g tokeniser-gramcheck-gt-desc.pmhfst | divvun-blanktag analyser-gt-whitespace.hfst | vislcg3 -g valency.cg3 | vislcg3 -g mwe-dis.cg3 | cg-mwesplit 
"<Dr.>"
	"dåktår" Area/NO N Sem/Hum Attr "dok'tor"Phon <W:0.0>
		"dr" Area/NO N Sem/Hum ABBR Gram/TAbbr Attr <W:0.0>
	"dåktår" Area/NO N Sem/Hum Sg Acc "dok'tor"Phon <W:0.0>
		"dr" Area/NO N Sem/Hum ABBR Gram/TAbbr Sg Acc <W:0.0>
	"dåktår" Area/NO N Sem/Hum Sg Gen "dok'tor"Phon <W:0.0>
		"dr" Area/NO N Sem/Hum ABBR Gram/TAbbr Sg Gen <W:0.0>
	"dåktår" Area/NO N Sem/Hum Attr "dok'tor"Phon <W:0.0>
		"dr" Area/SE N Sem/Hum ABBR Gram/TAbbr Attr <W:0.0>
	"dåktår" Area/NO N Sem/Hum Sg Acc "dok'tor"Phon <W:0.0>
		"dr" Area/SE N Sem/Hum ABBR Gram/TAbbr Sg Acc <W:0.0>
	"dåktår" Area/NO N Sem/Hum Sg Gen "dok'tor"Phon <W:0.0>
		"dr" Area/SE N Sem/Hum ABBR Gram/TAbbr Sg Gen <W:0.0>
: 
...
```

I imagine the processing should go along these lines:

# 1. Create new lemma

1. for each analysis in the cohort, take the lemma from the input analysis, and send it to the normaliser fst
1. create a new analysis with only the normalised lemma; store the original analysis as a CG3 subreading

This should give a cohort similar to this:

```
"<Dr.>"
	"dåktår"
		"dr" Area/NO N Sem/Hum ABBR Gram/TAbbr Sg Acc <W:0.0>
```

By using a very simple fst for this purpose, we gain several things:
- we can use existing transcriptors for the data they cover - as is, more or less
- it is easy to expand
- it is easy to test coverage and lexical fst matches

**NB!** Beware that this step can generate multiple outputs. If so, each need to be given a new entry in the cohort, subject to filtering or removal later on. We need to investigate whether this is an issue, and maybe use additional info to avoid disambiguation later on.

Clues for choosing which one to keep - to be used in the next, generation step:
- the original POS might only fit with one of the substitutions
- if the original POS results in several generated alternatives, include the `+Sem/` tag when generating, or filter against the `+Sem/` tag as a postprocessing step (or just leave both/all, and let further CG disambiguation do the job)

# 2. Generate new surface form

1. Take the original analysis, and remove every prefixed tag (prefixed tags are those of the form `Abcd/xxx`, where `Abcd/` is the tag prefix) + the target tag (`ABBR` in this case):
   `Area/NO N Sem/Hum ABBR Gram/TAbbr Sg Acc` ⇒ `N Sg Acc`
1. Use the new lemma and the new analysis string to generate the corresponding surface form:
   `dåktår N Sg Acc` ⇒ `dåktårav`

Also this process can generate multiple forms. If so, they likely correspond to variant forms, and will be dealt with in the following steps.

This should give a cohort similar to this:

```
"<Dr.>"
	"dåktår" "dåktårav"Phon
		"dr" Area/NO N Sem/Hum ABBR Gram/TAbbr Sg Acc <W:0.0>
```

# 3. Generate deep/underlying form

For each surface form, send it to the surface form analyser. If one use a cascade of two fst's to produce the final analysis, the output of this first analyser is the underlying, lexical representation of the surface string, which is then used as the starting point for IPA conversion.

This should give a cohort similar to this:

```
"<Dr.>"
	"dåktår" "dåktår>av"Phon
		"dr" Area/NO N Sem/Hum ABBR Gram/TAbbr Sg Acc <W:0.0>
```

If a cascade is not used, the orthographic form is used as input for IPA conversion, and this step is skipped.

# 4. Update the analysis

If using a cascade, use the output from 3. above, if not, use the output from 2. above, and for each form:

1. send it through the analyser (surface or deep), to get the full analysis
1. update the analysis string to include all information

The result should be that different variants generated in step 2. above should get additional information that will tell them appart. Lack of differentating information is an error, and must be reported. It requires corrections in the lexc source files.

This should give a cohort similar to this:

```sh
"<Dr.>"
	"dåktår" Area/NO N Sem/Hum Sg Acc "dåktår>av"Phon <W:0.0>
		"dr" Area/NO N Sem/Hum ABBR Gram/TAbbr Sg Acc <W:0.0>
```

# Actual command

This is now mostly implemented, but must be tested thoroughly. Here's the command to run:

```sh
./tools/tts/modes/smj-normaliser.mode < textfile.txt  | \
  ../giella-core/scripts/vislcg-convert.py -t phon -1 | \
  cut -f1-2    | # This only to get rid of some commented out information
  grep -v '^:' | # that can be used for debugging
  grep -v '^$'
```
