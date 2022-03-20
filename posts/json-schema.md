# Using JSON Schema to Define a Grammar For A YAML Document

JSON Schema allowes us to define a grammar for a JSON or YAML document.
However if one is not familiar with JSON schema it can be a little daunting and
confusing to use, especially when you want to use multiple schemas and references.

In this blog post I will go over basic JSON schema concepts, as well as more
advanced concepts such as the `$ref` keyword, and schema composition with the
`oneOf` keyword.

## Learning by Example: JSON Schema For a Collection of Questions of a Quiz

I will be explaining the concepts by going through an example JSON schema that I
created for a personal project of mine.
The schema is for a quiz with multiple categories.
Each category can either have multiple subcategories (nested arbitrarily deep), or multiple questions.
That is or as in `XOR`, meaning we can't mix questions and categories on the same level.
Three types of questions exist: hidden answer questions, choose answer questions,
and music questions, where you have to guess songs.
Example document:

```YAML
---
name: Test Quiz
categories:
  - categoryName: League of Legends
  # category with subcategories
    childNodes:
      - categoryName: Easy
        childNodes:
            # choose answer question
          - questionText: What is the name of Poppy's ultimate?
            correctAnswer: Keeper's Verdict
            wrongAnswers: [Keeper's Hammer, Roundhouse Kick, Hammer of Justice]
      - categoryName: Medium
        points: 2 # default is 1
        childNodes:
            # hidden answer question
          - questionText: Where is Ashe from?
            correctAnswer: Freljord
  - categoryName: Guess The Song
  # simple category with no subcategories
    childNodes:
        # music question
      - questionText: What is this song's name?
        songs:
          - answer: Taylor Swift - Enchanted
            url: https://open.spotify.com/track/04S1pkp1VaIqjg8zZqknR5?si=8323c443ffad49ee
```

The plan is to have three schemas: one for the entire quiz, one for the categories,
and one for the questions.

## Scheming Schemas

Starting of with the most simple schema: the one for the entire quiz.
A quiz has just a name and one or more categories.
I will begin by showing the entire schema and then go over each point to explain it in more detail.

```JSON
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://maxrn.net/quiz.schema.json",
  "title": "Quiz",
  "description": "A collection of questions and categories for my local quiz app",
  "type": "object",
  "properties": {
    "name": {
      "type": "string",
      "minLength": 1
    },
    "categories": {
      "type": "array",
      "items": { "$ref": "category.schema.json" }
    }
  },
  "required": ["name", "categories"],
  "additionalProperties": false
}
```

### One Key At a Time

We start of the schema by specifying the version of JSON schema we are targeting.
This is not a required key, but it is good practice to specify the version of JSON
schema one is targeting.

```JSON
"$schema": "https://json-schema.org/draft/2020-12/schema",
```

Next comes the id, marked by the `$id` keyword.

```JSON
"$id": "https://maxrn.net/quiz.schema.json",
```

Notice the dollar sign: In an earlier JSON schema draft (version 4) the `$id`
used to be just `id`, however I think using `$id` is preferred as it makes it
clear to the reader that this is a standard key that belongs to the schema, and
not to the object which the schema is describing.

`title`, and `description` are pretty self-explanatory I hope:

```JSON
"title": "Quiz",
"description": "A collection of questions and categories for my local quiz app",
```

Now we get to the interesting part: The quiz is an object so we specify the `type` as `object`.

```JSON
"type": "object",
```

After that we can specify constraints for properties by adding entries to the `properties` key.
Note that adding properties here makes no statement on what properties you
are allowed to add to the object or what properties are required.
That can be managed by the properties `additionalProperties`, and `required` respectively.
(And we will look at those two in a minute, hang on.)

```JSON
"properties": {
  "name": {
    "type": "string"
  },
  "categories": {
    "type": "array",
    "items": { "$ref": "category.schema.json" },
    "minItems": 1
  }
},
```

We define constraints for the properties `name` and `categories`.
We define that `name` can only take `string` values and that a `name` has to be
atleast one character long.
For `categories` we say that this key takes an array with at least one item, and
each item of this array is of type `category.schema.json`.
Here we first encounter the `$ref` keyword which is used to reference other JSON schemas.
All this does is say: "each item in this array adheres to the constraints defined in this
schema right here".
In this case it references the schema which defines my quiz categories and we will
look at this schema next.
But before that, we have to finish the quiz schema by going over `additionalProperties` and `required`.

`required` takes an array of strings:

```JSON
"required": ["name", "categories"],
```

Each entry of the array is a key that has to appear in the object we are describing.
Note the keys we specify here don't have to appear in the `properties` key.
We can very well say that a certain key is required without making any further
constraints on its value.
In this case however, I want to make sure that each quiz object specifies its name
and its categories, because I don't want a quiz that has no categories - there
isn't any fun in that.

The last property we look at is `additionalProperties`:

```JSON
"additionalProperties": false
```

By declaring `additionalProperties` to be `false`, I say that I can't add any
properties to my quiz, besides the ones I defined in the `required` and `properties` keys.

### Categorically Correct Categories

Again, I will first paste the entire schema and then go over each (new) key to
explain what it does.

```JSON
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://maxrn.net/category.schema.json",
  "title": "Category",
  "description": "A category in the quiz",
  "type": "object",
  "properties": {
    "categoryName": {
      "type": "string",
      "minLength": 1
    },
    "childNodes": {
      "type": "array",
      "oneOf": [
        {
          "items": {
            "$ref": "#"
          }
        },
        {
          "items": {
            "$ref": "question.schema.json"
          }
        }
      ],
      "minItems": 1
    },
    "points": {
      "enum": [1, 2, 3]
    }
  },
  "required": ["categoryName", "childNodes"],
  "additionalProperties": false
}
```

The beginning is very similar to the quiz schema.

```JSON
"$schema": "https://json-schema.org/draft/2020-12/schema",
"$id": "https://maxrn.net/category.schema.json",
"title": "Category",
"description": "A category in the quiz",
"type": "object",
```

We declare which JSON schema version we are targeting, give the schema a unique
`$id`, give it a `title` and `description`, and we say that we are declaring an `object`.

Moving on to the `properties`, the `categoryName` is equivalent to the quizName:
A string of length >= 1.

```JSON
"properties": {
  "categoryName": {
    "type": "string",
    "minLength": 1
  },
```

The interesting stuff happens in the `childNodes` key.
There is a lot to breakdown here so let's take it step-by-step.
First of all let's remember what we want a category to look like:
Each category should have a name and _either_ a list of subcategories _or_
a list of questions that belong to this category.

```JSON
"childNodes": {
  "type": "array",
  "oneOf": [
    {
      "items": {
        "$ref": "#"
      }
    },
    {
      "items": {
        "$ref": "question.schema.json"
      }
    }
  ],
  "minItems": 1
},
```

The first thing we do is say that `childNodes` should be and array with `"type": "array"`.
And we also say that the array should have atleast on item with `"minItems": 1`.
Now, regarding the entries of this array we make use of the `oneOf` keyword which acts
as a kind of `XOR`.

The expression starting with `oneOf` says that the entries of `childNodes` are
either of type `question.schema.json` or of type `#` which just means a
self-reference, so `category.schema.json`.
