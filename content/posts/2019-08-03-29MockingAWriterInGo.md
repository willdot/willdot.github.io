---
title:  How I was able to mock ioutil.WriteFile in go
author: Will Andrews
date: 2019-08-03
order: 29
published: true
---


I'm currently writing an application that takes some data from the user and writes that data to a file for later use. The function that does the saving is very small and is called from within another function. My problem, is that I wanted to write a unit test to test that my parent function did everything it should in terms of preparing the data, but I don't want it to call the save function and actually save data to disk. 

I looked everywhere for a way around faking the package ioutil and it's WriteFile function, but found nothing. Then I remembered something that Mat Ryer had said in one of his conference talks. It was something like "if they haven't provided an interface, then make one yourself". So that's what I did!

## The problem

Here is the code I had before that would use ioutil.WriteFile() to save my data to a json file.

``` go
func Save(filename string, requestData map[string]interface{}) error {

	file, err := json.MarshalIndent(requestData, "", " ")

	if err != nil {
		return err
	}

	err = ioutil.WriteFile(filename+".json", file, 0644)

	return err
}
```

All it does is takes a filename, and some data and saves it as a .json file. But if I want to test the function that calls it, it will save data which I don't want.

Unfortunately, ioutil.Writefile() doesn't implement an interface, so it can't be mocked. So instead I'm going to create one.

## The solution

The signature for ioutil.WriteFile() is as follows:

``` go
func WriteFile(filename string, data []byte, perm os.FileMode) error
```

I create an interface that specifies that function with that signature:

``` go
type Writer interface {
	WriteFile(filename string, data []byte, perm os.FileMode) error
}
```

I can now create a struct and give it a method that implements that interface:

``` go
// FileWriter implements is an abstraction of ioutil.WriterFile
type FileWriter struct {
}

// WriteFile implements the Writer interface that's been created so that ioutil.WriteFile can be mocked
func (w FileWriter) WriteFile(filename string, data []byte, perm os.FileMode) error {
	return ioutil.WriteFile(filename, data, perm)
}
```

What I've done here is create a method on a struct that implements the interface, and returned the ioutil.WriteFile() function. So as soon as my method is called, it calls the ioutil function. They both have the same parameters and returns, so it's nice and simple.

Now all I need to do is change my original save function to take the Writer interface as a parameter.

``` go
func Save(filename string, requestData map[string]interface{}, w Writer) error {

	file, err := json.MarshalIndent(requestData, "", " ")

	if err != nil {
		return err
	}

	err = w.WriteFile(filename+".json", file, 0644)

	return err
}
```

Now when I call that function from my other code, I can pass in a FileWriter struct.

``` go
Save(filename.(string), request, FileWriter{})
```

On the method I want to test, if I add a field to the struct for a the Writer interface and create a method that creates a new struct with a FileWriter, and it will all work. Then when I want to test my method, I can create a different struct that implements the Writer interface, writer my dummy code in there and pass it to the Struct I want to test, like so:

``` go
type IWantToTestThis struct {
	MyFileWriter Writer
}

func NewIWantToTestThis() IWantToTestThis {
	return IWantToTestThis {
		MyFileWriter : FileWriter{},
	}
}
```

Then my test 
``` go
type FakeFileWriter struct {
}

func (f FakeFileWriter) WriteFile(filename string, data []byte, perm os.FileMode) error {
	return nil
}

func TestThing(t *testing.T) {

	testThing := IWantToTestThis{
		MyFileWriter: FakeFileWriter{},
	}

	var something map[string]interface{}

	got := testThing.Save("test", something, testThing.MyFileWriter)

	if got != nil {
		t.Errorf("Failed")
	}
}
```

This test creates a fake struct and implements the Writer interface by adding a method that implements the WriteFile function on the interface, but it does nothing.

Then when I call the method I want to save, I'm passing in my fake struct which means in the tested function, when it comes to calling the WriteFile function, it will call my fake method and return without saving a file. Problem solved!