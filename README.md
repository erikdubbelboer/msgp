msgp
=======

[![forthebadge](http://forthebadge.com/badges/uses-badges.svg)](http://forthebadge.com)
[![forthebadge](http://forthebadge.com/badges/certified-snoop-lion.svg)](http://forthebadge.com)

This is a code generation tool for serializing Go `struct`s using the [MesssagePack](http://msgpack.org) standard. It is targeted 
at the `go generate` [tool](http://tip.golang.org/cmd/go/#hdr-Generate_Go_files_by_processing_source).

### Use

*** Note: `go generate` is a go 1.4+ feature. ***

In a source file, include the following directive:

```go
//go:generate msgp
```

The `msgp` command will generate `Unmarshal`, `Marshal`, `WriteTo`, and `ReadFrom` methods for all exported struct
definitions in the file. You will need to include that directive in every file that contains structs that 
need code generation. The generated files will be named {filename}_gen.go by default (but can 
be overridden with the `-o` flag.)

Field names can be overridden in much the same way as the `encoding/json` package. For example:

```go
type Person struct {
	Name       string `msg:"name"`
	Address    string `msg:"address"`
	Age        int    `msg:"age"`
	Hidden     string `msg:"-"` // this field is ignored
	unexported bool             // this field is also ignored
}
```

Running `msgp` on a file containing the above declaration would generate the following methods:

```go
func (z *Person) Marshal() ([]byte, error)

func (z *Person) Unmarshal(b []byte) error

func (z *Person) WriteTo(w io.Writer) (n int, err error)

func (z *Person) ReadFrom(r io.Reader) (n int, err error)
```

The `msgp/enc` package has a function called `CopyToJSON` which can take MessagePack-encoded binary
and translate it directly into JSON. It has reasonably high performance, and works much the same way that `io.Copy` does.


### Features

 - Proper handling of pointers, nullables, and recursive types
 - Backwards-compatible decoding logic
 - Efficient and readable generated code
 - JSON interoperability
 - Support for embedded fields and anonymous structs (see Benchmark 1) and inline comma-separated declarations (see Benchmark 2)
 - Identifier resolution (see below)
 - Support for Go's `complex64` and `complex128` types through extensions
 - Generation of both `[]byte`-oriented and `io.Reader/io.Writer`-oriented methods.

For example, this will work fine:
```go
type MyInt int
type Data []byte
type Struct struct {
	Which  map[string]*MyInt `msg:"which"`
	Other  Data              `msg:"other"`
}
```
As long as the declarations of `MyInt` and `Data` are in the same file, the parser will figure out that 
`MyInt` is really an `int`, and `Data` is really just a `[]byte`. Note that this only works for "base" types 
(no slices or maps, although `[]byte` is supported as a special case.)


### Status

Very alpha. Here are the known limitations:

 - All fields of a struct that are not Go built-ins are assumed to satisfy the `io.WriterTo` and `io.ReaderFrom`
   interface. This will only *actually* be the case if the declaration for that type is in a file touched by the code generator.
   The generator will output a warning if it can't resolve an identifier in the file, or if it ignores an exported field.
 - Like most serializers, `chan` and `func` fields are ignored, as well as non-exported fields.
 - Methods are only generated for `struct` definitions.
 - Encoding/decoding of `interface{}` doesn't work yet.

### Performance

As you might imagine, the generated code is quite a lot more performant than reflection-based serialization. Here 
are the two built-in benchmarking cases provided. Each benchmark writes the struct to a buffer and then extracts 
it again. Keep in mind, too, that the generated methods support `io.Reader` and `io.Writer` *natively*. (This means
that, unlike many other serializers, the methods never have to hold the serialized object in memory, unlike `Marashal`- and
`Unmarshal`-style methods. So, in general, the generated methods run slower than they would if they operated on `[]byte`,
but they generate very little garbage.)


##### Benchmark 1

Type specification:
```go
type TestType struct {
	F     *float64          `msg:"float"`
	Els   map[string]string `msg:"elements"`
	Obj   struct {
		  ValueA string     `msg:"value_a"`
		  ValueB []byte     `msg:"value_b"`
	}                       `msg:"object"`
	Child *TestType         `msg:"child"`
}
```

|  Method | Time | Heap Use | Heap Allocs |
|:-------:|:----:|:--------:|:-----------:|
| msgp codegen | 2553ns | 129 B | 5 allocs |
| [ugorji/go](http://github.com/ugorji/go) | 8467ns | 2015 B | 43 allocs |
| encoding/json | 11976ns | 2004 B | 33 allocs |



##### Benchmark 2

Type specification:
```go
type TestFast struct {
	Lat, Long, Alt float64
	Data []byte
}
```
|  Method | Time | Heap Use | Heap Allocs |
|:-------:|:----:|:--------:|:-----------:|
| msgp codegen | 1151ns | 57 B | 1 allocs |
| [ugorji/go](http://github.com/ugorji/go) | 5199ns | 1227 B | 27 allocs |
| encoding/json | 8120ns | 1457 B | 11 allocs |


### TODO

 - Support JSON method generation. The front-end for the existing generator will work fine; we just need new templates.
 - Support non-struct types.
 - Support finer-grained directive control (e.g. read string as []byte and vice-versa), probably via struct tags
 - Support `Seek`ing internal fields in the encoded binary

If you've found a bug, please open an issue! Thanks.
