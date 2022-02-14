# Møte om IPA-konvertering 11.2.2022

Flammie og Sjur.

# Problem som skal løysast

- ord der ulike delar av ordet skal konverterast med ulike fst-ar

Kan brytast ned til desse stega:
- identifisera kvar del
- identifisera rett fst for delen
- konvertera kvar del
- setja saman resultatet

Idear:
- dela opp i CG-underlesingar
- ein fst pr CG-underlesing

# Ulike døme på problemord

Døme: `4-hestak`

```
echo 4-hestak \
| hfst-tokenise -g tools/tokenisers/tokeniser-tts-cggt-desc.pmhfst 
"<4-hestak>"
        "4-hestak" ?
:\n
```

Korleis skal vi dela opp? Ved bindestrek?

```
echo 4 \
| hfst-tokenise -g tools/tokenisers/tokeniser-tts-cggt-desc.pmhfst 
"<4>"
        "4" A Arab Ord Attr CLBfinal "4"MIDTAPE <W:0.0>
        "4" Num Arab Sg Ela Attr "4"MIDTAPE <W:0.0>
        "4" Num Arab Sg Gen "4>"MIDTAPE <W:0.0>
        "4" Num Arab Sg Ill Attr "4"MIDTAPE <W:0.0>
        "4" Num Arab Sg Ine Attr "4"MIDTAPE <W:0.0>
        "4" Num Arab Sg Nom "4>"MIDTAPE <W:0.0>
        "4" Num Sem/ID "4"MIDTAPE <W:0.0>
:\n
echo hestak \
| hfst-tokenise -g tools/tokenisers/tokeniser-tts-cggt-desc.pmhfst 
"<hestak>"
        "hestak" A Attr "hestag9>"MIDTAPE <W:0.0>
        "hestak" A Sg Nom "hestag9>"MIDTAPE <W:0.0>
:\n
```

Hint om delingspunkt:

- bindestrek (fungerer ved ukjente ord med bindestrek)
- morfemgrenser `# » >` (krev analyse)

Eit anna døme: `Vuodnabat-Mikál`

```
echo Vuodnabat- \
| hfst-tokenise -g tools/tokenisers/tokeniser-tts-cggt-desc.pmhfst 
"<Vuodnabat->"
        "Vuodnabahta" N Prop Sem/Plc Cmp/Sh Cmp/SplitR Cmp "Vuodnabat>-"MIDTAPE <W:0.0>
:\n
echo Mikál \
| hfst-tokenise -g tools/tokenisers/tokeniser-tts-cggt-desc.pmhfst 
"<Mikál>"
	"Mikál" ?
:\n
```

Liste over alle ukjende ord med bindestrek i testkorpuset, fyrst dei som byrjar
med stor fyrstebokstav, deretter dei som byrjar med liten:

```
Vuodnabat-Mikál
Gámmel-Rápp
Lofot-guolle
Vuollegåt-Ánndaris
Tjierreg-luoktaj
Iell-áhkko
Tjierrek-Mikkil
Tjieŋal-Erik
Tjieŋal-Erik
Pier-Árnna
Sis-Vásján
Sis-vásjága
Davve-vásjága
Sis-Vásjá
Sis-vásjága
Pier-Knuhtso
Davve-Vásján
Sjæggel-Piera
Tjierrek-Ivár
Tjierrek-Mikkil

4-hestak
gietja-gávtse
giesj-gávtse
páhppa-ræsko
tyska-giellaj
sallabiel-mállagijn
væsto-tuvrran
åt-guok
```

Nokre av orda er skrivefeil, som `tyska-giellaj` for `tuska-giellaj`

```
echo tyska-giellaj \
| hfst-tokenise -g tools/tokenisers/tokeniser-tts-cggt-desc.pmhfst 
"<tyska-giellaj>"
	"tyska-giellaj" ?
:\n
echo tuska-giellaj \
| hfst-tokenise -g tools/tokenisers/tokeniser-tts-cggt-desc.pmhfst 
"<tuska-giellaj>"
        "giella" N Sem/Lang_Tool-catch Sg Ill "tusska>Q1-giella>X4j"MIDTAPE <W:0.0>
                "dujskagiella" v2 N Sem/Lang OLang/NOB Cmp/SgGen Cmp/Hyph Cmp "tusska>Q1-giella>X4j"MIDTAPE <W:0.0>
:\n
```

**NB!!** Samansetjingar med `-giella` får doble siste lemma => må rettast.

# Klassifisering av ord

## Ukjente ord

- skrivefeil (norske ord og samiske)
- ukjente samansetjingar - kan delast ved bindestreken, og behandlast kvar for seg
    - bruk backtracking/deling
- ukjente namn (både usamansette og samansette) - samansette blir elt ved bindestrek, sjå førre punkt

## Kjente ord, kompleks ipa-konvertering:

- maskindivudahka = Cmp/Unass
- masj-ijnna = OLang/NOB
- rad-io = ikkje-samisk sistestaving
- audito-rium = ikkje-samisk sistestaving

Meir detaljert:

### maskindivudahka

* `Cmp/Unass` - norsk fyrstedel, samisk andredel

```
echo maskindivudahka \
| hfst-tokenise -g tools/tokenisers/tokeniser-tts-cggt-desc.pmhfst 
"<maskindivudahka>"
        "divudahka" N Sem/Plc Sg Nom "maskin>∑#divudahka>"MIDTAPE <W:0.0>
                "masjijnna" N Sem/Obj-el OLang/NOB Cmp/Unass Cmp "maskin>∑#divudahka>"MIDTAPE <W:0.0>
:\n
```

### Masj-ijnna

* `OLang/NOB`

```
echo masjijnnadivudahka \
| hfst-tokenise -g tools/tokenisers/tokeniser-tts-cggt-desc.pmhfst 
"<masjijnnadivudahka>"
        "divudahka" N Sem/Plc Sg Nom "masjijnna∑#divudahka>"MIDTAPE <W:0.0>
                "masjijnna" N Sem/Obj-el OLang/NOB Cmp/SgNom Cmp "masjijnna∑#divudahka>"MIDTAPE <W:0.0>
:\n
```

### Rad-io

```
echo radio \
| hfst-tokenise -g tools/tokenisers/tokeniser-tts-cggt-desc.pmhfst 
"<radio>"
        "radio" N Sem/Obj-el OLang/NOB Pl Nom "radio>"MIDTAPE <W:0.0>
        "radio" N Sem/Obj-el OLang/NOB Sg Gen "radio>"MIDTAPE <W:0.0>
        "radio" N Sem/Obj-el OLang/NOB Sg Nom "radio>"MIDTAPE <W:0.0>
:\n
```

### Audito-rium

```
echo auditorium \
| hfst-tokenise -g tools/tokenisers/tokeniser-tts-cggt-desc.pmhfst 
"<auditorium>"
        "auditorium" ?
:\n
```

# Oppsummering

- føretrekkja dynamisk analyse
- backtracking ved bindestrek i pmatch/tokenise, slik at vi deler opp ukjende, samansette ord
- separate ipa-fst-ar for ulike underlesingar, konverter pr underlesing
