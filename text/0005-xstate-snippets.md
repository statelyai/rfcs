- **Start Date:** 2022-05-27
- **RFC PR:** (leave this empty)

# RFC: XState Snippets

## Summary

This RFC proposes introducing snippets to help write XState code and include them with our VS Code extension.

## Motivation

XState has an extensive API with many features. It can be challenging to remember the correct syntax for every part when building a machine. Code snippets could be helpful as they provide a template to get started.

## Detailed design

Snippets in VS Code are templates that make it easier to enter repeating code patterns. We can use this for writing out machines.

To help remember the snippets, we should prefix them so that they are easy to remember and don't clash with the code you typically write.

Luckily, it doesn't seem typical to start code with `xs` (**xs**tate), which could be our prefix.

#### A few suggestions for snippets

`xsm` - which would be short for XState Machine.
![xsm](https://user-images.githubusercontent.com/167574/170657981-6b94717d-7629-44a4-92ac-9598f39967a7.gif)

`xstm` - XState Typegen Machine.
![xstm](https://user-images.githubusercontent.com/167574/170658001-c95e2b92-b37b-4db1-9b4a-4ef81f2e4b3c.gif)

#### Other ideas for snippets
- `xss` - XState State.
- `xsus` - XState useSelector.

### Technical Background

Snippets live in a `.code-snippets` file. We can include this file with our extension to augment the snippets on a user's machine. You can also create snippets independently using the command `Configure User Snippets` in VS Code.

### Implementation

Snippets are a list of strings that will get inserted into a file when a prefix is typed and triggered. Each string can have placeholders that the user can cycle through and change.

For XState snippets, we will include the relevant imports needed as the first string. Having imports in the first string works best if the user has [enabled organizing imports on save](https://community.vscodetips.com/tonyhicks20/automatically-organize-imports-346d). With that setting enabled, the imports are moved to the right place and merged with any existing imports on save.

We provide a template for the part of the API relevant to the snippet for the subsequent strings.

A full snippet for the XState Typegen Machine looks like this:

```json
"XState Typegen Machine": {
    "prefix": "xstm",
    "body": [
      "import { createMachine } from 'xstate';",
      "const ${1:nameOf}Machine = createMachine({\n\tid: '${1:nameOf}',\n\ttsTypes: {},\n\tschema: {\n\t\tcontext: {} as { value: string },\n\t\tevents: {} as { type: 'FOO' },\n\t},\n\tcontext: {\n\t\tvalue: '',\n\t},\n\tinitial: '${2:initialState}',\n\tstates: {\n\t\t${2:initialState}: {$0},\n\t},\n});"
    ],
    "description": "Outline for XState Typegen Machine"
  },
```

## How we teach this

We add a section to the README file of the extension with helpful text and videos. The file will be visible in the VS Code marketplace and the extensions panel. For inspiration, we can look at popular snippet libraries such as [Reactjs code snippets](https://marketplace.visualstudio.com/items?itemName=xabikos.ReactSnippets).

## Drawbacks

We risk the snippets going out of sync with the XState API.

## Alternatives

There are other solutions for creating and using snippets, such as TextExpander, and Rocket Typist, to name a few. And there's also the option of not using snippets or copy-pasting from the XState documentation.

## Unresolved questions

Which snippets should be provided in the extension, if any?
