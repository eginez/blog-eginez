+++
title = "Go generics and error handling"
description = ""
tags = [
    "go",
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
# Error handling, and go generics an experience report
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
	//log.Println("Bytes encrypted:", n)

	encrypted.Close()
	armored.Close()
	zipped.Close()
	encbytes = plain.Bytes()
	//log.Println(base64.StdEncoding.EncodeToString(encbytes))
	return
}
```
That works pretty well if what all I wanted to do is exit the program, but for the cases where I needed more than just exiting, this tecnique does not work.

Still unstatisfied witht the states of affair I went down looking for answers in the internet. I started by looking at other people's code too see how they have been solving the same problem. Surprisingly, I found that the vast majority of projects I looked into were basicall `if-checking` on every error.




