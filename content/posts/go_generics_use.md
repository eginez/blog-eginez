+++
title = "Error hanlding in golang, an experience report"
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

Error handlign sits high amongst the things that I can probably do better while writing go, thus while working on a recent project I decided to invest some time looking into how to improve the way I have been handling errors.

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

That is a short of example showcasing something pretty obvious: there is a lot repetitive error handlign pretty much doing the same thing. My first intuiton was to replace all the error checking statemtns with a function that would check for the status of an error variable and then just panic.


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
That works pretty well if what all I wanted to do is exit the program, but for the cases where I needed more than just exiting, this tecnique does not work.

Still unstatisfied witht the states of affair I went down looking for answers in the internet. I started by looking at other people's code too see how they have been solving the same problem. Surprisingly, I found that the vast majority of projects I looked into were basicall `if-checking` on every error.

That seemed (and it still seems) less than ideal to me. And as far as I can tell it is a source of complains amongst gophers. So much so that Rob Pike himself write a [post] (https://blog.golang.org/errors-are-values) on this specific topic a couple of years ago. In it he describes a clever technique to deal with errors. He makes the point that error should be treated as values(which they should) and goes on to provide an example on how to program around errors. I won't describe his whole solution however I'd like to hightlight a key part of his example. Consider the following interface and function

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

Give the above, and as explained in Rob's post, you could rewritte a function to use the above solution

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

At a first glance this is great solution for the problem. The error checking logic is localized to a single function, and the consumer only has to worry about checking the error when is about to return. However, when I tried to extende this solution to work on more than just once use case, I started running into some problems.

The first thing I noticed is that one might want to do more than just call `ew.write`. In fact, ideally, we would handle arbitrary error throwing functions.

Closures seem like the write tool for that, our new error check structure would like so:

```go
type AnyFn func() (interface{}, error)

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
Great, we now have a function called `TryFile`.It will perform an error throwing closure if, there are no prior errors. Thus we can now can do something like this.

```go
func errorChecker() (error) {
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
Other than verbosity of the closure declaration, this acomplishes what we needed. Until we realize that our error ttthrowing function could return an arbitrary type. Replacing the concrete type with go's interface{} would look like this:

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

And we can now utilize our error checking structure much more freely

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

Few observations:

 - We now have to type assert the return value. This seems silly, the return type is well known in all cases. (just not the same). Why can't go let me express that!. Falling back to the type that expresses no information, seems hardly like the right solution.
 - The above fragment looks marginally better(if not the same) than the manually error checking fragment, mostly due to the typing of the closure argument.





I felt I could build on top of this to allow for something..

- Talk about an erro inteface that recieves a lambda function
   - What type to  return?, why an interface? 
   - what about the arguments? what to do with the arguments
- How would generics have fixed this
   - give an exaple with fake syntax on it 
   
   

   
   







