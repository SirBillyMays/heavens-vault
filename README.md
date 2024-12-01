This repository is for a [Heavens Vault](https://store.steampowered.com/app/774201/Heavens_Vault/) language decoder.

The page is hosted on Github Pages, under [heavens-vault.hatman.dev.](https://heavens-vault.hatman.dev/)

The font was gathered from the Heavens Vault discord, although no license or original source for the font could be found. The font is available in this repository, but I ask that you please seek out an original source if you wish you reuse the font. If you are the creator of the font and take issue with this project, please let me know.

Heavens Vault and the ancient script are property of Inkle Ltd., and this project is not related to, affiliated with or endorsed by Inkle Ltd. This project is a fan made project and does not intend to intrude upon rights by any party.

The definitions of the words and lexemes (atoms) are sourced from the game files (ExportedProject/Assets/Resources/translation game/data/GameData.json, if you're using AssetRipper.)

ExportedProject/Assets/TextAsset/core.json contains a number of different text events in the game, most notably the "book of truth". This file is in its entirety under the "assets" folder of this project, although with time I intend to cut it down to just contain the core and necessary text elements.


---

Notes:

- The JoinGlyph1 connector is used when connecting base glyphs(?)
- The JoinGlyph1 connector indicates a subject/objerct relationship(?)


Missing from atom list in the base GameData.json are:
“primitive”
“joininglyph1”
“joininglyph2”

The above glyphs are instead added programatically; I have chosen to replicate the behaviour found in "PrototypeWordModel.cs:Validate()".
All the "misssing" atoms have been added to atoms.json and conform to the types listed in AtomModel.cs. (Type:Punctuation)


From ExportedProject/Assets/Scripts/Assembly-CSharp/Translation/AtomModel.cs:
public enum AtomType {
        Type = 0,
        Prefix = 1,
        Modifier = 2,
        Atom = 3,
        Punctuation = 4
}

AtomType is set in the constructor.

from ExportedProject/Assets/Scripts/Assembly-CSharp/Translation/LexDictionary.cs:

static LexDictionary()
{
        _words = new List<PrototypeWordModel>();
        _atoms = new List<AtomModel>();
        atomsDictionary = new Dictionary<string, AtomModel>(StringComparer.OrdinalIgnoreCase);
        PrototypeWordModelsByEquivalentEnglishStrings = new Dictionary<string, PrototypeWordModel>(StringComparer.OrdinalIgnoreCase);
        ignoredWords = new string[4] { "the", "a", "an", "to" };
        BuildLanguage();
}

private static void BuildLanguageAtoms()
{
        JsonData jsonData = TranslationGameUtils.gameData["atoms"];
        for (int i = 0; i < jsonData.Count; i++)
        {
                JsonData atomJsonData = jsonData[i];
                AtomModel atomModel = BuildAtom(atomJsonData);
                atomsDictionary.Add(atomModel.englishWord, atomModel);
                _atoms.Add(atomModel);
        }
        AtomModel atomModel2 = BuildAtom("joinGlyph1", AtomModel.AtomType.Punctuation);
        atomsDictionary.Add(atomModel2.englishWord, atomModel2);
        _atoms.Add(atomModel2);
        AtomModel atomModel3 = BuildAtom("joinGlyph2", AtomModel.AtomType.Punctuation);
        atomsDictionary.Add(atomModel3.englishWord, atomModel3);
        _atoms.Add(atomModel3);
        AtomModel atomModel4 = BuildAtom("primitive", AtomModel.AtomType.Punctuation);
        atomsDictionary.Add(atomModel4.englishWord, atomModel4);
        _atoms.Add(atomModel4);
}

private static AtomModel BuildAtom(JsonData atomJsonData)
{
        string name = (string)atomJsonData["name"];
        AtomModel.AtomType type = EnumX.TryParse<AtomModel.AtomType>((string)atomJsonData["atomType"]);
        return BuildAtom(name, type);
}

private static void BuildLanguageWords()
{
        JsonData jsonData = TranslationGameUtils.gameData["words"];
        for (int i = 0; i < jsonData.Count; i++)
        {
                JsonData jsonData2 = jsonData[i];
                string text = (string)jsonData2["name"];
                List<string> list = new List<string>();
                list.Add(text);
                if (jsonData2.Keys.Contains("equivalences"))
                {
                        for (int j = 0; j < jsonData2["equivalences"].Count; j++)
                        {
                                string item = (string)jsonData2["equivalences"][j];
                                list.Add(item);
                        }
                }
                List<string> list2 = new List<string>();
                for (int k = 0; k < jsonData2["components"].Count; k++)
                {
                        string item2 = (string)jsonData2["components"][k];
                        list2.Add(item2);
                }
                PrototypeWordModel prototypeWordModel = new PrototypeWordModel(text, new List<AtomModel>(), list);
                prototypeWordModel.atomNames = list2;
                RecordWordAndEquivalences(prototypeWordModel);
        }
        for (int l = 0; l < jsonData.Count; l++)
        {
                string name = (string)jsonData[l]["name"];
                PrototypeWordModel prototypeWordModelFromEnglishString = GetPrototypeWordModelFromEnglishString(name);
                CreateWord(prototypeWordModelFromEnglishString, new List<string>());
        }
}

From ExportedProject/Assets/Scripts/Assembly-CSharp/Translation/PrototypeWordModel.cs - comments are mine:
public void Validate()
{
        List<AtomModel> list = new List<AtomModel>();
        AtomModel atom_primitive = LexDictionary.FindAtom("primitive");
        AtomModel atom_of= LexDictionary.FindAtom("of");
        _atoms.RemoveAll((AtomModel x) => x.atomType == AtomModel.AtomType.Punctuation);
        AtomModel atom_iterator = null;
        
        // iterate over _atoms; remove duplicate atoms of type "prefix" and "type"
        for (int i = 0; i < _atoms.Count; i++) {
                if (_atoms[i].atomType == AtomModel.AtomType.Prefix || _atoms[i].atomType == AtomModel.AtomType.Type) {
                        if (atom_iterator == _atoms[i])
                        {
                                continue;
                        }
                        atom_iterator = _atoms[i];
                } else if (_atoms[i].atomType == AtomModel.AtomType.Atom) {
                        atom_iterator = null;
                }

                list.Add(_atoms[i]);
        }
        
        // this handles inserting the joinGlyph1 and joinGlyph2
        bool is_prefix = list[0].atomType == AtomModel.AtomType.Prefix;
        for (int j = 1; j < list.Count; j++) {
                // Firstly, if the atomType is already punctuation, skip.
                if (list[j - 1].atomType == AtomModel.AtomType.Punctuation) {
                        continue;
                }
                
                /*
                Secondly, it checks if the atom before the current one is "type", and the current one is NOT "type"
                Then inserts a "joinGlyph2"
                
                Example that does not get a joinGlyph2 is "robot", aka mineral creature person == Type Type Type
                Example that gets a joinGlyph2 is chain, which is mineral noun join == type (jg2) pref atom
                */
                if (list[j - 1].atomType == AtomModel.AtomType.Type && list[j].atomType != 0)
                {
                        list.Insert(j, LexDictionary.FindAtom("joinGlyph2"));
                }
                
                /*
                if the word length is less than or equal to 4, and the previous atom was "of"; insert a jg2
                */
                if (list[j - 1] == atom_of && list.Count <= 4)
                {
                        list.Insert(j, LexDictionary.FindAtom("joinGlyph2"));
                }

                /*
                if the current atom is a prefix, and the previous atom was NOT a prefix
                Then if the previous atom was NOT an "atom" OR the "is_prefix" flag is set to true, then insert a joinglyph1
                Else; set the is_prefix flag to true.
                */
                if (list[j].atomType == AtomModel.AtomType.Prefix && list[j - 1].atomType != AtomModel.AtomType.Prefix) {
                        if (is_prefix || list[j - 1].atomType != AtomModel.AtomType.Atom) {
                                list.Insert(j, LexDictionary.FindAtom("joinGlyph1"));
                        }
                        is_prefix = true;
                }
        }
        
        /*
        this handles inserting the "primitive" atom
        A word is a primitive \i\f the first atom is a prefix and the word only has two atoms
        The word should get the "primitive" added as a suffix
        */
        if (_atoms.Count == 2 && _atoms[0].atomType == AtomModel.AtomType.Prefix){
                list.Add(atom_primitive);
        }

        // This makes a new list, and removes all instances of joinGlyph1 from it         
        List<AtomModel> list2 = new List<AtomModel>(list);
        for (int num = list2.Count - 1; num >= 0; num--){
                if (list2[num].englishWord == "joinGlyph1")
                {
                        list2.RemoveAt(num);
                }
        }

        // If the resulting list from the step above is shorter than or equal to three atoms, it replaces the returned list.
        if (list2.Count <= 3) {
                list = list2;
        }

        _atoms = list;
        validated = true;
}
