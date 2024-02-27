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

BubbleTea uses the [Elm Architecture](https://guide.elm-lang.org/architecture/) (for better or worse[^2]), which means that we need a Model representing our application's state, an Update function that will mutate the model (if needed), and a View function that will draw the screen.

The following is a basic example 

[^1]: Turns out this is easily fixed with `pipx inject awslogs setuptools` but let's not let that spoil things.

[^2]: The Elm architecture makes sense - it's a nice way of adding some rigour to UIs. One _very key_ difference between Elm and BubbleTea, however, is that Elm has a whole browser sitting underneath it. That means it's very easy to attach handlers etc to DOM elements and have them pass messages to Elm. BubbleTea does not have this and the user is responsible for everything that the browser would do. You'll need to handle tabbing between elements, "focus" of controls, and even things like hit detection for mouse events. That's a lot of work.