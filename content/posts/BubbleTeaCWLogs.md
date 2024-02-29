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

Now we want the ability to have a user select a log group to view. The bare-bones approach to this with BubbleTea is to alter our model to track the user's selection. We'll need to update the model to something like:

```go
type model struct {
    logGroups []string
    cursor    int
}
```

And then we change the `Update` function to handle key presses for navigation (e.g. `up`, `down`, `j`, `k`, etc) and use the cursor field in the model to display an indicator in our `View` function. But that's a lot of work, and we'll need to handle things like reaching the top and bottom of the list, pagination if we have a lot of log groups, and various other display issues.

Thankfully, BubbleTea comes with [Bubbles](https://github.com/charmbracelet/bubbles/). These are ready-made components that you can add to your app. For our use case, we'll want to use the [`List`](https://github.com/charmbracelet/bubbles/?tab=readme-ov-file#list) component.

First we'll need to update the model to use the `list` bubble's internal model:

```go
type model struct {
	logGroups list.Model
}
```

The list's model expects item's to be added to it that match the `list.Item` interface, so let's define a type that does just that:

```go
type item string
func (i item) FilterValue() string { return "" }
```

(The `list` bubble let's you filter the list, so the component needs to know what value to apply the filter to. At the moment, we're not supporting filtering, so we just return an empty string.)

The rendering of each list item is handled by an `ItemDelegate`. We can use the `DefaultItemDelegate` that is provided by the `list` package but this requires that our list `item`'s have a title and a description, which is more than what we need for our simple display, so we'll create our own delegate instead.

```go
type itemDelegate struct{}
func (d itemDelegate) Height() int                             { return 1 }
func (d itemDelegate) Spacing() int                            { return 0 }
func (d itemDelegate) Update(_ tea.Msg, _ *list.Model) tea.Cmd { return nil }
func (d itemDelegate) Render(w io.Writer, m list.Model, index int, listItem list.Item) {
	i, ok := listItem.(item)
	if !ok {
		return
	}

	str := fmt.Sprintf("%d. %s", index+1, i)

	if index == m.Index() {
		str = "> " + str
	} else {
		str = "  " + str
	}

	fmt.Fprint(w, fn(str))
}
```
The `ItemDelegate` interface requires us to have `Height`, `Spacing`, `Render`, and `Update` methods on our struct. Here you can see that the `Render` method is doing most of the work. We check if the index of the selection is the same as the index of the item being rendered, and put a `>` indicator if so.

We then need to change our `main` function to construct the list of `list.Item`s and to set up the model to use our delegate:

```go
func main() {

	// Create the model with an empty list of items that uses our delegate.
	// The `20` is the list width and the `10` is the list height. Note that this
	// height is the total height of the list bubble, which includes help text,
	// title, filter text, and the items.
	model := model{logGroups: list.New([]list.Item{}, itemDelegate{}, 20, 10)}


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


	// Create a slice of items
	groups := []list.Item{}

	for paginator.HasMorePages() {
		output, err := paginator.NextPage(context.Background())
		if err != nil {
			log.Fatalf("failed to list log groups, %v", err)
		}
		for _, logGroup := range output.LogGroups {
			// Add the items to the list
			groups = append(groups, item(string(*logGroup.LogGroupName)))
		}
	}

	// Update the model
	model.logGroups.SetItems(groups)

	btProg := tea.NewProgram(model, tea.WithAltScreen())

	if _, err := btProg.Run(); err != nil {
		log.Fatalf("failed to start program, %v", err)
	}
}
```

We then need to change our `Update` method. We need to handle what we need to handle (quitting etc) and then pass the message to the `list`'s model to do its updates:

```go
func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
	var (
		cmd tea.Cmd
	)

	switch msg := msg.(type) {
	case tea.KeyMsg:
		switch msg.String() {

		case "q", "ctrl+c":
			return m, tea.Quit
		}
	}
	m.logGroups, cmd = m.logGroups.Update(msg)

	return m, cmd
}
```

If we run the app, we'll get a full screen with a tiny list. You'll also see the help text, and you can use `/` to turn on the filter function and filter the items in the list. One bug is that if we try and filter the list to any items that contain the letter `q`... the app will quit. This is because our `Update` function checks for the key first, and quits the app before we call `Update` on the list.

To fix this, we need to check the state of the `list`'s filtering feature:

```go
func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
	var (
		cmd tea.Cmd
	)

	switch msg := msg.(type) {
	case tea.KeyMsg:
		// If we're not filtering then we can add our own logic
		if !(m.logGroups.FilterState() == list.Filtering) {
			switch msg.String() {

			case "q", "ctrl+c":
				return m, tea.Quit
			}
		}
	}
	m.logGroups, cmd = m.logGroups.Update(msg)

	return m, cmd
}
```

[^1]: Turns out this is easily fixed with `pipx inject awslogs setuptools` but let's not let that spoil things.

[^2]: The Elm architecture makes sense - it's a nice way of adding some rigour to UIs. One _very key_ difference between Elm and BubbleTea, however, is that Elm has a whole browser sitting underneath it. That means it's very easy to attach handlers etc to DOM elements and have them pass messages to Elm. BubbleTea does not have this and the user is responsible for everything that the browser would do. You'll need to handle tabbing between elements, "focus" of controls, and even things like hit detection for mouse events. That's a lot of work.