# Conversion to IPA

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

```sh
"<álggosijddo>"
        "sijddo" N Sem/Obj-surfc Sg Nom "sijddo"Phon <W:0.0> @SUBJ> #7->16
                "álggo" N Sem/Time Cmp/SgNom Cmp "álggo"Phon <W:0.0> #7->16

```

or:

```sh
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

We need a very simple tool that will take a VISLCG3 stream, and for each cohort only output the `Phon` strings  (possibly with the plain-text input as comments, for debugging).

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
