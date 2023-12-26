---
title: Mangling JSON Data with jq
description: A gentle introduction to jq a command-line JSON processing tool
date: 2023-12-25
tldr:  Using jq can help you with manipulation of JSON data.
draft: false
tags: ["json", "jq", "cli", "terminal"]
---

# Introduction 

https://jqlang.github.io/jq/

This is how `jq` is described at it's homepage

> `jq` is like sed for JSON data - you can use it to slice and filter and map and transform structured data with the same ease that `sed`, `awk`, `grep` and friends let you play with text.

> `jq` is written in portable C, and it has zero runtime dependencies. You can download a single binary, `scp` it to a far away machine of the same type, and expect it to work.

> `jq` can mangle the data format that you have into the one that you want with very little effort, and the program to do so is often shorter and simpler than you'd expect.

Now to understand a bit better what `jq` really is let my quote few more paragraphs from its docs. If you are familiar with how piping in `bash` is used you may find a slight resamblence to its philosophy.

> A `jq` program is a "filter": it takes an input, and produces an output. There are a lot of builtin filters for extracting a particular field of an object, or converting a number to a string, or various other standard tasks.

> Filters can be combined in various ways - you can pipe the output of one filter into another filter, or collect the output of a filter into an array.

Now this may seem a bit intimidating, afterall we just wanted to mangle some JSON, so let's get our hand dirty doing that on some superhero themed examples. By the end of this article you will no longer be that guy who is still using `grep` on JSON.

# Processing superhero data with jq

We find a JSON suitable for our processing needs and save it somewhere handy.

https://developer.mozilla.org/en-US/docs/Learn/JavaScript/Objects/JSON#json_structure

Let's assume we have this `json` formatted blob of data saved as  `superherosquad.json` and we want to process it with `jq`. 



```json
{
  "squadName": "Super hero squad",
  "homeTown": "Metro City",
  "formed": 2016,
  "secretBase": "Super tower",
  "active": true,
  "members": [
    {
      "name": "Molecule Man",
      "age": 29,
      "secretIdentity": "Dan Jukes",
      "powers": ["Radiation resistance", "Turning tiny", "Radiation blast"]
    },
    {
      "name": "Madame Uppercut",
      "age": 39,
      "secretIdentity": "Jane Wilson",
      "powers": [
        "Million tonne punch",
        "Damage resistance",
        "Superhuman reflexes"
      ]
    },
    {
      "name": "Eternal Flame",
      "age": 1000000,
      "secretIdentity": "Unknown",
      "powers": [
        "Immortality",
        "Heat Immunity",
        "Inferno",
        "Teleportation",
        "Interdimensional travel"
      ]
    }
  ]
}
```

## Pretty-print a json file

It can be helpful to pipe the response through `jq` to pretty-print it. The simplest `jq` program is the expression `.`, which takes the input and produces it unchanged as output.

```bash
cat superherosquad.json | jq '.'
# or without torturing the cat
jq '.' superherosquad.json
```

## Selecting a single field

To select a single field we can just use the name of the field `.homeTown`

```bash
jq '.homeTown' superherosquad.json
```
Which outputs 
```
"Metro City"
```

## Selecting just the superhero array 

To select just the array of superheros w


```bash
jq '.members' superherosquad.json
```

```json
[
  {
    "name": "Molecule Man",
    "age": 29,
    "secretIdentity": "Dan Jukes",
    "powers": [
      "Radiation resistance",
      "Turning tiny",
      "Radiation blast"
    ]
  },
  {
    "name": "Madame Uppercut",
    "age": 39,
    "secretIdentity": "Jane Wilson",
    "powers": [
      "Million tonne punch",
      "Damage resistance",
      "Superhuman reflexes"
    ]
  },
  {
    "name": "Eternal Flame",
    "age": 1000000,
    "secretIdentity": "Unknown",
    "powers": [
      "Immortality",
      "Heat Immunity",
      "Inferno",
      "Teleportation",
      "Interdimensional travel"
    ]
  }
]
```


## Selecting first superhero from the superhero array

Now to select only the first member of the superhero array we can do 

```bash
jq '.members[0]' superherosquad.json
```

```json
{
  "name": "Molecule Man",
  "age": 29,
  "secretIdentity": "Dan Jukes",
  "powers": [
    "Radiation resistance",
    "Turning tiny",
    "Radiation blast"
  ]
}
```


## Array slicing 

We can use array slicing syntax when accessing array, if we want to work with subarray based on the indexes like this 

```
[1:3]
```
will be of length 2, containing the elements from index 1 (inclusive) to index 3 (exclusive).

```bash
jq '.members[1:3]' superherosquad.json
```


```json
[
  {
    "name": "Madame Uppercut",
    "age": 39,
    "secretIdentity": "Jane Wilson",
    "powers": [
      "Million tonne punch",
      "Damage resistance",
      "Superhuman reflexes"
    ]
  },
  {
    "name": "Eternal Flame",
    "age": 1000000,
    "secretIdentity": "Unknown",
    "powers": [
      "Immortality",
      "Heat Immunity",
      "Inferno",
      "Teleportation",
      "Interdimensional travel"
    ]
  }
]
```


## Selecting first superpower of the first superhero

Similarly if we wanted to get the first superpower of the first superhero we can simply continue using this syntax chaining it together like so 

```
.members[0].powers[0]
```

```bash
 jq '.members[0].powers[0]' superherosquad.json
```

```
"Radiation resistance"
```

## Getting age of all superheros

Now we may be intrested in a just a single field from the superhero array like age, we can simply do the by doing 

```
.members[].age
```

```bash
jq '.members[].age' superherosquad.json
```

```
29
39
1000000
```


## Finding maximum age 

To find a maximum age we might be tempted to do fallback to something like `sort`  and `head` after getting the age of all superheros using `jq` like this 


```bash
jq '.members[].age' superherosquad.json | sort | head -n 1
```

```
1000000
```

but there is a way to do this with just `jq` by using `map` and `max`


```bash
jq '.members | map(.age) | max' superherosquad.json
```

```
1000000
```

now you may be thinking well thats not really that much better, but now imagine you do not want to just find the `max` age but you actually want to output the data of superhero with max age, that's where  the `sort | head` approach falls apart, but with `jq` you can simply reach to `max_by`

```bash
jq '.members | max_by(.age)' superherosquad.json
```

```json
{
  "name": "Eternal Flame",
  "age": 1000000,
  "secretIdentity": "Unknown",
  "powers": [
    "Immortality",
    "Heat Immunity",
    "Inferno",
    "Teleportation",
    "Interdimensional travel"
  ]
}
```

## Finding a hero with max age less than 100

This is very similar to the previous example with one change, we first want to `filter` out any heroes which do not satisfy a boolean predicate, more specifically we want `age <= 100` to be true. We achieve that by adding `map(select(.age <= 100))` before we do the `max_by`

```bash
jq '.members | map(select(.age <= 100)) | max_by(.age)' superherosquad.json
```

```json
{
  "name": "Madame Uppercut",
  "age": 39,
  "secretIdentity": "Jane Wilson",
  "powers": [
    "Million tonne punch",
    "Damage resistance",
    "Superhuman reflexes"
  ]
}
```

## Regex matching

Just as we were able to filter for heroes that are not older than 100, we can also do regex matching to select only matching entries, so lets find all heroes with name matching regex 

```
[dl]{1}ame
```

```bash
jq '.members | map(select(.name | test("[dl]{1}ame")))' superherosquad.json
```

```json
[
  {
    "name": "Madame Uppercut",
    "age": 39,
    "secretIdentity": "Jane Wilson",
    "powers": [
      "Million tonne punch",
      "Damage resistance",
      "Superhuman reflexes"
    ]
  },
  {
    "name": "Eternal Flame",
    "age": 1000000,
    "secretIdentity": "Unknown",
    "powers": [
      "Immortality",
      "Heat Immunity",
      "Inferno",
      "Teleportation",
      "Interdimensional travel"
    ]
  }
]
```

## Renaming and deleting fields

Now we want to rename the array of `powers` to  `superpowers` and get rid of the field `secretIdentity` entirely. Once again we reach out for `map` to assign the values to a new key using `.superpowers = .powers` and delete them after using `del`.

```bash
jq '.members | map(.superpowers = .powers| del(.powers, .secretIdentity))' superherosquad.json
```

```json
[
  {
    "name": "Molecule Man",
    "age": 29,
    "superpowers": [
      "Radiation resistance",
      "Turning tiny",
      "Radiation blast"
    ]
  },
  {
    "name": "Madame Uppercut",
    "age": 39,
    "superpowers": [
      "Million tonne punch",
      "Damage resistance",
      "Superhuman reflexes"
    ]
  },
  {
    "name": "Eternal Flame",
    "age": 1000000,
    "superpowers": [
      "Immortality",
      "Heat Immunity",
      "Inferno",
      "Teleportation",
      "Interdimensional travel"
    ]
  }
]
```


## Generating a custom structure

Now let's try going the other way around, instead of removing and renaming fields to fit our needs, we want to generate a custom JSON structure using the data in the original JSON.

More specifically we want to use `current_age` instead of `age`, `superheroname` instead of name, add a new field `is_cool` which is obviously always `true` and also create a new field `first_power` which is the first power in the `powers` array.


```bash
jq '.members[] | {current_age: .age, superheroname: .name, is_cool: true, first_power: .powers[0]}' superherosquad.json
```

```json
{
  "current_age": 29,
  "superheroname": "Molecule Man",
  "is_cool": true,
  "first_power": "Radiation resistance"
}
{
  "current_age": 39,
  "superheroname": "Madame Uppercut",
  "is_cool": true,
  "first_power": "Million tonne punch"
}
{
  "current_age": 1000000,
  "superheroname": "Eternal Flame",
  "is_cool": true,
  "first_power": "Immortality"
}
```


## Other tools

There is many more like `jq` for other common formats like `XML`, `yaml`, `toml`, `csv` etc.

- Go `yq` a lightweight and portable command-line YAML, JSON and XML processor https://github.com/mikefarah/yq/
- Python `yq` jq wrapper for YAML, XML, TOML documents https://kislyuk.github.io/yq/
- `xmlstar` for `XML` https://xmlstar.sourceforge.net/doc/UG/xmlstarlet-ug.html
- `xsv` for `csv` https://github.com/BurntSushi/xsv 

## Conclusion 

JSON is a widely employed structured data format typically used in most modern APIs and data services. Unfortunately, shells such as `bash` have no good way of working with structured data like JSON directly. This means that working with JSON via the command line can be painful, just as trying to parse HTML with regex is. You can fix this awkwardness by using `jq` a cool command-line processor for JSON.

This was more of a hands on introduction to basics of `jq` for a more formal introduction I really recommend reading trough `man jq` and also this in-browser interactive `jq` guide https://ishan.page/blog/2023-11-06-jq-by-example/


Now the key takeaway here is that using the right tool for the job instead of trying to hack something together with just pure `bash` or whatever you favourite shell is may make you life significantly less miserable. 