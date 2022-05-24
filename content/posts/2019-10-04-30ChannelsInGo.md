---
title:  Channels and Go routines
author: Will Andrews
date: 2019-10-04
order: 30
published: true
---

I started investigating how to use channels to run concurrent things and then finish tasks in a go routine depending on the outcome of another routine. 

### How I used channels

The first thing to create would be the function that sleeps and then writes to the channel when done. 

I create a channel variable of type bool. Call my startSession function and pass in the channel and then wait for that channel to have some data that I can receive to continue. 

``` go
func main() {

	sessionFinished := make(chan bool)

	go startSession(sessionFinished)

	<-sessionFinished
}

func startSession(ch chan bool) {

	fmt.Println("GO GO GO")

	time.Sleep(time.Duration(10) * time.Second)

	fmt.Println("FINISHED")

	ch <- true
}
```

The line <-sessionFinished is blocking, meaning that nothing else will happen until the sessionFinished channel receives some data. 

The startSession function sleeps for 10 seconds, and then writes a bool value to the channel. The blocking line in the main func will then receive the data and the function will exit.

This will output:

```
GO GO GO

// there will be a 10 second gap here

FINISHED
```

### Doing other things while waiting for that channel

Now that I have something that will pass data to a channel to signal that the session has finished, I need to create another go routine to do something until it gets the sessionFinished channel data.

I've created a function that takes a channel as an input and then starts an infinite loop. Inside this loop, is a select command. A select command takes data from channels and depending on which case returns first, it will carry out that code. So in our example, if the channel outputs data, it will exit the function, otherwise it will do our sleep and then printing the value of i to the console.

```go
func doSomething(ch chan bool) {
	i := 0
	for {
		select {
		case <-ch:
			return
		default:
			time.Sleep(time.Duration(1) * time.Second)

			i++
			fmt.Println(i)
		}
	}
}

func main() {

	sessionFinished := make(chan bool)

	go startSession(sessionFinished)

	doSomething(sessionFinished)
}
```

This will output the following:
```
GO GO GO
1
2
3
4
FINISHED
```

### Multiple channels

What if I wanted the select command to do something else based on a different channel? That's as simple as creating another channel, passing it into the function and then adding it to the case of the select. Then which ever channel outputs first, will have it's code run.

I've modified the start session to sleep a random time between 5 and 10, and then created another channel that does the same thing as the first channel I created. I pass this into my function and then which ever finishes first will print that it won to the console and then exit.

I've also modified the startSession function to print the channel's name and the amount of time it will sleep for.

Complete code:
```go
func main() {

	rand.Seed(time.Now().UnixNano())

	fmt.Println(("GO GO GO"))
	sessionOne := make(chan bool)
	sessionTwo := make(chan bool)

	go startSession(sessionOne, "1")
	go startSession(sessionTwo, "2")

	doSomething(sessionOne, sessionTwo)
}

func doSomething(ch1, ch2 chan bool) {
	i := 0
	for {
		select {
		case <-ch1:
			fmt.Println("channel 1 won")
			return
		case <-ch2:
			fmt.Println("channel 2 won")
			return
		default:
			time.Sleep(time.Duration(1) * time.Second)

			i++
			fmt.Println(i)
		}
	}
}

func startSession(ch chan bool, channelName string) {

	t := 5 + rand.Float64()*(10-5)

	fmt.Printf("Channel %v will run for %v seconds\n", channelName, t)
	time.Sleep(time.Duration(t) * time.Second)

	ch <- true
}
```

This will output something like this, and each time you run it, it will be different:
```
GO GO GO
Channel 1 will run for 5.490952514229586 seconds
Channel 2 will run for 6.912125321340266 seconds
1
2
3
4
5
channel 1 won
```

This what a short intro into channels. I used what learnt to create a kart timing service, which can be found in the timingService [here](https://github.com/willdot/KartTiming)  on github. It will start a race session that sleeps for a set time, and then each of the racers will lap a time and then finally the amount of laps each racer has done is printed. It uses the same principal I've spoken about, with the addition of another data channel to allow the racers lap times to be logged after the session has finished.

