---
title:  Creating the 1p savings challenge calculator - Pt1 Go Backend
author: Will Andrews
date: 2019-07-31
order: 28
published: true
---

I bank with Monzo (for so many reasons that I won't post here). One of the cool things they offer is IFTTT integration, and one of their applets is the 1p Savings Challenge. It goes like this:

1st January it puts 1p into a savings pot.
2nd January it puts 2p into a savings pot.
3rd January it puts 3p into a savings pot.

It keeps going until the 31st of December where it'll put £3.65 into a savings pot.

The first few months were fine as it was pennies per month, but soon it started being £30 a month, which soon started eating into my monthly allowance I set myself to spend. I wanted to work out how much I needed from 1 payday until they next, so that I could account for this into my budget (another great thing about Monzo). And this is where I got the idea for this app.

## The 1p savings challenge calculator - Requirements

* Like most people, I get a set payday but if that day falls on a weekend or bank holiday, its the closest working day. So I need to be able to say how much will be deposited into my savings jar between payday on date 'X' and the day before the next payday on date 'Y'.
<br>
* It runs from 1st January until 31st December, so day 1 will always be 1st January. But with 2020 being a leap year, the 31st December won't be day 365 like it is this year. So leap years need to be taking into account.
<br>
* I'm learning Go at the moment, so the backend will be written in Go.
<br>
* It needs to be hosted on Azure and containerized, so using Docker is required.
<br>
* There will need to be a front end, and as I'm still learning Angular, this framework will be used.

## The library package to calculate the amount saved

I need to create a package that will take 2 dates, a start and end date and then return a result that will be how much will be saved between these 2 dates. I'm creating a package instead of having functions on my main server code, so that the code for calculating this can be used by others. <3 open source.

#### Testing testing, 123

When I write packages like this, I like to use TDD so that it's not over complicated.

The first thing I need to work out, is the first day of the year, of the dates passed in. So for 28/04/2019 I need to return 1/1/2019.

The test code I used is this, which will try 2 test dates. 28/4/2019 which will return 1/1/2019 and 28/4/2022 which will return 1/1/2022.

``` go
func TestCalculateStartDateOfYear(t *testing.T) {
	cases := []struct {
		Name        string
		CurrentYear time.Time
		Expected    time.Time
	}{
		{
			"2019",
			time.Date(2019, time.Month(4), 28, 0, 0, 0, 0, time.UTC),
			time.Date(2019, time.Month(1), 1, 0, 0, 0, 0, time.UTC),
		},
		{
			"2022",
			time.Date(2022, time.Month(4), 28, 0, 0, 0, 0, time.UTC),
			time.Date(2022, time.Month(1), 1, 0, 0, 0, 0, time.UTC),
		},
	}

	for _, test := range cases {
		t.Run(test.Name, func(t *testing.T) {
			got := calculateStartDateOfYear(test.CurrentYear)

			if got != test.Expected {
				t.Errorf("got %v, want %v", got, test.Expected)
			}
		})
	}
}
```

Now to implement the 'calculateStartDateOfYear' code. Nice and simple. Get the year value of the incoming date and return a new date with that year.

``` go
func calculateStartDateOfYear(currentDate time.Time) time.Time {

	year, _, _ := currentDate.Date()
	return time.Date(year, time.Month(1), 1, 0, 0, 0, 0, time.UTC)
}
```

#### Get the day number of the year

Next I need to work out the day numbers of each date. So 4th January will be day 4 of the year. 3rd Feb will be day 34 of the year.

So I need a function that will calculate the number of days between 2 dates. The first date will be the 1st January of the current year. The next date will either be the start of end date the user has requested.

Here is the test code.

``` go
func TestCalculateDaysBetween(t *testing.T) {

	cases := []struct {
		Name     string
		Start    time.Time
		End      time.Time
		Expected int
	}{
		{
			"1st Jan to 2nd Jan",
			time.Date(2019, time.Month(1), 1, 0, 0, 0, 0, time.UTC),
			time.Date(2019, time.Month(1), 2, 0, 0, 0, 0, time.UTC),
			2,
		},
		{
			"1st Feb to 28th feb",
			time.Date(2019, time.Month(2), 1, 0, 0, 0, 0, time.UTC),
			time.Date(2019, time.Month(2), 28, 0, 0, 0, 0, time.UTC),
			28,
		},
		{
			"1st Jan to 31st dec - leap year",
			time.Date(2020, time.Month(1), 1, 0, 0, 0, 0, time.UTC),
			time.Date(2020, time.Month(12), 31, 0, 0, 0, 0, time.UTC),
			366,
		},
		{
			"1st Jan to 31st dec",
			time.Date(2019, time.Month(1), 1, 0, 0, 0, 0, time.UTC),
			time.Date(2019, time.Month(12), 31, 0, 0, 0, 0, time.UTC),
			365,
		},
	}

	for _, test := range cases {
		t.Run(test.Name, func(t *testing.T) {
			got := calculateDaysBetween(test.Start, test.End)

			if got != test.Expected {
				t.Errorf("got %v, want %v", got, test.Expected)
			}
		})
	}
}
```

Here is the implementation code. In Go, to get the difference between 2 dates, you take away the start date from the end date and it returns the duration, of which you can specify the units. So seconds, minutes or hours. It won't return days, so I divide the hours by 24 to get the days.

Note, this also takes into account leap years, which is also one of our requirements.

``` go
func calculateDaysBetween(start, end time.Time) int {

	days := end.Sub(start).Hours() / 24

	return int(days + 1)
}
```

#### How much?!

Now that I can get the number of days between the first of the year and each of the dates given by the user, I can use these figures to work out how much will be saved between the dates.

So for the total for day 1 of the year and day 2 of the year will be 3 (1p + 2p). The total for day 32 of the year (1st Feb) and day 59 (28th Feb) will be 66795 (32 + 33 + 34 .... and so on).

``` go
func TestCalculateCostOfDays(t *testing.T) {

	cases := []struct {
		Name     string
		Start    int
		End      int
		Expected int
	}{
		{
			"1st Jan to 2nd Jan",
			1,
			2,
			3,
		},
		{
			"1st Feb to 28th feb",
			32,
			59,
			1274,
		},
		{
			"Whole year",
			1,
			365,
			66795,
		},
	}

	for _, test := range cases {
		t.Run(test.Name, func(t *testing.T) {
			got := calculateCostOfDays(test.Start, test.End)

			if got != test.Expected {
				t.Errorf("got %v, want %v", got, test.Expected)
			}
		})
	}
}
```

To achieve this, I used a simple loop to go through each day until the last day, adding each day as I went along.

``` go
func calculateCostOfDays(start, end int) int {

	result := 0

	for i := start; i <= end; i++ {
		result += i
	}

	return result
}
```

#### All the pieces of the puzzle

I now have all the functions I need to make this work, so I now crete an exportable function, that accepts a start and end date and returns a calculated figure. I also add in a check, to make sure that the start date, isn't after the end date....

``` go
// ErrStartDateAfterEndDate is for when user tries to use a start date that is after the end date
var ErrStartDateAfterEndDate = errors.New("Start date can't be after end date")

// CalculateHowMuchToSaveBetweenDays takes a start date and an end date and returns how much to save for this period
func CalculateHowMuchToSaveBetweenDays(start, end time.Time) (int, error) {

	if end.Before(start) {
		return 0, ErrStartDateAfterEndDate
	}

	yearStart := calculateStartDateOfYear(start)

	startDayFromFirstOfYear := calculateDaysBetween(yearStart, start)

	endDayFromFirstOfYear := calculateDaysBetween(yearStart, end)

	totalToSave := calculateCostOfDays(startDayFromFirstOfYear, endDayFromFirstOfYear)

	return totalToSave, nil
}
```

And of course, there were some tests.

``` go
func TestCalculateHowMuchToSaveBetweenDays(t *testing.T) {

	cases := []struct {
		Name          string
		Start         time.Time
		End           time.Time
		Expected      int
		ExpectedError error
	}{
		{
			"1st Jan to 2nd Jan",
			time.Date(2019, time.Month(1), 1, 0, 0, 0, 0, time.UTC),
			time.Date(2019, time.Month(1), 2, 0, 0, 0, 0, time.UTC),
			3,
			nil,
		},
		{
			"1st Feb to 28th feb",
			time.Date(2019, time.Month(2), 1, 0, 0, 0, 0, time.UTC),
			time.Date(2019, time.Month(2), 28, 0, 0, 0, 0, time.UTC),
			1274,
			nil,
		},
		{
			"1st Jan to 31st Dec 2019",
			time.Date(2019, time.Month(1), 1, 0, 0, 0, 0, time.UTC),
			time.Date(2019, time.Month(12), 31, 0, 0, 0, 0, time.UTC),
			66795,
			nil,
		},
		{
			"1st Jan to 31st Dec 2020 (leap year)",
			time.Date(2020, time.Month(1), 1, 0, 0, 0, 0, time.UTC),
			time.Date(2020, time.Month(12), 31, 0, 0, 0, 0, time.UTC),
			67161,
			nil,
		},
		{
			"Start date after end date",
			time.Date(2021, time.Month(1), 1, 0, 0, 0, 0, time.UTC),
			time.Date(2020, time.Month(12), 31, 0, 0, 0, 0, time.UTC),
			0,
			ErrStartDateAfterEndDate,
		},
	}

	for _, test := range cases {
		t.Run(test.Name, func(t *testing.T) {
			got, err := CalculateHowMuchToSaveBetweenDays(test.Start, test.End)

			if err != test.ExpectedError {
				t.Errorf("got error %v, want %v", got, test.ExpectedError)
			}

			if got != test.Expected {
				t.Errorf("got %v, want %v", got, test.Expected)
			}
		})
	}
}
```

## A Go http server

Now I have the package to do the calculation, I need a HTTP server that my Angular app will call, with 2 dates and will then use the calculator package. In Go this is super easy to create a http server.

I will be using the Gorilla/Mux package which makes the routing of handlers super easy. 

First thing I do is create a new Mux router. I then use the built in http package to start a http server, passing the router and a port. That's how easy it is to create a http server in Go!

``` go
package main

import (
	"log"
	"net/http"
	"os"

	"github.com/gorilla/mux"
)

func main() {
	var PORT string
	if PORT = os.Getenv("PORT"); PORT == "" {
		PORT = "8080"
	}
	router := mux.NewRouter()

	err := http.ListenAndServe(":"+PORT, router)

	if err != nil {
		log.Fatal(err)
	}

}
```

I now need to create a route and a handler that will handle my calculate request. To do this I create a new function that takes a http.ResponseWriter and a http.Request. Once I have this, I tell my router to add a new route by calling the HandleFunc method, passing it a route path and a handler function.

My handler will try and decode a request body which should be in JSON. This body should contain 2 properties. A start date and end date. It will then use the calculate package to work out how much is needed between these 2 dates, and then return it in the response as JSON.

I've also added in some error handling so that if decoding the body fails or the calculation fails, it will return a http 400 error response.

``` go

type request struct {
	Start time.Time
	End   time.Time
}


func GetBudget(w http.ResponseWriter, r *http.Request) {

	var req request

	decoder := json.NewDecoder(r.Body)

	err := decoder.Decode(&req)

	if err != nil {
		http.Error(w, err.Error(), 400)
		return
	}

	amount, err := calculator.CalculateHowMuchToSaveBetweenDays(req.Start, req.End)

	if err != nil {
		http.Error(w, err.Error(), 400)
		return
	}

	result := fmt.Sprintf("£%v", float64(amount)/100)

	json.NewEncoder(w).Encode(result)
}
```

Now I can plug this into my main function code.

``` go
    router := mux.NewRouter()

    router.HandleFunc("/calculate", GetBudget)
```

Now, if I run my main.go code, and open up Postman to send a POST request to http://localhost:8080/calulate with a body of ...

``` json
{
    "Start" : "2019-01-01T00:00:00Z",
    "End" : "2019-01-02T00:00:00Z"
}
```

it should return a response body of 

``` json
{
    "£0.01"
}
```

## CORS - Dreaded CORS

I know for a fact, that when trying to make a call to this server from an Angular app, I will get a cross origin error. So I used a package rs/cors to add in an allowed origin. I then have to slightly tweak the code to start the http server, by creating a new http.Server,passing in a handler and then calling the ListenAndServe method on it. The finished code looks like this.

``` go
func main() {

    var PORT string
    if PORT = os.Getenv("PORT"); PORT == "" {
        PORT = "8080"
    }

    router := mux.NewRouter()

    router.HandleFunc("/calculate", GetBudget)

    c := cors.New(cors.Options{
        AllowedOrigins: []string{
            "http://localhost:3001",
        },
    })

    handler := c.Handler(router)

    srv := &http.Server{
        Handler: handler,
        Addr:    ":" + PORT,    
    }
	
    log.Fatal(srv.ListenAndServe())
}
```

## Enter Docker

Now that I have everything for my back end in place, I need to create a Docker image for it, so that it can be containerized on Azure.

That's nice and simple. 

``` dockerfile
# Specify the base image needed for the Go application
FROM golang:1.12

# Create an /app directory within the image that will hold the application
# source files
RUN mkdir /app

# Copy everything in the root directory into the /app directory
ADD . /app

# Specify that all other commands will now come from within the /app directory
WORKDIR /app

# Go get dependancies
RUN go get -d -v ./...

# Run go build to compile the binary executable of the Go program
RUN go build -o main .

# Start command that will run the program
CMD ["/app/main"]
```

## Coming up

The next post will be on creating the Angular app that will call this http server and display the result to the user.

Source code for this can be found on Github here: [1 Penny Savings Calculator project](https://github.com/willdot/PennySavingsCalculator) 