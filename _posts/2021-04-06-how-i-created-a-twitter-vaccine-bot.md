---
layout: article
title: How I Created a Twitter Vaccine Bot
tags: twitter-bot covid personal-projects
description: >
  COVID vaccination appointments are hard to find. Here, I'll show you how
  I created a bot to automatically find new vaccination appointments and
  tweet about them.
sitemap:
    priority: 1
    changefreq: 'monthly'
    lastmod: 2021-04-07T11:00:00-05:00
---

<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

With COVID vaccines becoming more available to younger age groups, I decided to start looking to schedule a vaccination appointment for myself. Unfortunately, I found that trying to schedule an appointment was a game of constantly hitting the refresh button.

The local pharmacy chain in my area that I was trying to get vaccinated at provided a sign up form online for appointments, but there was no way to be notified when new appointments were published. Lore circulated on local Twitter saying, "new appointments are published at midnight, check then!" Or, "check around 8 AM and 4 PM, that's when they release new appointments!" Clearly, there was no official answer.

After a lucky refresh, I finally found a pharmacy with availability. Straight to the sign up page I went. I selected an appointment time, filled out about 10 minutes worth of forms, and finally hit the submit button. Instead of getting a sweet confirmation of my newly reserved appointment, I received an error that said the appointment time had already been taken. Clearly, someone completed their forms faster than I did and was able to get their appointment scheduled before mine. "No worries, I'll just select another timeslot," I naively thought. Going back to the time selection page, I found that all the previous appointments that were available had disappeared. Growing increasingly frustrated, I thought that there must be another way. And, of course, like most things, there is. Luckily, I'm a CS student that is completely remote (thanks pandemic), so I had plenty of time on my hands to devise a solution. 

Ultimately, out of the goodness of my heart and a little spite, I created a Twitter bot that automatically tweets when new vaccine appointments are found at my local pharmacy chain. It has been extremely successful and has helped plenty of people get signed up for vaccine appointments. Here, I'll show you how I did that.

<blockquote align="center" class="twitter-tweet"><p lang="en" dir="ltr">This account tweets when new vaccine appointments become available at HyVee pharmacies in the Omaha/Lincoln/Council Bluffs area. Appointments go fast so enabling notifications is recommended.<br><br>You&#39;ll still need to register for an appointment by using the link in each tweet.</p>&mdash; HyVee Vaccine Tracker (@HyveeTracker) <a href="https://twitter.com/HyveeTracker/status/1377029788293066756?ref_src=twsrc%5Etfw">March 30, 2021</a></blockquote>

# Exploring the API
The first thing I had to do was figure out how the appointment sign up page worked on the [pharmacy's webiste](https://www.hy-vee.com/my-pharmacy/covid-vaccine). After playing around in the web inspector for a bit, I found that a GET request was being made to the endpoint `https://www.hy-vee.com/my-pharmacy/api/graphql`. Now, this looked like a GraphQL API call, but I don't have experience in GraphQL. Thankfully, it was easy to figure out. The body that was being sent in the request looked like this:

{% highlight json %}
{
  "operationName": "SearchPharmaciesNearPointWithCovidVaccineAvailability",
  "variables": {
    "radius": 10,
    "latitude": redacted,
    "longitude": redacted
  },
  "query": "query SearchPharmaciesNearPointWithCovidVaccineAvailability($latitude: Float!, $longitude: Float!, $radius: Int! = 10) {\n  searchPharmaciesNearPoint(latitude: $latitude, longitude: $longitude, radius: $radius) {\n    distance\n    location {\n      locationId\n      name\n      nickname\n      phoneNumber\n      businessCode\n      isCovidVaccineAvailable\n      covidVaccineEligibilityTerms\n      address {\n        line1\n        line2\n        city\n        state\n        zip\n        latitude\n        longitude\n        __typename\n      }\n      __typename\n    }\n    __typename\n  }\n}\n"
}
{% endhighlight %}

This didn't look too complicated. There's a named operation which describes itself, some location variables, and they query itself. Great, let's try playing around with those variables.

I found that by changing the radius, along with te latitude and longitude, I was able to search for available pharmacies with vaccine appointments in an arbitrary area.

Let's also look at what the response is to this request before we move forward.

{% highlight json %}
{
  "data": {
    "searchPharmaciesNearPoint": [
      {
        "distance": 5.17,
        "location": {
          "locationId": "131df0a6-d970-4abd-8666-123a97b7d9c8",
          "name": "Omaha #10",
          "nickname": "156th & Maple",
          "phoneNumber": "+14024930390",
          "businessCode": "1474",
          "isCovidVaccineAvailable": false,
          "covidVaccineEligibilityTerms": "No eligibility terms defined.",
          "address": {
            "line1": "3410 N 156th St",
            "line2": null,
            "city": "Omaha",
            "state": "NE",
            "zip": "68116",
            "latitude": 41.289924,
            "longitude": -96.160812,
            "__typename": "LocationAddress"
          },
          "__typename": "Location"
        },
        "__typename": "SearchLocationNearResult"
      },
      {
        "distance": 5.545,
        "location": {
          "locationId": "76a421b1-4230-47ce-9998-2a882dac3f55",
          "name": "Omaha #04",
          "nickname": "Fort Street Hy-Vee",
          "phoneNumber": "+14024932089",
          "businessCode": "1467",
          "isCovidVaccineAvailable": false,
          "covidVaccineEligibilityTerms": "No eligibility terms defined.",
          "address": {
            "line1": "10808 Fort St",
            "line2": null,
            "city": "Omaha",
            "state": "NE",
            "zip": "68164",
            "latitude": 41.307727,
            "longitude": -96.082551,
            "__typename": "LocationAddress"
          },
          "__typename": "Location"
        },
        "__typename": "SearchLocationNearResult"
      },
      {
        "distance": 7.136,
        "location": {
          "locationId": "3c49e93f-5cc7-4b64-83b7-52d098df9465",
          "name": "Omaha #08",
          "nickname": "Linden Market Hy-Vee",
          "phoneNumber": "+14024932911",
          "businessCode": "1471",
          "isCovidVaccineAvailable": false,
          "covidVaccineEligibilityTerms": "No eligibility terms defined.",
          "address": {
            "line1": "747 N 132nd St",
            "line2": null,
            "city": "Omaha",
            "state": "NE",
            "zip": "68154",
            "latitude": 41.26592,
            "longitude": -96.11776,
            "__typename": "LocationAddress"
          },
          "__typename": "Location"
        },
        "__typename": "SearchLocationNearResult"
      },
      {
        "distance": 8.142,
        "location": {
          "locationId": "33672f43-45be-4002-990f-8c18fa7021af",
          "name": "Omaha #11",
          "nickname": "180th & Pacific",
          "phoneNumber": "+14023344444",
          "businessCode": "1478",
          "isCovidVaccineAvailable": false,
          "covidVaccineEligibilityTerms": "No eligibility terms defined.",
          "address": {
            "line1": "1000 S 178th St",
            "line2": null,
            "city": "Omaha",
            "state": "NE",
            "zip": "68118",
            "latitude": 41.250367,
            "longitude": -96.195654,
            "__typename": "LocationAddress"
          },
          "__typename": "Location"
        },
        "__typename": "SearchLocationNearResult"
      },
      {
        "distance": 9.231,
        "location": {
          "locationId": "2523a368-7a05-4595-abba-0c416014052e",
          "name": "Omaha #05",
          "nickname": "Peony Park Hy-Vee",
          "phoneNumber": "+14023848668",
          "businessCode": "1470",
          "isCovidVaccineAvailable": false,
          "covidVaccineEligibilityTerms": "No eligibility terms defined.",
          "address": {
            "line1": "7910 Cass St",
            "line2": null,
            "city": "Omaha",
            "state": "NE",
            "zip": "68114",
            "latitude": 41.265138,
            "longitude": -96.039202,
            "__typename": "LocationAddress"
          },
          "__typename": "Location"
        },
        "__typename": "SearchLocationNearResult"
      }
    ]
  },
  "extensions": {
    "tracing": {
      "version": 1,
      "startTime": "2021-04-05T17:55:15.343Z",
      "endTime": "2021-04-05T17:55:16.191Z",
      "duration": 847251634,
      "execution": {
        "resolvers": []
      }
    }
  }
}
{% endhighlight %}

Nice. It looks like a simple array of pharmacies, with each pharmacy object containing location information about the pharmacy, as well as if it has vaccine appointments available. Too easy.

# Scraping the API

Now that we can get pharmacies with available appointments, we need continuously make GET requests to the API to find pharmacies with new openings.

## Choosing a backend

Really any backend would work for something like this. I ended up going with Golang (no pun intended), because it is really a no-frills language and there are plenty of libraries available on GitHub, one of which being a Twitter API library.

## Mocking out the API

The first thing I needed to do before I could start performing Hy-Vee API requests is mock out the objects that are returned from the API. Thankfully, GoLang makes this dead simple with struct annotation.

Here are a few examples of the structs.

{% highlight go %}

type GraphQLRequest struct {
	OperationName string    `json:"operationName"`
	Variables     Variables `json:"variables"`
	Query         string    `json:"query"`
}

type Pharmacy struct {
	Distance float64  `json:"distance"`
	Location Location `json:"location"`
}

type Location struct {
	Nickname                string  `json:"nickname"`
	PhoneNumber             string  `json:"phoneNumber"`
	IsCovidVaccineAvailable bool    `json:"isCovidVaccineAvailable"`
	Address                 Address `json:"address"`
}

type Address struct {
	Line1 string `json:"line1"`
	Line2 string `json:"line2"`
	City  string `json:"city"`
	State string `json:"state"`
	Zip   string `json:"zip"`
}

type Variables struct {
	Radius    int     `json:"radius"`
	Latitude  float64 `json:"latitude"`
	Longitude float64 `json:"longitude"`
}

{% endhighlight %}

## Creating domain structs

At this point, I could just use the above mocked objects for my application domain. In fact, in my original version, I did just this. But, this isn't good programming practice because it ties my application's domain to the API's domain. What if Hy-Vee changes the way a pharmacy is represented? Then my application domain has to change by extension. A better approach is to create a domain just for my application, and use the adapter pattern to change from one domain representation to another.

Here is an example of my domain structs:

{% highlight go %}

type PhoneNumber string
type PharmacyID string

type Pharmacy struct {
	ID                    PharmacyID
	Name                  string
	Address               Address
	PhoneNumber           PhoneNumber
	VaccinationsAvailable bool
}

type Address struct {
	Line1 string
	Line2 string
	City  string
	State string
	Zip   int
}

{% endhighlight %}

## Performing an API request

Now that I had the domain for the API mocked out, I could try to perform a request. For this, I created a simple function

{% highlight go %}
func (h *HyVeeAPI) GetPharmacies(variables Variables) []Pharmacy {
	reqURL := HYVEE_URL + "/my-pharmacy/api/graphql"

	graphReq := &GraphQLRequest{
		OperationName: "SearchPharmaciesNearPointWithCovidVaccineAvailability",
		Variables:     variables,
		Query:         "query SearchPharmaciesNearPointWithCovidVaccineAvailability($latitude: Float!, $longitude: Float!, $radius: Int! = 10) {\n  searchPharmaciesNearPoint(latitude: $latitude, longitude: $longitude, radius: $radius) {\n    distance\n    location {\n      locationId\n      name\n      nickname\n      phoneNumber\n      businessCode\n      isCovidVaccineAvailable\n      covidVaccineEligibilityTerms\n      address {\n        line1\n        line2\n        city\n        state\n        zip\n        latitude\n        longitude\n        __typename\n      }\n      __typename\n    }\n    __typename\n  }\n}\n",
	}
	
	requestBody, err := json.Marshal(graphReq)
	if err != nil {
		fmt.Println(err.Error())
	}

	buffer := bytes.NewBuffer(requestBody)

	req, err := http.NewRequest(http.MethodPost, reqURL, buffer)

	res, err := h.Client.Do(req)
	if err != nil {
		fmt.Println(err.Error())
	}

	defer req.Body.Close()

	type ResponseWrapper struct {
		Data Data `json:"data"`
	}

	var responseList ResponseWrapper
	err = json.NewDecoder(res.Body).Decode(&responseList)
	if err != nil {
		fmt.Println(err.Error())
	}

	return responseList.Data.SearchPharmaciesNearPoint
}
{% endhighlight %}

This functions returns a slice of pharmacies in API representation. I also made a helper function (not shown here), that converts a pharmacy API slice to an pharmacy domain slice.

# Tweeting

In order to start tweeting new vaccine appointments, I needed to setup a new Twitter account. I also needed to request a developer account under the new account so I could get access to Twitter's API. I won't cover this process here, as Twitter provides great documentation and support for this process.

## Not reinventing the wheel

To call Twitter's API, I could have done the same process as I described above, but for Twitter's endpoints. This is a lot of work, and somebody has probably already done this for us. Sure enough, after one search on GitHub, I found [go-twitter](https://github.com/dghubble/go-twitter).

## Sending a Tweet

Next, I made a Twitter struct which has a function called Deliver. Deliver accepts a pharmacy and tweets out information about that pharmacy. I also made a helper function called pharmacyToTweet, which converts the important information contained within a Pharmacy to a formatted string.

Below, I've left out the oauthConfig and token variables since these are obviously secret values. A more robust application might retrieve these values from a config file or even a secrets manager.

{% highlight go %}

type Twitter struct {
	Client *twitter.Client
}

func New() *Twitter {
	oauthConfig := oauth1.NewConfig("", "")
	token := oauth1.NewToken("", "")
	httpClient := oauthConfig.Client(oauth1.NoContext, token)

	returnTwitter := &Twitter{Client: twitter.NewClient(httpClient)}

	return returnTwitter
}

func (t *Twitter) Deliver(pharmacy domain.Pharmacy) error {
	_, res, err := t.Client.Statuses.Update(pharmacyToTweet(pharmacy), nil)
	if err != nil {
		fmt.Println(err.Error())
	} else {
		if res != nil {
			fmt.Printf("Tweet for %s response code %d", pharmacy.PhoneNumber, res.StatusCode)
		}
	}

	return nil
}

func pharmacyToTweet(pharmacy domain.Pharmacy) string {
	addressLineCombination := pharmacy.Address.Line1
	if pharmacy.Address.Line2 != "" {
		addressLineCombination = addressLineCombination + "\n" + pharmacy.Address.Line2
	}

	url := "https://www.hy-vee.com/my-pharmacy/covid-vaccine-consent"

	return fmt.Sprintf("New appointments available at\n%s\n%s, %s %d\n\nPhone: %s\n\n%s",
		addressLineCombination,
		pharmacy.Address.City,
		pharmacy.Address.State,
		pharmacy.Address.Zip,
		pharmacy.PhoneNumber,
		url)
}
{% endhighlight %}

# Putting It All Together

Now that I had the ability to get information about pharmacies from Hy-Vee's API and tweet about them, I could then move on to tying these components together.

## Running periodically

The whole point of this project was to scan the Hy-Vee API at regular intervals. But what is regular? 10 seconds? 5 minutes? 10 minutes? Keep in mind, these vaccine appointments go fast so if I waited too long, I might miss a window of new appointments. I also wanted to be *nice* to Hy-Vee and not make too many requests to their API.

I decided on an interval of 60 seconds. I found this to be a perfect balance between not missing new appointments and not making too many requests. Thats only 60 requests an hour.

Originally, my idea for making API requests periodically looked a little something like this

{% highlight go %}
func main() {
  for true {
    scanAPI()
  }
}

func scanAPI() {
  // Perform API call
  // Detect new appointments
  // Tweet appointments
  time.Sleep(time.Minute)
}
{% endhighlight %}

This definitely worked, and was simple, but I found it to be a little lackluster. I always try to teach myself something new when it comes to making a new projects, so I decided to use Go's `time.Ticker`. 

{% highlight go %}
func main() {
  pharmacyRepo := make(PharmacyMap)
	done := make(chan bool)
	ticker := time.NewTicker(time.Minute)
	// Do initial pharmacy update here
	updatePharmacies(&pharmacyRepo)
	startBot(&pharmacyRepo, done, ticker)
}

func startBot(pharmacyRepo *PharmacyMap, done chan bool, ticker *time.Ticker) {
	for  {
		select {
			case <-ticker.C:
				updatePharmacies(pharmacyRepo)
			case <- done:
				ticker.Stop()
				return
		}
	}
}

{% endhighlight %}

The benefit of using a `Ticker` over just `time.Sleep()` is that the Ticker gets us really close to running exactly on our selected interval.

For example, with the sleep method, let's say our pharmacy update takes 10 seconds. The update would take place, and then the program would sleep for 60 seconds. This means our updating is only being performed every 70 seconds.

The internals of `Ticker` take this problem into account. So, if our update takes 10 seconds, the next scheduled interval will be adjusted to 50 seconds, bringing us to a grand total of a 60s interval.

Keep in mind, with a `Ticker`, our interval does not *tick* right away. So, that is why I need to call `updatePharmacies()` before starting the function that consumes the ticker. That way we don't have to wait 60 seconds for the first update to occur.

## Update logic

Okay, so now I had the ability to run a function periodically, but what does our update function actually look like? Above, I called `updatePharmacies(pharmacyRepo)`. We'll see what that function looks like below.

{% highlight go %}
func updatePharmacies(pharmacyRepo *PharmacyMap) {
	fmt.Printf("Updating pharmacies... at %s\n", time.Now())
	omahaSearchParams := api.Variables{
		Radius:    75,
		Latitude:  redacted,
		Longitude: redacted,
	}

	deliverers := []Deliverer{tweet.New() ,consoleprinter.New()}

	bot := Bot{
		API:       api.HyVeeAPI{Client: http.DefaultClient},
		Deliverers: deliverers,
	}

	newPharmaciesStatuses := getPharmacyMap(bot.API, omahaSearchParams)

	for _, pharmacy := range newPharmaciesStatuses {
		if p, ok := (*pharmacyRepo)[domain.PharmacyID(pharmacy.PhoneNumber)]; ok {
			if p.VaccinationsAvailable == false && pharmacy.VaccinationsAvailable {
				for _, d := range bot.Deliverers {
					_ = d.Deliver(*p)
				}
			}
		}
		(*pharmacyRepo)[pharmacy.ID] = pharmacy
	}
}
{% endhighlight %}

I also have a helper struct, interface, and typee.

{% highlight go %}
type PharmacyMap map[domain.PharmacyID]*domain.Pharmacy

type Deliverer interface {
	Deliver(pharmacy domain.Pharmacy) error
}

type Bot struct {
	API       api.HyVeeAPI
	Deliverers []Deliverer
}
{% endhighlight %}

What's neat is that a `Bot` contains multiple `Deliverers`. Each time new appointments are detected, the `Deliver()` function is called for each Deliverer. I'm not listing it here, but I have a simple `ConsolePrinter` which is a Deliverer which simply prints information about the pharmacy to the console. This helps for debugging.

The logic for detecting new appointments is fairly simple. When the initial update is performed, all found pharmacies are loaded into a map with the key being their ID and the value being the pharmacy object itself. Whenever a new update is performed, a lookup is performed for the existing key for each pharmacy, and if vaccination appointments are now available, and they weren't before, that pharmacy gets delivered using each deliverer.

For simplicities sake, I've used a simple map as my pharmacy *repository*. A more robust application might consider the use of a database to persist pharmacy data.


# Closing Thoughts

I really enjoyed this project. To get an initial, crude version up and running it only took me about an hour. After making some improvements by refactoring and switching to `time.Ticker` I put in another couple of hours. This application is hosted on a tiny AWS EC2 instance, and costs next to nothing to operate.

This Twitter bot has helped a lot of people sign up for their vaccination appointments so far, and that is truly rewarding. To see an application I've developed actually impact peoples' lives for the better is priceless, and has given me motivation to continue in my career field.

<blockquote class="twitter-tweet" align="center"><p lang="en" dir="ltr">Shout out to <a href="https://twitter.com/itsjakehansen?ref_src=twsrc%5Etfw">@itsjakehansen</a> for the amazing vaccine bot that helped me and my bf get our first shots today, and I was able to help a friend with immune issues get an appointment for Friday. He tried for a month and got nowhere until <a href="https://twitter.com/HyveeTracker?ref_src=twsrc%5Etfw">@HyveeTracker</a> thank you so much, Jake</p>&mdash; Jaycie (@jaycre_leigh) <a href="https://twitter.com/jaycre_leigh/status/1379188456480456707?ref_src=twsrc%5Etfw">April 5, 2021</a></blockquote>

<blockquote class="twitter-tweet" align="center"><p lang="en" dir="ltr">Hey Omaha friends, looking to get vaccinated? <br><br>Literally, follow <a href="https://twitter.com/HyveeTracker?ref_src=twsrc%5Etfw">@HyveeTracker</a> and you&#39;ll probably have an appointment by end of today or tomorrow.<br><br>Don&#39;t know who developed this thing, but it&#39;s extremely on point. I&#39;ve helped a couple friends get their shot through this!</p>&mdash; Austin Gaule (@austinomaha) <a href="https://twitter.com/austinomaha/status/1379094387070799873?ref_src=twsrc%5Etfw">April 5, 2021</a></blockquote>

<blockquote class="twitter-tweet" align="center"><p lang="en" dir="ltr">With vaccine eligibility opening more and more, I know people are trying to schedule an appointment.<a href="https://twitter.com/HyveeTracker?ref_src=twsrc%5Etfw">@HyveeTracker</a> is a bot that tracks when Hy-Vees have appointments open on their website.<br><br>Might be a good follow if you&#39;re still trying to get something set up!</p>&mdash; Matt Serwe KETV (@MattSerweKETV) <a href="https://twitter.com/MattSerweKETV/status/1378466416760913925?ref_src=twsrc%5Etfw">April 3, 2021</a></blockquote>

<blockquote class="twitter-tweet" align="center"><p lang="en" dir="ltr">I can’t love this enough!! I set his appointment for Monday. Thank you!!</p>&mdash; Victoria Parks 🇨🇦🇺🇸 (@ParksThriller) <a href="https://twitter.com/ParksThriller/status/1378754722195238918?ref_src=twsrc%5Etfw">April 4, 2021</a></blockquote>

<blockquote class="twitter-tweet" align="center"><p lang="en" dir="ltr">Finally scored a vaccine appointment for my husband, thanks to <a href="https://twitter.com/HyveeTracker?ref_src=twsrc%5Etfw">@HyveeTracker</a> Great work, <a href="https://twitter.com/itsjakehansen?ref_src=twsrc%5Etfw">@itsjakehansen</a> --thank you! <a href="https://t.co/QSXEyEj8TE">https://t.co/QSXEyEj8TE</a></p>&mdash; elliejo (@jomamaliza) <a href="https://twitter.com/jomamaliza/status/1377989321081430021?ref_src=twsrc%5Etfw">April 2, 2021</a></blockquote>
