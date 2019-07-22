# Chatito Spec

## 1 -  Overview

Chatito is a domain specific language designed to simplify the process of creating, extending and maintaining
datasets for training natural language processing (NLP) models for text classification, named entity recognition, slot filling or equivalent tasks.

Chatito design principles:

- Simplicity: should be understandable by someone looking at it for the first time

- Speed: generate samples by pulling them from a cloud of probabilities on demand

- Practicality: this tool is meant to help people who use it, the design should be guided by the community needs

Following those principles this is an example of the language and its generated output:

```
%[greet]('training': '2')
    ~[hi] @[name?] ~[whatsUp?]

~[hi]
    hi
    hey

@[name]
    Janis
    Bob

~[whatsUp]
    whats up
    how is it going
```

This code could produce a maximum of 18 examples, the output format is independent from the DSL language,
although it is recommended to use a newline delimited format to just stream results to a file, a format like ndjson is recommended over plain json and using the `training` entity argument to limit the dataset size is recommended for large dataset where there should be no need to generate all variations.

That said, the earlier DSL code generates two training examples for the `greet` intent. Here is the `Newline Delimited JSON` (ndjson.org) examples generated from the previous code:

```
[{"type":"Text","value":"hi how is it going"}]
[{"type":"Text","value":"hey "},{"type":"Slot","value":"Bob","slot":"name"}]
```

Given this principles in mind, this document is the specification of such language.

## 2 - Language

A chatito file, is a document containing the grammar definitions. Because of the different encoding formats and range of
non printable characters, this are the requirements of document source text and some terminology:

- Format: UTF-8
- Valid characters: Allow international language characters.
- White space: allows white space character, not horizontal tab
- Line end: new line, carriage return, carriage return + new line (supporting non windows and windows)
- Indentation: should use a 4 space character to define the scope of entities
- Entities: Special keywords with special behaviors used to declare the sentence combinations
- Sentences: 4 space indented text lines after an entity definition
- Definition order: It does not matter if an entity is defined after it is being referenced
- Comments: Lines of text starting with '//' or '#' (no spaces before)
- Imports: Lines of text starting with 'import' keyword followed by a relative filepath
- Entity arguments: Optional key-values that can be declared at intents and slot definitions
- Probability operator: an optional keyword declared at the start of sentences to control the odds.

### 2.1 - Entities
Entities are the way to define keywords that wrap sentence variations and attach some properties to them.
There are three types of entities: `intent`, `slot` and `alias`.

#### 2.1.1 - Intent

The intent entity is defined by the `%[` symbols at the start of a line, following by the entity name and `]`.

Intent names should be at least 1 character long and can contain any characters except `]`, `line end` and `?`
. e.g.: (%[intentName], %[intent_name], %[intent name])

Repeating intent name definitions should not be allowed.

Each intent defined in a file is an entry point for the generation, the intent is the classification tag that is
added to the sentences defined inside. e.g.:

```
%[greet]
    hello
    hi
```

The previous example will generate all possible unique examples for greet (in this case 2 utterances). But there are cases where there is no need to generate all utterances, or when we want to attach some extra properties to the generated utterance, that is where entity arguments can help.

Entity arguments are comma separated key-values declared with the entity definition inside parenthesis. Each entity argument is composed of a key, followed by the `:` symbol and the value. The argument key or value are just strings wrapped with single or double quotes, optional spaces between the parenthesis and comma are allowed, the format is similar to ndjson but only for string values.

By default, intent definitions can expect the `training` and `testing` argument keys, when defined, are used to declare the maximum number of unique examples to generate for the given intent, and splitting them in two datasets, the training dataset is to be used to train the NLU model, and the testing dataset should be used to evaluate the accuracy of the model with examples it never trained with. Creating a testing dataset is not required, but it is important to be aware of the accuracy of your model to detect overfitting and compare against previous accuracies. The generator will first populate the training dataset, then testing dataset until reaching the sum of both values, each value must be `>= 1`. e.g.:

```
%[greet]('training': '2', 'testing': '1')
    hello
    hi
    hola
    salute
```

In this example, the greet intent could generate a maximum of 4 examples, but the declaration only requests 3. The training dataset will contain 2 utterances for greet intent and the testing dataset 1. Other entity arguments are ignored by default and their functionality depend on the dataset generator/adapter, this means that each adapter may use the other entity arguments differently in its own context (e.g.: Rasa/Snips adapter may expect different entity arguments).

Nesting entities: Sentences defined inside an intent can refer to slots and alias entities.

#### 2.1.2 - Slot
The slot entity is defined by the `@[` symbols at the start of a line, following by the name of the slot and `]`.

Slot names should be at least 1 character long and can contain any characters except `]`, `line end`, `?` and `#` (as # is used for variations).
. e.g.: (@[slotName], %[slot_name], %[slot name])

Repeating slot name definitions should not be allowed.

From the output perspective, a slot is the tag that is added the relevant words in a generated sentence. e.g.:

```
%[greet]
    ~[hi] @[name?]

~[hi]
    hi
    hey

@[name]
    Janis
    Bob
```

Slot entities referenced within sentences, can have `?` symbol at the end of the reference name. (e.g.: @[name?]).
In that context, the `?` symbol means that the slot combination is optional, and could be omitted at generation. The probabilities of being omitted are defined by the number of sentence definitions at the entity. If the entity defines only one sentence, then the probabilities of empty string will be 50%, if the sentences defines 2 sentences, the probabilities of being omitted are 33.3333%, and so on.

Slots provide a particular property at their definitions called variations.

- Variations: There are cases where a slot combination only makes sense in a given context, variations allow to map one slot to different sentences in different contexts. e.g.:

```
%[ask_for_delivery]
    my parcel should be delivered in @[delivery_time#time_in_hours]
    my parcel should be delivered @[delivery_time#relative_time]

@[delivery_time#time_in_hours]
    3 days
    5 hours

@[delivery_time#relative_time]
    as fast as possible
    quickly
```

In this example, both combinations map to the `delivery_time` slot, but 
the generated sentences only generate their variations contexts where they make sense.

Slot definitions can have entity arguments too but there are no default argument keys. Entity arguments are ignored by default and their functionality depends on the dataset adapter, this means that each adapter may use the entity arguments differently in its own context (e.g.: Rasa/Snips adapter may expect different entity arguments like for pre-build date parsing, or text value aliases mappings).

Nesting entities: Sentences defined inside a slot can only reference alias entities.

#### 2.1.3 - Alias
The alias entity is defined by the `~[` symbols at the start of a line, following by the name of the alias and `]`.
Alias are just variations of a word and does not generate any tag. By default if an alias is referenced but not defined (like in the next example for `how are you`, it just uses the alias key name, this is useful for making a word optional but not having to add the extra lines of code defining a new alias. (This 'auto alias' behavior can be configurable) e.g.:

```
%[greet]
    ~[hi] ~[how are you?]

~[hi]
    hi
    hey
```

Same as with slots, alias references can be ommited using a `?` symbol at the end of the reference name. (e.g.: ~[hi?]).

When an alias is referenced inside a slot definition, and it is the only token of the slot sentence, by default the generator will tag the generated alias value as a `synonym` of the alias key name.

Alias definitions are not allowed to declare entity arguments.

Nesting entities: Sentences defined inside aliases can reference slots and other aliases but preventing recursive loops.


### 2.2 - Importing chatito files

To allow reusing entity declarations. It is possible to import another chatito file using the import keyword. Importing another chatito file only allows using the slots and aliases defined there, if the imported file defines intents, they will be ignored since intents are generation entry points.

As an example, given two chatito files:

```
# file slot1.chatito
@[slot1]
    s1v1
    s1v2
```

and

```
# file main.chatito
import ./slot1.chatito

%[some intent]
    ~[word] @[slot1]
```

The file `main.chatito` will import all alias and slot definitions from `./slot1.chatito`.
The text next to the import statement should be a relative path from the main file to the imported file. Imports can be nested, and the path is always relative to the file that declares the reference.

Note: Chatito will throw an exception if two imports define the same entity.


### 2.2 - Controlling probabilities

The way Chatito works, is like pulling samples from a cloud of possible combinations and avoiding duplicates. Once the sentences definitions gain complexity, the max possible combinations increments exponentially, causing a problem where the generator will most likely pick sentences that have more possible combinations, and omit some sentences that may be more important at the dataset. To overcome this problem, semantics for controlling the data generation probabilities are provided.

#### 2.2.1 - Frequency distribution strategies

When generating samples for an entity, the generator will randomly pick a sentence model using one of the two frequency distribution strategies available: `regular` or `even`.

For a regular distribution strategy, each sentence probabilities are defined by their maximum possible combinations, in other words, a sentence that can produce more combinations will have more probabilities. For even distribution strategy, sentence probabilities are the same.

The distribution strategy can be declared as an argument at the entity level. If not declared, the generator should use the default strategy configured (at the IDE or CLI level), if there is no default definition, then `regular` should be the default.

Lets look at an example, here, all the alias entities are defined at `./aliases.chatito`, and are named by the maximum possible combinations each provide:

```
import ./aliases.chatito

%[intent with a maximum of 1k combinations]('distribution': 'regular')
    first sentence equals ~[100 maximum combinations]
    second sentence equals ~[50 maximum combinations] multiplied by ~[10 maximum combinations]
    third sentence equals ~[400 maximum combinations]
```

Since the intent declares a `regular` distribution, this would be the odds:

|            | Max combinations | Weight | Probability % |
|------------|------------------|--------|---------------|
| sentence 1 | 100              | 100    | 10%           |
| sentence 2 | 500              | 500    | 50%           |
| sentence 3 | 400              | 400    | 40%           |


Now the code to get an `even` distribution:

```
import ./aliases.chatito

%[intent with a maximum of 1k combinations]('distribution': 'even')
    first sentence equals ~[100 maximum combinations]
    second sentence equals ~[50 maximum combinations] multiplied by ~[10 maximum combinations]
    third sentence equals ~[400 maximum combinations]
```

For `even` distribution using the previous example:

|            | Max combinations | Weight | Probability % |
|------------|------------------|--------|---------------|
| sentence 1 | 100              | 1      | 33.3333%      |
| sentence 2 | 500              | 1      | 33.3333%      |
| sentence 3 | 400              | 1      | 33.3333%      |


#### 2.2.1 - Sentence probability operator

The sentence probability operator is defined by the `*[` symbol at the start of a sentence following by the probability value and `]`. The probability value may be expressed in two ways, as a plain number (considered as weighted probabilty, e.g.: `1`) or as a percentage value (a number ending with `%`, e.g.: `33.3333%`), but once an entity defines a probabilty as either weight or percentage, then all the other sentences for that entity should use the same type. Inconsistencies declaring entity sentence probabilty values should be considered an input error and if the value is not a valid integer, float or percentual value, the input should be considered as simple text and not as a sentence probability definition.
 
NOTE: If the probabilty value is a percentage type, then and the sum of all sentence probabilty operators declared inside the entity definition should never exceed 100.

Lets continue with some examples:

```
%[intent with a maximum of 1k combinations]
    *[20%] first sentence ~[100 maximum combinations]
    second sentence ~[50 maximum combinations] multiplied by ~[10 maximum combinations]
    third sentence ~[400 maximum combinations]
```

The previous example, declares `20%` probabilties for the first sentence. This would be odds table for the two strategy distributions:

|            | Max combinations | % with even | % with regular        |
|------------|------------------|-------------|-----------------------|
| sentence 1 | 100              | 20%         | 20%                   |
| sentence 2 | 500              | 40%         | 44.4444% (500*80/900) |
| sentence 3 | 400              | 40%         | 35.5556% (400*80/900) |


When probabilty value is a weight with regular distribution, multiply that value with the maximum combinations for that sentence, if distribution is even, that value is the actual weighted probability. E.g.:

```
%[intent with a maximum of 1k combinations]
    *[2] first sentence ~[100 maximum combinations]
    second sentence ~[50 maximum combinations] multiplied by ~[10 maximum combinations]
    third sentence ~[400 maximum combinations]
```

And the odds table:

|            | Max combinations | even weight | even % | regular weight | regular % |
|------------|------------------|-------------|--------|----------------|-----------|
| sentence 1 | 100              | 2           | 50%    | 200            | 18.1818%  |
| sentence 2 | 500              | 1           | 25%    | 500            | 45.4545%  |
| sentence 3 | 400              | 1           | 25%    | 400            | 36.3636%  |


NOTE: Be careful when using probabilty operator, because if the sentence reaches its max number of unique generated values, it will start producing duplicates and slowing down the generator that filters duplicates.

## 3 - Data Generation

The entry points for the data generation are the intent definitions, for each intent definition available:
- If the intent does not specify the 'training' or 'testing' arguments, generate all possible unique combinations and add them to the training dataset.

- Respect probabilty operator declarations and distribution strategy.

- Generate unique combinations for the training and testing dataset until the provided sum of both argument numbers are reached.

- Recursive loop references should be prevented.
