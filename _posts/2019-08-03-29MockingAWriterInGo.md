---
title:  How I was able to mock ioutil.WriteFile in go
author: Will Andrews
date: 2019-08-03
order: 29
published: false
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

Unfortunately, ioutil.Writefile() doesn't implement an interface, so it can't be mocked. 