+++
title = "Error hanlding and go generics, and experience report"
description = ""
tags = [
    "golang",
    "error handling",
    "generics"
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

That seemed (and it still seems) less than ideal to me. And as far as I can tell it is a source of complains amongst gophers. So much so that Rob Pike himself write a [post] (https://blog.golang.org/errors-are-values) on this specific topic a couple of years ago. In it he describes a clever technique to deal with errors. He makes the point that error should be treated as values(which the should) and goes on to provide an example on how to program around errors. I won't describe his whole solution however I'd like to hightlight a key part of his example. Consider the following interface and function

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

Give the above, and as explained in Rob's post, you could rewritte a function like solution

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

At a first glance this is great solution for the problem. The error checking logic is localized to a single function, and the consumer only has to worry about checking the error when is about to return.
I felt I could build on top of this to allow for something..

- Talk about an erro inteface that recieves a lambda function
   - What type to  return?, why an interface? 
   - what about the arguments? what to do with the arguments
- How would generics have fixed this
   - give an exaple with fake syntax on it 
   
   

   
   







