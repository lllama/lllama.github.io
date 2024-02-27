---
title: "A BubbleTea App to View CloudWatch Logs"
date: 2024-02-27T10:00:33
draft: false
---
My favourite way to view CloudWatch logs is with the [awslogs](https://github.com/jorgebastida/awslogs) tool. Unfortunately, when I last installed it with [`pipx`](https://github.com/pypa/pipx) it gave me errors about missing packages[^1], so I decided to do what any sane person would do and write a new version in a completely different programming language.

I'm a big fan of python's [Textual](https://textual.textualize.io/) library and TUIs in general, and I want 2024 to be my year of learning Go, so I've decided to look at using that along with [BubbleTea](https://github.com/charmbracelet/bubbletea).

So firstly, we'll use the [AWS SDK](https://aws.amazon.com/sdk-for-go/) to get a list of CloudWatch log groups:

```go
// main.go
package main

import (
	"context"
	"fmt"
	"log"

	"github.com/aws/aws-sdk-go-v2/config"
	"github.com/aws/aws-sdk-go-v2/service/cloudwatchlogs"
	"github.com/aws/aws-sdk-go-v2/service/sts"
)

func main() {

    // Hardcode the region for now
	config, err := config.LoadDefaultConfig(context.TODO(), config.WithRegion("eu-west-1"))
	if err != nil {
		log.Fatalf("failed to load configuration, %v", err)
	}

    // We'll use an STS client to check that the credentials are working
	sts_client := sts.NewFromConfig(config)

	_, err = sts_client.GetCallerIdentity(context.Background(), &sts.GetCallerIdentityInput{})
	if err != nil {
		log.Fatalf("Bad AWS Credentials: %v ", err)
	}

    // Now get the CloudWatch Log Groups
	client := cloudwatchlogs.NewFromConfig(config)

    // Use a paginator in case there are more than one
	paginator := cloudwatchlogs.NewDescribeLogGroupsPaginator(client, &cloudwatchlogs.DescribeLogGroupsInput{})

	groups := []string{}

	for paginator.HasMorePages() {
		output, err := paginator.NextPage(context.Background())
		if err != nil {
			log.Fatalf("failed to list log groups, %v", err)
		}
		for _, logGroup := range output.LogGroups {
			groups = append(groups, string(*logGroup.LogGroupName))
		}
	}
    
    for logGroup := range groups {
        fmt.Println(logGroup)
    }
}
```

The above will attempt to log into AWS and get all the log groups from the `eu-west-1` region. The SDK will read the config from the AWS config files, or from the environment variables, so those will need to be set up first.

Next we'll add a sprinkling of BubbleTea to let the user pick which group they want to look at.

BubbleTea uses the [Elm Architecture](https://guide.elm-lang.org/architecture/) (for better or worse[^2]), which means that we need a Model representing our application's state, an `Init` function to perform initialisation, an `Update` function that will mutate the model (if needed), and a `View` function that will draw the screen.

The following is a basic example that displays our list of log groups using BubbleTea.

First we need to define a model:

```go
type model struct {
    logGroups []string{}
}
```

Then the `Init`, `Update` and `View` functions:

```go
func (m model) Init() tea.Cmd {
    return nil
}

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    return m, nil
}

func (m model) View() string {
    return strings.Join(m.logGroups, "\n")
}
```

We then need to update our `main` function to use BubbleTea:

```go
func main() {

    // Create an empty model
	model := model{}

    config, err := config.LoadDefaultConfig(context.TODO(), config.WithRegion("eu-west-1"))
	if err != nil {
		log.Fatalf("failed to load configuration, %v", err)
	}

	sts_client := sts.NewFromConfig(config)

	_, err = sts_client.GetCallerIdentity(context.Background(), &sts.GetCallerIdentityInput{})
	if err != nil {
		fmt.Println(errorStyle.Render(fmt.Sprintf("Bad AWS Credentials: %v ", err)))
		return
	}

	client := cloudwatchlogs.NewFromConfig(config)

	paginator := cloudwatchlogs.NewDescribeLogGroupsPaginator(client, &cloudwatchlogs.DescribeLogGroupsInput{})

	for paginator.HasMorePages() {
		output, err := paginator.NextPage(context.Background())
		if err != nil {
			log.Fatalf("failed to list log groups, %v", err)
		}
		for _, logGroup := range output.LogGroups {
            model.logGroups = append(model.logGroups, string(*logGroup.LogGroupName))
		}
	}

    // Now create our BubbleTea program
	btProg := tea.NewProgram(model)

	if _, err := btProg.Run(); err != nil {
		log.Fatalf("failed to start program, %v", err)
	}
}
```

This is a very simple example of a BubbleTea app ***BUT DON'T RUN IT!***

BubbleTea requires you to do most/every thing yourself and in this case, it requires you to provide a way to quit the application. If you run the above code, then the app will display the list of log groups but then sit there waiting for input. You'll have to kill it from a separate terminal window to make it quit. To provide a way to handle quitting, we need to change our `Update` function:

```go

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    switch msg := msg.(type) {

        case tea.KeyMsg:
        switch msg.String() {

        case "ctrl+c", "q":
            return m, tea.Quit
        }
    }
    return m, nil
}
```
The `Update` function now checks the type of the message passed to it and, if it's a `KeyMsg`, it quits the app if the user pressed `q` or `ctrl-c`. Running this app will now print the list of log groups and wait for the user to quit the app.

I'm a big fan of Textual and it creates full screen apps, so let's do a little tweak to run this app full screen. Change the call to `NewProgram` to have the `tea.WithAltScreen()` option:

```go
	btProg := tea.NewProgram(model, tea.WithAltScreen())
	if _, err := btProg.Run(); err != nil {
		log.Fatalf("failed to start program, %v", err)
	}
```



[^1]: Turns out this is easily fixed with `pipx inject awslogs setuptools` but let's not let that spoil things.

[^2]: The Elm architecture makes sense - it's a nice way of adding some rigour to UIs. One _very key_ difference between Elm and BubbleTea, however, is that Elm has a whole browser sitting underneath it. That means it's very easy to attach handlers etc to DOM elements and have them pass messages to Elm. BubbleTea does not have this and the user is responsible for everything that the browser would do. You'll need to handle tabbing between elements, "focus" of controls, and even things like hit detection for mouse events. That's a lot of work.