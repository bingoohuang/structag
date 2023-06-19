# :sparkles: `structag`

fork from [this repo](https://github.com/matt1484/spectagular)

A simple go library for parsing and managing struct tags as structs using generics and nothing but the standard library.

In general, it's not very often you need to parse struct tags, but if you do this library is designed to help by
automatically converting struct tags to their expected value as well as cache/validate said values. A good example of
how this could be useful is a package like the `encoding/json` package which uses a well-defined set of struct tag
options that could be described with the following struct:

```go
type JSONTag struct {
// here $name indicates that it is either the first option or the name of the field if empty
Name      string `structag:"$name"`
OmitEmpty bool   `structag:"omitempty"`
String    bool   `structag:"string"`
}
```

By using this package you could then parse any struct with `json` tags like so:

```go
// an example of a typical struct with json tags
type Person struct {
Name string `json:",omitempty"`
Age  int    `json:"age"`
}

// look at the "json" struct tags and convert the options to type JSONStructTag
fieldTags, err := structag.Parse[JSONTag]("json", reflect.TypeOf(&Person{}))
// fieldTags is a []structag.FieldTag that is equivalent to:
/*
[]FieldTag{{
    FieldName: "Name",
    FieldIndex: 0,
    Value: JSONTag{ Name: "Name", OmitEmpty: true, String: false },
}, {
    FieldName: "Age",
    FieldIndex: 1,
    Value: JSONTag{ Name: "age", OmitEmpty: false, String: false },
}}
*/
```

You can even set up a cache of parsed tags which is good for repetitive workflows like web servers.

```go
// functionally equivalent to the above example
cache, err := structag.New[JSONTag]("json")
fieldTags, err := cache.GetOrAdd(reflect.TypeOf(&Person{}))
// there are also individual Get/Add methods
```

## How it works

`structag` supports all the simple go types including:

- integers: `int`, `int8`, `int16`, `int32`, `int64`, `uint`, `uint8`, `uint16`, `uint32`, `uint64`
- floats: `float32`, `float64`
- `time.Duration`
- complex: `complex64`, `complex128`
- `string`
- `bool`

as well as pointers/slices (not arrays) of any one of the above. There is also support for parsing custom types that
implement this interface:

```golang
type OptionUnmarshaler interface {
UnmarshalTagOption(field reflect.StructField, value string) (reflect.Value, error)
}
```

Internally, `strconv` is used to parse most types and `time.ParseDuration` for `time.Duration` fields. Other parsing
rules are as follows:

- Anything with a `structag` value of `$name` will always match the first field in its entirety. If it is empty, then it
  will default to the field name (i.e. how `encoding/json` uses struct tags).
- Fields can be marked as `required` which will cause parsing to return an error if the field fails parsing or is not
  found. If a field is not `required` then errors will be ignored unless they are formatting errors that could affect
  other fields (i.e. missing an end bracket/quote).
- Fields are comma delimeted and are expected to be of one of the following forms:
    - `key=value` (true `bool` values are implicit and dont require value)
    - `key='value'` (start/end quotes will be ignored and can be escaped with `\\'`. parsing will fail if the end quote
      isn't matched)
    - `key=[value,...]` (if the field is a slice then everything between the brackets will be parsed with the above
      rules, otherwise the brackets will just be ignored. ending brackets that are literal must be escaped with `\\]`)

### Limitations:

This library does not currently support:

- `map` (although I guess you could parse JSON)
- `arrays` (supported slices, but sized arrays are not currently supported)
- `time.Time` (mostly due to not wanting to deal with varying formats for date strings, I'd like to fix this at some
  point though)
- matrices (i.e. `[][]int`, having to recursively match inner brackets seems painful and struct tags really shouldn't be
  used for such complicated logic IMO)
