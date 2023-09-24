---
title:  The dangers of struct embeding
author: Will Andrews
date: 2023-09-24
order: 23
published: true
---

Go comes with the ability to embed structs into other structs. It allows you to "reuse" fields and functions from one struct (the base) with another struct that embeds it. This can often be misunderstood as inheritance that you get from object orientated programming languages. While it can be useful and prevents having to duplicate code, it can also cause bugs that can be hidden in plain sight, which is what I encountered recently.

### What is struct embeding?

Let's say we need to write some code that handles configuration of different message brokers. Each message broker has some common fields, `url` and `port`. But then they each have some different fields that aren't common. 

``` go
type RabbitConfig struct {
	url       string
	port      int
	queueName string
}

type NatsConfig struct {
	url        string
	port       int
	streamName string
}
```

What we could do it create some base config that contains the shared config fields and then embed them into the other types.

``` go
type BaseConfig struct {
	url     string
	port    int
}
type RabbitConfig struct {
	BaseConfig
	queueName string
}

type NatsConfig struct {
	BaseConfig
	streamName string
}

func newRabbitConfig() RabbitConfig {
	c := RabbitConfig{
		queueName: "test queue",
	}

	c.port = 1
	c.url = "http://localhost"

	return c
}

func newNatsConfig() NatsConfig {
	c := NatsConfig{
		streamName: "test stream",
	}

	c.port = 1
	c.url = "http://localhost"

	return c
}
```

Now we have 2 different config types that have common fields but you've not needed to write those common fields out twice. 

### Function overriding

If the base struct contains a receiver function, you can override that in the other types as long as it has the same function signature. Then when you call that function, it was call the base structs function unless there is an override function in which case it will call that. 

So if we want to have a function that joins the `url` field and the `port` field together, we can add that to the base config struct. Then if there is something else that needs doing we can add an override function.

```Go
func (b *BaseConfig) createAddr() string {
	return fmt.Sprintf("%s:%d", b.url, b.port)
}

func (n *NatsConfig) createAddr() string {
	return fmt.Sprintf("%s:%d/nats", n.url, n.port)
}
```

Then if we create a Rabbit config, it will use the base configs function but if we use the NATs config, it will use the overriden function.

### DANGER

So where's the danger? Once you're inside the the base functions function, the type is the base type, even if you created a variable of a type that embeded the base type. So if we create a variable of type Rabbit config and called the `createAddr` function (which is the bases function), the function on the base type won't have access to the `queueName` field of the Rabbit Config type. Seems reasonable.

However there's a catch. If you're base function calls any other function, it will call the function of the base type. This is easiest to explain using some print statements.

``` go
func (b *BaseConfig) run() {
	fmt.Println("running from base")
	b.printAddr()
}

func (n *NatsConfig) run() {
	fmt.Println("running from NATs")
	n.printAddr()
}

func (b *BaseConfig) printAddr() {
	addr := b.createAddr()
	fmt.Printf("addr: %s\n", addr)
}

func main() {
	nc := newNatsConfig()
	nc.run()
}
```

Here we have created some `run()` functions that print out what the type and then calls another function `printAddr()`. 

This `printAddr()` function calls the `createAddr()` function we created earlier. If you remember we created an override function for the NATs type that also put `/nats` on the end of the address.

But we haven't got an override function for `printAddr()` on the NATs type and so it naturally calls the base function. This is where the bug can be found. 

We are now inside the base types `printAddr()` which is calling the base `createAddr()` not the NATs types. 

So when we run this, we get the output:
```
running from NATs
addr: http://localhost:1
```

Note how the addr does not contain the `/nats` that we added to the end of the addr in the NATs `createAddr()` override function.

This means that if you have some complex code that's hard to follow, you can easily mistake the type you're in and think you're calling a function when in fact you're calling a different one.

### Summary
Struct embeding can be useful if you're creating different structs with lots of common fields and functionality. However in my opinion they should be reached out to as a last resort and instead you should favor a bit of copy paste. 

I was pairing recently with someone and we spent over an hour working out why something wasn't working only to realise that a base function was a NOOP funtion and it wasn't obvious that this was the function being called, not the override functions.