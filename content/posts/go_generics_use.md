+++
title = "Error handling in golang, an experience report"
description = ""
tags = [
    "golang",
    "error handling",
]
date = "2017-08-05"
categories = [
    "Development",
    "golang",
]
menu = "main"
+++
In the spirit of contributing to the `golang` community, I would like to document my experience while trying to make error handling a  little more palatable. 

Error handling sits high among the things that I would like to do better in go, thus while working on a recent project I decided to invest some time looking into how to improve.

I'll start by showing code snippets of what I thought could have been improved.
```go
//Loads the data structures
func Load(root string, keyRing string, p pgp.PromptFunction) (index *MeerkatsIndex, key *MeerkatsKeys, e error) {

	//Load keys
	index = &MeerkatsIndex{}
	key = &MeerkatsKeys{}
	fr, e := os.OpenFile(filepath.Join(root, ".keys"), os.O_RDWR, os.FileMode(0700))
	e = key.Read(fr)
	if e != nil {
		return
	}

	//Load index
	fr, e = os.OpenFile(filepath.Join(root, ".index"), os.O_RDWR, os.FileMode(0700))
	if e != nil {
		return
	}
	ent, entList, e := LoadKeysFromDisk(keyRing, p, key)
	if e != nil {
		return
	}

	e = index.Read(fr, ent, entList)
	return
}
```

That is a short of example showcasing something pretty obvious: there is a lot repetitive error handling pretty much doing the same thing. My first intuition was to replace all the error checking statements with a function that would check for the status of an error variable and then just panic.


```go
//check for errors
func logAndExitIfError(e error) {
	if e != nil {
		panic(e)
	}
}

func EncryptPipeline(to pgp.EntityList, from *pgp.Entity, secret []byte) (encbytes []byte, e error) {
	e = nil
	plain := bytes.NewBuffer(nil)
	zipped := gzip.NewWriter(plain)
	armored, e := armor.Encode(zipped, "PGP MESSAGE", nil)
	logAndExitIfError(e)
	encrypted, e := pgp.Encrypt(armored, to, from, nil, nil)
	logAndExitIfError(e)

	_, e = encrypted.Write(secret)
	logAndExitIfError(e)
	//...
}
```
That works pretty well if all I want to do is exit the program, but for the cases where I needed more than just exiting, this technique does not work.

Still unsatisfied with the state of affairs I went looking for answers in the internet. I started by looking at other people's code too see how they have been solving the same problem. Surprisingly, I found that the vast majority of projects I looked into were basically `if-checking` on every error.

That seems less than ideal to me. And as far as I can tell it is a source of complaint amongst gophers. So much so that Rob Pike himself wrote a [post] (https://blog.golang.org/errors-are-values) on this specific topic a couple of years ago. In it, he describes a clever technique to deal with errors. He makes the point that errors should be treated as values (which they should) and goes on to provide an example on how to program around errors. I won't describe his whole solution however I'd like to highlight a key part of his example. Consider the following interface and function:

```go
type errWriter struct {
    w   io.Writer
    err error
}

func (ew *errWriter) write(buf []byte) {
    if ew.err != nil {
        return
    }
    _, ew.err = ew.w.Write(buf)
}
```

Give the above, and as explained in Rob's post, you could write a function to use the above solution, roughly looking like so:

```go
...
ew := &errWriter{w: fd}
ew.write(..)
ew.write(..)

// and so on
if ew.err != nil {
    return ew.err
}
```

At a first glance this is great solution for the problem. The error checking logic is localized to a single function and the consumer only has to worry about checking the error when the function is  about to return. However, when I tried to extend this solution to work on more than just one use case, I started running into some problems.

The first thing I noticed is that one might want to do more than just call `ew.write`. In fact, ideally, we should be able to handle arbitrary error throwing functions.

Closures seem like the right tool for that. Our new error check structure would like so:

```go
type ErrorCatcher struct {
	err error
}

func (ec *ErrorCatcher) TryFile(fn func() (*os.File, error)) (ret *os.File) {
	if ec.err != nil {
		return nil
	}

	ret, ec.err = fn()
	return ret
}
```

Great, we now have a function called `TryFile`. It will execute an error throwing closure if there are no prior errors. Thus we can now can do something like this:

```go
func possibleErrorThrowingFunction() (error) {
	erc := ErrorCatcher{}
	file1 := erc.TryFile(func() (*os.File, error) {
		return os.Open("/some/fine/file.txt")
	})

	file2 := erc.TryFile(func() (*os.File, error) {
		return os.Open("/some/fine/file.txt")
	})
    
    return erc.err
}
```

Other than the verbosity of the closure declaration, this accomplishes what we need if all we needed to return was of type `File`. Replacing the concrete type with go's `interface{}` would look like this:

```go
func (ec *ErrorCatcher) TryAny(fn func() (interface{}, error)) interface{} {
	if ec.err != nil {
		return nil
	}

	var ret interface{}
	ret, ec.err = fn()
	return ret
}
```

And we can now utilize our error checking structure much more freely, like so:

```go
func GetFleInfo() (info os.FileInfo, err error) {
	erc := ErrorCatcher{}
	file := erc.TryAny(func() (interface{}, error) {
		return os.Open("/some/fine/file.txt")
	}).(*os.File)

	fileInfo := erc.TryAny(func() (interface{}, error) {
		return file.Stat()
	}).(os.FileInfo)

	return fileInfo, erc.err
}
```

A few observations:

 - We now have to type assert the return value. This seems silly since the return type is well known in all cases (just not the same). Why can't go let me express that!? Falling back to a type that expresses no information hardly seems like the right solution.
 - The above fragment looks marginally better, than the manual error checking fragment, mostly due to the typing of the closure argument. Maybe we could have an alias to denote the type of a closure? After all go's compiler can already infer variable's types, why not function's type? It will cut down on a lot of typing.
 
 In my opinion, the above experience highlights the importance of generics. Having some way to express an arbitrary type is very useful when trying to express solutions to problems like the one above:
 
 ```go
 func (ec *ErrorCatcher) TryAny(fn func() (`T`, error) `generic: T` {
 ...
 ...
     return fn()
 }
 ```

